---
description: Poll CI for a PR, reproduce failing system tests locally, classify and fix root causes, and push fixes until green — or escalate after 5 attempts.
argument-hint: <pr-number>
---

Given PR `$ARGUMENTS`, drive CI to green. Three `bin/worktree` subcommands handle all the shell work; Claude classifies the failure and edits the one file that needs fixing.

---

## Known failure playbook

Match the test output from `/tmp/heal-ci-state.json` and the last log file against these patterns:

| ID | Symptom | Root cause | Fix |
|----|---------|-----------|-----|
| **VCR** | `WebMock::NetConnectNotAllowedError` or `VCR::Errors::UnhandledHTTPRequestError` referencing `localhost` or `127.0.0.1` | VCR/WebMock blocks real HTTP to localhost | Add `c.allow_localhost = true` in the VCR configure block, or `WebMock.disable_net_connect!(allow_localhost: true)` in `test/test_helper.rb`; delete stale cassette under `test/cassettes/` |
| **HEIGHT** | `Capybara::ElementNotFound` or assertion failure mentioning `height`, `min-height`, or `overflow` | CSS height constraint exceeds test viewport | Insert `page.driver.browser.manage.window.resize_to(1280, 900)` before the failing step; or fix the stylesheet value |
| **BRAKEMAN** | Brakeman exits non-zero; message references `_before_action`, `skip_before_action`, or a custom filter | Hook metadata changed; false positive or real finding | False positive: add `# brakeman:ignore:WarningType -- reason` on the offending line; real: apply minimum security fix |
| **CREDS** | `ActiveSupport::MessageEncryptor::InvalidMessage`, `key must be 32 bytes`, or `credentials` returning `nil` | `master.key` missing from the worktree | Copy from the primary worktree (see §Fix loop, CREDS) |
| **FLAKY** | Failure disappears on re-run with no code change | Race condition, time-zone sensitivity, or ordering assumption | Wrap test body with `travel_to(Time.zone.parse("2024-01-01")) { ... }`; or add `.order(:id)` to the relevant query |

---

## Steps

### 1. Validate arguments

```bash
if [ -z "$ARGUMENTS" ]; then echo "Usage: /heal-ci <pr-number>"; exit 1; fi
PR="$ARGUMENTS"
```

### 2. Open an isolated worktree (once)

```bash
bin/worktree prepare "$PR" --pr > /tmp/heal-ci-wt.json
```

This runs `bin/setup --skip-server`, assigns a port, and writes `.worktree-session.json`. The worktree path is available in the JSON if needed.

### 3. Poll → classify → fix loop

**This loop runs until `bin/worktree heal-poll` exits 0 (green) or 2 (limit reached).**

---

**3a. Poll CI**

```bash
bin/worktree heal-poll "$PR"
POLL_EXIT=$?
```

- Exit 0 → CI is green. Jump to **§Done**.
- Exit 2 → Limit reached. Jump to **§Escalate**.
- Exit 1 → Failures found. Continue.

---

**3b. Fetch logs for failing jobs**

Read `/tmp/heal-ci-state.json` to get `failures[].detailsUrl`. For each failing job, fetch the full log:

```
ToolSearch: select:mcp__github__get_job_logs
```

Call `mcp__github__get_job_logs` with the job ID extracted from `detailsUrl`. Parse the log to extract:
- Test file path and line number (e.g. `test/system/foo_test.rb:42`)
- Error message and backtrace top frame

---

**3c. Reproduce locally**

```bash
bin/worktree heal-reproduce "$PR" "<file>:<line>"
REPRO_EXIT=$?
```

- Runs `bin/rails test <file>:<line>` in the worktree.
- If `REPRO_EXIT=0`: the test passes locally (flaky). Skip to **3e** with classification `FLAKY` and no file edit.

---

**3d. Classify and fix**

Read the log at the path in `state.json` as `last_log`. Match against the **Known failure playbook** above. Then edit exactly one file:

- **VCR**: Add `allow_localhost: true` / `disable_net_connect!(allow_localhost: true)` in `test/test_helper.rb`, or delete the stale cassette under `test/cassettes/`.
- **HEIGHT**: Insert `page.driver.browser.manage.window.resize_to(1280, 900)` before the failing step in the test file.
- **BRAKEMAN**: Add `# brakeman:ignore:WarningType -- reason` on the offending line, or apply the minimum security fix.
- **CREDS**: Copy the key from the primary worktree:
  ```bash
  PRIMARY=$(git worktree list --porcelain | head -2 | grep "^worktree" | awk '{print $2}')
  cp "$PRIMARY/config/master.key" "$(jq -r .wt_path /tmp/heal-ci-state.json)/config/master.key"
  ```
  If the key is absent from the primary worktree too, jump to **§Escalate**.
- **FLAKY**: Wrap the test body with `travel_to(...)` or add `.order(:id)`.
- **UNKNOWN**: Read the full backtrace, find the first application frame, apply a minimal fix.

---

**3e. Verify locally**

```bash
bin/worktree heal-reproduce "$PR" "<file>:<line>"
REPRO_EXIT=$?
```

If `REPRO_EXIT=1`: the fix didn't work. Go back to **3d** with the new log. `heal-reproduce` does not increment the counter — only `heal-commit` does, so re-classifying doesn't burn an attempt.

---

**3f. Commit, push, re-poll**

```bash
bin/worktree heal-commit "$PR" "<edited-file>" "fix(ci): <classification> — <one-line description>"
COMMIT_EXIT=$?
```

- Exit 0 → committed and pushed. Return to **3a** to re-poll.
- Exit 5 → attempt limit reached. Jump to **§Escalate**.

---

### Done

```
✓ CI green for PR #$PR
```

Tear down the worktree:

```bash
bin/worktree cleanup "$PR"
```

---

## Escalate

Use `AskUserQuestion` with:

- PR number and URL (from `/tmp/heal-ci-state.json`)
- Persistent failing test(s) with file:line
- Classification reached on the last attempt
- Last 40 lines of the log file named in `state.json` as `last_log`
- Question: "I could not reach green after 5 attempts. The remaining failure is `<classification>` in `<file:line>`. Here is the last output — what should I try next?"

Leave the worktree in place so the state is inspectable.
