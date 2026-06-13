---
description: Open a worktree + tmux window with Claude (or Gemini) in plan mode to tackle a GitHub issue
argument-hint: <issue-number> [--gemini]
---

Set up an isolated worktree and launch Claude (or Gemini if --gemini is specified) in plan mode, primed with the full issue description, for issue $ARGUMENTS.

## Steps

**1. Prepare the worktree**

```bash
if [ -z "$ARGUMENTS" ]; then echo "Usage: /start-pr <issue-number> [--gemini]"; exit 1; fi

bin/worktree prepare "$1" --issue > /tmp/issue.json
```

Then read the fields you need:

```bash
jq -r '.title' /tmp/issue.json
jq -r '.url' /tmp/issue.json
jq -r '.remote_control_name' /tmp/issue.json
```

**2. Launch the harness**

```bash
bin/worktree harness "$1" ${2:---}
```

**3. Print a confirmation**

Print a summary including the worktree path, issue title/URL, and the remote control name from the JSON.
Mention that the agent is in plan mode.
