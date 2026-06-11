---
description: Poll CI for a PR, reproduce failing system tests locally, classify and fix root causes, and push fixes until green — or escalate after 5 attempts.
argument-hint: <pr-number>
---

Given PR `$ARGUMENTS`, drive CI to green by diagnosing and fixing failing system tests in an isolated worktree, looping up to 5 attempts total before asking for help.

---

## Known failure playbook

Before touching any code, consult this classification table. Match the test output against the patterns and jump straight to the fix.

| ID | Symptom | Root cause | Fix |
|----|---------|-----------|-----|
| **VCR** | `WebMock::NetConnectNotAllowedError` or `VCR::Errors::UnhandledHTTPRequestError` referencing `localhost` or `127.0.0.1` | VCR/WebMock blocks real HTTP; test hits a localhost service that wasn't recorded | Add `allow_localhost: true` in the VCR configure block or `WebMock.disable_net_connect!(allow_localhost: true)` in the relevant test helper; or delete the cassette file under `test/cassettes/` so it re-records |
| **HEIGHT** | `Capybara::ElementNotFound` or assertion failure mentioning `height`, `min-height`, or `overflow` | CSS layout height constraint not accounted for in test viewport | Before the failing step insert `page.driver.browser.manage.window.resize_to(1280, 900)`; or fix the height constraint in the stylesheet |
| **BRAKEMAN** | Brakeman exits non-zero; message references a hook property like `_before_action`, `skip_before_action`, or a custom filter | Brakeman inspects hook metadata that changed; false positive or real finding | If false positive: add `# brakeman:ignore:WarningType -- reason` on the offending line; if real: fix the underlying security issue |
| **CREDS** | `ActiveSupport::MessageEncryptor::InvalidMessage`, `ArgumentError: key must be 32 bytes`, or `Rails.application.credentials` returning `nil` | `credentials.yml.enc` or `master.key` absent/mismatched in the worktree | Copy the key from the primary worktree (see CREDS fix in §8a); never commit the key |
| **FLAKY** | Test failure that disappears on re-run with no code change | Race condition, time-zone sensitivity, or DB ordering assumption | Re-run once; if it passes, wrap the test body with `travel_to(Time.zone.parse("2024-01-01")) { ... }` and commit |

---

## Steps

### 1. Validate arguments

```bash
if [ -z "$ARGUMENTS" ]; then echo "Usage: /heal-ci <pr-number>"; exit 1; fi
PR="$ARGUMENTS"
```

### 2. Initialize the attempt counter

**Set `ATTEMPT=0` once here, before any loop.** This counter is shared across every inner fix iteration and every outer poll→fix→push cycle. It is never reset. Escalate the moment `ATTEMPT` reaches 5.

### 3. Fetch PR metadata

```bash
gh pr view "$PR" --json headRefName,title,url | tee /tmp/heal-ci-pr.json
```

Extract:
- `HEAD_BRANCH` = `.headRefName`
- `PR_TITLE` = `.title`
- `PR_URL` = `.url`

### 4. Open an isolated worktree (once)

Use the plugin's worktree tooling so that `bin/setup` runs (installs dependencies, prepares the DB) and a port is assigned:

```bash
bin/worktree prepare "$PR" --pr > /tmp/heal-ci-wt.json
WORKTREE_PATH=$(jq -r '.path' /tmp/heal-ci-wt.json)
```

If `bin/worktree prepare` is unavailable, fall back to:

```bash
WORKTREE_PATH=$(git worktree list --porcelain \
  | grep -B2 "branch refs/heads/$HEAD_BRANCH" \
  | grep "^worktree" | awk '{print $2}')
if [ -z "$WORKTREE_PATH" ]; then
  git fetch origin "$HEAD_BRANCH"
  git worktree add "/tmp/heal-ci-pr-${PR}" "origin/$HEAD_BRANCH"
  WORKTREE_PATH="/tmp/heal-ci-pr-${PR}"
  cd "$WORKTREE_PATH" && bin/setup --skip-server
fi
```

All subsequent shell commands run inside `$WORKTREE_PATH`.

### 5. Detect test framework

```bash
if [ -f "$WORKTREE_PATH/.rspec" ] || [ -d "$WORKTREE_PATH/spec" ]; then
  TEST_FRAMEWORK="rspec"
else
  TEST_FRAMEWORK="minitest"
fi
```

Use this variable throughout steps 7–8 to select the correct run command:

- **minitest**: `cd "$WORKTREE_PATH" && bin/rails test <test-file>:<line>`
- **rspec**: `cd "$WORKTREE_PATH" && bundle exec rspec <test-file>:<line> --format documentation`

Test file paths differ by framework:
- minitest: `test/system/*_test.rb`, `test/cassettes/`
- rspec: `spec/system/*_spec.rb`, `spec/cassettes/`

### 6. Poll CI until complete

```bash
gh pr checks "$PR" --watch --interval 10
```

If all checks pass → print "CI green for PR #$PR — nothing to fix." and stop (clean up the worktree per §10 first).

If any checks fail, collect failing jobs. Note: `gh pr checks --json` emits a `state` field with values `SUCCESS`, `FAILURE`, `PENDING`, `SKIPPED`, or `CANCELLED`. Filter on `FAILURE`:

```bash
gh pr checks "$PR" --json name,state,detailsUrl \
  | jq '[.[] | select(.state == "FAILURE")]' \
  > /tmp/heal-ci-failures.json
```

If the `--json` flag is unsupported by the installed `gh` version, parse the tabular output instead:

```bash
gh pr checks "$PR" 2>&1 | grep -E '\bfail\b|\bFAIL\b|\bfailure\b' \
  > /tmp/heal-ci-failures.txt
```

### 7. Fetch CI logs for failing jobs

For each failing job, fetch the raw log using the GitHub MCP tool:

```
ToolSearch: select:mcp__github__get_job_logs
```

Call `mcp__github__get_job_logs` for each failing job ID (extract from `detailsUrl`).

Parse the log to extract:
- The **test file path and line number** (e.g. `test/system/foo_test.rb:42` or `spec/system/foo_spec.rb:42`)
- The **error message** and **backtrace top frame**

### 8. Fix loop

**Check `ATTEMPT` before each iteration. If `ATTEMPT >= 5`, jump immediately to §Escalate.**

Increment `ATTEMPT` at the start of each iteration (before applying the fix).

**8a. Classify root cause**

Match the extracted error against the **Known failure playbook** table above. Record the ID (`VCR`, `HEIGHT`, `BRAKEMAN`, `CREDS`, or `FLAKY`). If no pattern matches, label `UNKNOWN`.

**8b. Apply targeted fix** based on classification:

- **VCR**: In the test file or its `test/test_helper.rb` / `spec/support` helper, locate the VCR configure block and add `c.allow_localhost = true`, or add `WebMock.disable_net_connect!(allow_localhost: true)`. Delete any stale cassette file under `test/cassettes/` (minitest) or `spec/cassettes/` (rspec) so it re-records.
- **HEIGHT**: Insert `page.driver.browser.manage.window.resize_to(1280, 900)` immediately before the failing step in the test file. If a CSS height constraint is the root cause, fix the value in the stylesheet.
- **BRAKEMAN**: Run `bundle exec brakeman -f json 2>/dev/null | jq '.warnings[]'` to locate the exact warning. If false positive, add `# brakeman:ignore:WarningType -- reason` on the offending line. If real, apply the minimum fix.
- **CREDS**: Locate the primary worktree path via `git worktree list --porcelain | head -2 | grep "^worktree" | awk '{print $2}'`, then copy the key:
  ```bash
  PRIMARY=$(git worktree list --porcelain | head -2 | grep "^worktree" | awk '{print $2}')
  cp "$PRIMARY/config/master.key" "$WORKTREE_PATH/config/master.key"
  ```
  If the key is absent from the primary worktree too, note this in the escalation message.
- **FLAKY**: Wrap the failing test body with `travel_to(Time.zone.parse("2024-01-01")) { ... }`. For ordering assumptions, add `.order(:id)` to the relevant query.
- **UNKNOWN**: Read the full backtrace, identify the first application frame, and apply a minimal targeted fix.

**8c. Verify fix locally**:

```bash
# minitest
cd "$WORKTREE_PATH" && bin/rails test <test-file>:<line> 2>&1 \
  | tee /tmp/heal-ci-attempt-${ATTEMPT}.log

# rspec
cd "$WORKTREE_PATH" && bundle exec rspec <test-file>:<line> --format documentation 2>&1 \
  | tee /tmp/heal-ci-attempt-${ATTEMPT}.log
```

If the test passes, proceed to **8d**. If it fails and `ATTEMPT < 5`: re-read the new failure output, update classification if needed, increment `ATTEMPT`, and return to **8b**. If `ATTEMPT >= 5`, jump to **§Escalate**.

**8d. Commit atomically**:

```bash
cd "$WORKTREE_PATH"
git add -p   # stage only the targeted fix
git commit -m "fix(ci): <one-line description of root cause and fix> [attempt $ATTEMPT]"
```

### 9. Push and re-poll

```bash
cd "$WORKTREE_PATH"
git push origin HEAD:"$HEAD_BRANCH"
```

Then return to **§6. Poll CI**. Do not reset `ATTEMPT` — the same counter continues across all outer cycles.

### 10. Report green

When `gh pr checks "$PR"` shows all checks passing:

```
✓ CI green for PR #$PR after $ATTEMPT attempt(s).
  Branch: $HEAD_BRANCH
  PR: $PR_URL
  Fixes applied: <bullet list of each classification and the file changed>
```

Tear down the worktree using the plugin tooling:

```bash
bin/worktree cleanup "$PR"
```

If `bin/worktree cleanup` is unavailable: `git worktree remove --force "$WORKTREE_PATH"`.

---

## Escalate

If `ATTEMPT >= 5` without green CI, use `AskUserQuestion` with:

- The PR number and URL
- The persistent failing test(s) with file:line
- The classification reached on the last attempt
- The last 40 lines of `/tmp/heal-ci-attempt-${ATTEMPT}.log`
- A clear question: "I could not reach green after 5 attempts. The remaining failure is `<classification>` in `<file:line>`. Here is the last output — what should I try next?"

Do **not** push uncommitted changes before escalating. Leave the worktree in place so the state is inspectable.
