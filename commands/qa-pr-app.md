---
description: Verify app-change PR items via dev server and browser automation
argument-hint: <pr-number>
---

Start the dev server for the PR's worktree, exercise each test-plan item with browser automation and the verify skill, elicit operator confirmation, and mark passed items in the PR body.

## Steps

**1. Fetch PR and extract test-plan items**

```bash
gh pr view $ARGUMENTS --json number,title,body,headRefName,url
```

Parse lines matching `^- \[ \]` in the section whose heading contains "test" (case-insensitive). Store as an ordered list. If no items are found, print a notice and return to the caller.

**2. Read the port and start the dev server if needed**

```bash
grep "^PORT=" .env.local | cut -d= -f2 > /tmp/qa-pr-port.txt
curl -s -o /dev/null -w "%{http_code}" http://localhost:$(cat /tmp/qa-pr-port.txt)/up > /tmp/qa-pr-status.txt 2>/dev/null
```

If the content of `/tmp/qa-pr-status.txt` is not `200`:
```bash
bin/worktree server <headRefName>
```

Poll `/up` every 2 seconds, up to 10 attempts. If still not up after 10 s, abort:
```
ERROR: dev server did not start within 10 s. Check log/dev_server.log for details.
```

**3. Open the app and take a baseline snapshot**

Load the Playwright MCP tools, navigate to the app, and snapshot the accessibility tree to confirm it loaded and identify available affordances:

```
ToolSearch: select:mcp__playwright__browser_navigate,mcp__playwright__browser_snapshot,mcp__playwright__browser_take_screenshot

mcp__playwright__browser_navigate { url: "http://localhost:<port>/" }  # <port> = content of /tmp/qa-pr-port.txt
mcp__playwright__browser_snapshot {}
```

Confirm the expected affordances are present (e.g. `starting_integer` input, Submit button, `#results` region). Then take one screenshot to show the operator the live state:

```
mcp__playwright__browser_take_screenshot {}
```

Present the screenshot and any accessibility gaps to the operator. On cloud environments there is no live browser window; the screenshot confirms the page is up.

**4. Walk through each test-plan item**

Process items sequentially. For each item N of M:

**4a. Announce the item**

```
── Step N/M ─────────────────────────────────────────
<item text>
```

**4b. Run automated pre-checks**

Based on the item text, run relevant checks and present findings:
- If the item mentions a specific route or URL: curl it and show the status
- If the item mentions a file: check it exists and show relevant content
- If the item mentions a git check: run `git status` or `git log --oneline -3`
- If the item mentions a form or UI interaction: use `mcp__playwright__browser_navigate` to go to the relevant page and `mcp__playwright__browser_snapshot` to read the accessibility tree and identify available affordances

**4c. Use the verify skill**

Invoke the `cpb-dev-workflow:verify` skill, passing item text and server context as a multi-line args string:

```
<item text>
--port <port>
--server-up
```

`<item text>` is the verbatim item text (no `- [ ] ` prefix); `<port>` is `$(cat /tmp/qa-pr-port.txt)`; `--server-up` signals the server is already confirmed running. Present the findings to the operator.

**4d. Ask for operator confirmation**

Use `AskUserQuestion`:

- Question: `"Step N/M: <item text> — pass or fail?"`
- Options: `"Pass"` / `"Fail"`

**4e. On Pass**

Fetch the current PR body and replace `- [ ] <item text>` with `- [x] <item text>`:
```bash
gh pr view <number> --json body --jq '.body' > /tmp/qa-pr-<number>-body.txt
# apply replacement for this item on the content of /tmp/qa-pr-<number>-body.txt
gh pr edit <number> --body "<updated_body>"
```

Print: `✓ Step N marked as passed in PR #<number>.`

**4f. On Fail**

Print:
```
✗ Step N failed: <item text>
```

Signal failure back to the router. Do not continue to the next item.

**5. All items passed**

Print:
```
✓ All M app test-plan items passed for PR #<number>.
```

Return to the caller (router).
