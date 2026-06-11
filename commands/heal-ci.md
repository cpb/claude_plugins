---
description: Poll CI for a PR, reproduce failing system tests locally, classify and fix root causes, and push fixes until green — or escalate after 5 attempts.
argument-hint: <pr-number>
---

Given PR `$ARGUMENTS`, drive CI to green by diagnosing and fixing failing system tests in an isolated worktree, looping up to 5 times before asking for help.

---

## Known failure playbook

Before touching any code, consult this classification table. Match the test output against the patterns and jump straight to the fix.

| ID | Symptom | Root cause | Fix |
|----|---------|-----------|-----|
| **VCR** | `WebMock::NetConnectNotAllowedError` or `VCR::Errors::UnhandledHTTPRequestError` referencing `localhost` or `127.0.0.1` | VCR/WebMock blocks real HTTP; test hits a localhost service that wasn't recorded | Add `:allow_localhost` option in the cassette or `WebMock.disable_net_connect!(allow_localhost: true)` in the relevant spec helper; or record a new cassette by setting `VCR_RECORD=all` |
| **HEIGHT** | `Capybara::ElementNotFound` or layout assertion failure containing words like `height`, `min-height`, or `overflow` | CSS layout height constraint not accounted for in test viewport | Increase `Capybara.default_max_wait_time` **or** set an explicit `page.driver.browser.manage.window.resize_to` before the step, or fix the constraint in the view |
| **BRAKEMAN** | Brakeman exits non-zero; message references a hook property like `_before_action`, `skip_before_action`, or a custom filter | Brakeman inspects hook metadata that changed; false positive or real finding | If false positive: add a `# brakeman:ignore` annotation with a reason; if real: fix the underlying security issue (mass assignment, redirect, SQL injection, etc.) |
| **CREDS** | `ActiveSupport::MessageEncryptor::InvalidMessage` or `ArgumentError: key must be 32 bytes` or `Rails.application.credentials` returning `nil` | `credentials.yml.enc` or `master.key` absent/mismatched in the test environment | Ensure `config/master.key` (or `config/credentials/<env>.key`) is present in the worktree; re-run `rails credentials:edit` if corrupted; never commit the key |
| **FLAKY** | Test failure that disappears on re-run with no code change | Race condition, time-zone sensitivity, or DB ordering assumption | Re-run once; if it passes, commit a `--order random --seed` annotation or add `travel_to` / `Timecop.freeze` wrapper |

---

## Steps

### 1. Validate arguments

```bash
if [ -z "$ARGUMENTS" ]; then echo "Usage: /heal-ci <pr-number>"; exit 1; fi
PR="$ARGUMENTS"
```

### 2. Fetch PR metadata

```bash
gh pr view "$PR" --json headRefName,title,url,statusCheckRollup \
  | tee /tmp/heal-ci-pr.json
```

Extract:
- `HEAD_BRANCH` = `.headRefName`
- `PR_TITLE` = `.title`
- `PR_URL` = `.url`

### 3. Poll CI until complete

```bash
gh pr checks "$PR" --watch --interval 10
```

If all checks pass → print "CI green for PR #$PR — nothing to fix." and stop.

If any checks fail, collect the list of failing check names and job URLs:

```bash
gh pr checks "$PR" --json name,state,detailsUrl \
  | jq '[.[] | select(.state == "FAILURE" or .state == "ERROR")]' \
  > /tmp/heal-ci-failures.json
```

### 4. Open an isolated worktree

```bash
WORKTREE_PATH="/tmp/heal-ci-pr-${PR}"
if [ ! -d "$WORKTREE_PATH" ]; then
  git fetch origin "$HEAD_BRANCH"
  git worktree add "$WORKTREE_PATH" "origin/$HEAD_BRANCH"
fi
```

All subsequent shell commands in steps 5–8 run inside `$WORKTREE_PATH` (prefix with `cd "$WORKTREE_PATH" &&` or use subshells).

### 5. Fetch CI logs for failing jobs

For each failing job URL, fetch the raw log:

```bash
gh run view --log <run-id>   # extract run-id from detailsUrl
```

Or use the GitHub MCP tool:
```
ToolSearch: select:mcp__github__get_job_logs
```
then call `mcp__github__get_job_logs` for each failing job.

Parse the log to extract:
- The **test file path** and line number (e.g. `spec/system/foo_spec.rb:42`)
- The **error message** and **backtrace top frame**

### 6. Classify root cause

Match the extracted error against the **Known failure playbook** table above (§ Known failure playbook). Record the matched ID (`VCR`, `HEIGHT`, `BRAKEMAN`, `CREDS`, or `FLAKY`). If no pattern matches, label it `UNKNOWN`.

### 7. Reproduce locally

```bash
cd "$WORKTREE_PATH"
bundle exec rspec <test-file>:<line> --format documentation 2>&1 | tee /tmp/heal-ci-local.log
```

Confirm the failure reproduces. If it does not reproduce locally, label `FLAKY` and proceed to § Commit & push with a re-run annotation only.

### 8. Fix loop (up to 5 attempts)

Initialise `ATTEMPT=1`.

**8a. Apply targeted fix** based on classification:

- **VCR**: In the spec file or its `spec/support` helper, locate the cassette block or `WebMock.disable_net_connect!` call and add `allow_localhost: true`. If a cassette file exists under `spec/cassettes/`, delete it so it re-records on next run; or set `record: :new_episodes`.
- **HEIGHT**: Find the `resize_to` or `Capybara.current_session.current_window.resize_to` call nearest the failing step. If absent, insert `page.driver.browser.manage.window.resize_to(1280, 900)` before the step. If a height constraint in CSS is the root cause, adjust the value in the stylesheet.
- **BRAKEMAN**: Run `bundle exec brakeman -f json 2>/dev/null | jq '.warnings[]'` to locate the exact warning. If false positive, add `# brakeman:ignore:WarningType -- reason` on the offending line. If real, apply the minimum fix (sanitize input, restrict attribute, etc.).
- **CREDS**: Check for `config/master.key`. If missing: copy from the main worktree if available (`cp "$(git rev-parse --show-toplevel)/../config/master.key" config/master.key`). If unavailable, note this in the escalation message.
- **FLAKY**: Add `aggregate_failures` or wrap the example with `travel_to(Time.zone.parse("2024-01-01"))` and re-run.
- **UNKNOWN**: Read the full backtrace, identify the first application frame, and apply a minimal targeted fix.

**8b. Verify fix locally**:

```bash
cd "$WORKTREE_PATH"
bundle exec rspec <test-file>:<line> --format documentation 2>&1 | tee /tmp/heal-ci-attempt-${ATTEMPT}.log
```

If the test passes, proceed to **8c**. If it fails:
- Increment `ATTEMPT`.
- If `ATTEMPT > 5`, jump to **§ Escalate**.
- Re-read the new failure output, update the classification if needed, and return to **8a**.

**8c. Commit atomically**:

```bash
cd "$WORKTREE_PATH"
git add -p   # stage only the targeted fix; do not stage unrelated changes
git commit -m "fix(ci): <one-line description of root cause and fix> [attempt $ATTEMPT]"
```

### 9. Push and re-poll

```bash
cd "$WORKTREE_PATH"
git push origin HEAD:"$HEAD_BRANCH"
```

Then return to **§ 3. Poll CI** and repeat the full cycle (polling → classify → fix) counting each outer loop pass as one of the 5 attempts.

### 10. Report green

When `gh pr checks "$PR"` shows all checks passing:

```
✓ CI green for PR #$PR after $ATTEMPT attempt(s).
  Branch: $HEAD_BRANCH
  PR: $PR_URL
  Fixes applied: <bullet list of each classification and the file changed>
```

Remove the worktree:

```bash
git worktree remove --force "$WORKTREE_PATH"
```

---

## Escalate

If all 5 attempts are exhausted without green CI, use `AskUserQuestion` with:

- The PR number and URL
- The persistent failing test(s) with file:line
- The classification reached on attempt 5
- The last 40 lines of `/tmp/heal-ci-attempt-5.log`
- A clear question: "I could not reach green after 5 attempts. The remaining failure is `<classification>` in `<file:line>`. Here is the last output — what should I try next?"

Do **not** push any uncommitted changes before escalating.
