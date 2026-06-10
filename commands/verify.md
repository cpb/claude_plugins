---
description: Verify that a code change actually does what it's supposed to by running the app and observing behavior. Use when asked to verify a PR, confirm a fix works, test a change manually, check that a feature works, or validate local changes before pushing.
argument-hint: [item-text --port N --server-up]
---

Confirm that the application behaves as expected — either for one specific test-plan item (when called with arguments) or holistically from the diff (when called with no arguments).

## Steps

**1. Determine the mode**

If `$ARGUMENTS` is non-empty, enter **item mode**. Otherwise enter **discovery mode**.

---

### Item mode

**2i. Parse the arguments**

Split `$ARGUMENTS` on newlines:

- **Item text**: all content before the first line that starts with `--`, trimmed.
- **Port**: the value following `--port` (e.g. `--port 3001` → `3001`). If absent, read `PORT` from `.env.local`.
- **Server-up flag**: present if `--server-up` appears anywhere in the arguments.

**3i. Start the server if needed**

If `--server-up` was provided, skip this step — the server is already running.

Otherwise check:
```bash
status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/up 2>/dev/null)
```
If `status` is not `200`, run:
```bash
bin/worktree server $(git branch --show-current)
```
Poll `/up` every 2 seconds, up to 10 attempts. If still not up after 10 s, abort with:
```
ERROR: dev server did not start within 10 s. Check log/dev_server.log for details.
```

**4i. Confirm the item — do NOT read git diff**

The item text IS the specification. Based on what it describes, use the cheapest method:

- **Route, URL, or HTTP status** → curl and check the status code:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" http://localhost:<port><path>
  ```
- **File or git state** → bash check (`ls`, `git log --oneline -3`, etc.)
- **Form submit, redirect, visible content, or any UI interaction** → snapshot-first browser approach:
  1. Load tools: `ToolSearch: select:mcp__playwright__browser_navigate,mcp__playwright__browser_snapshot,mcp__playwright__browser_fill_form,mcp__playwright__browser_click,mcp__playwright__browser_wait_for,mcp__playwright__browser_take_screenshot`
  2. `mcp__playwright__browser_navigate { url: "http://localhost:<port>/" }` (or the relevant path)
  3. `mcp__playwright__browser_snapshot {}` — read the accessibility tree to identify affordances (inputs, buttons, links, headings, live regions). If an expected affordance is absent, record what IS present as the reconciliation map and note the gap.
  4. Execute the interaction using selectors from the tree: `browser_fill_form`, `browser_click`, `browser_wait_for` as needed
  5. `mcp__playwright__browser_snapshot {}` again to read the outcome (current URL, new elements, updated regions)
  6. Take one `mcp__playwright__browser_take_screenshot {}` for operator evidence

Report any accessibility gaps (missing labels, roles, or expected elements not in the tree) as secondary findings alongside the pass/fail verdict.

**5i. Report**

Print one line:
```
✓ Confirmed: <item text>
```
or:
```
✗ Not confirmed: <item text> — <one sentence describing what was observed instead>
```
Follow with any accessibility gap notes if found.

---

### Discovery mode

**2d. Read the diff**

```bash
git diff main...HEAD
```

Identify what has changed: routes, controllers, views, models, migrations, config.

**3d. Start the app**

Invoke the `run` skill (auto-dispatches to the project run skill when available; falls back gracefully for older worktrees).

**4d. Exercise the changes**

For each changed behavior, use the snapshot-first approach from step 4i above — `browser_snapshot` for observation, `browser_take_screenshot` only for operator evidence. Cover the full scope of what changed: affected routes, form interactions, renamed or removed paths, visible content.

**5d. Report**

Summarize what was checked and whether each check passed or revealed a discrepancy. Flag any accessibility gaps observed.
