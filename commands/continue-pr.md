---
description: Create a worktree + tmux window + Claude (or Gemini) session primed with a PR's description
argument-hint: <pr-number> [--gemini]
---

Set up an isolated worktree and launch a Claude (or Gemini if --gemini is specified) session primed with the full PR description for PR $ARGUMENTS.

## Steps

**1. Check prerequisites and prepare worktree**

```bash
if [ -z "$ARGUMENTS" ]; then echo "Usage: /continue-pr <pr-number> [--gemini]"; exit 1; fi

session=$(bin/worktree prepare "$1" --pr)
wt_path=$(echo "$session" | jq -r '.worktree_path')
title=$(echo "$session" | jq -r '.title')
url=$(echo "$session" | jq -r '.url')
remote_control=$(echo "$session" | jq -r '.remote_control')
hill_ready=$(echo "$session" | jq -r '.hill_ready')
```

**2. Check for hill-ready gate**

If `$hill_ready` is `true`, this PR is a hill under review. Enter the following loop:

**a) Wait for CI and report status:**

```bash
bin/worktree check-hill "$1"
```

**b) Reflect and elicit feedback with `AskUserQuestion`:**

Present a summary of what CI shows versus what was expected (use inference to explain the report).

**c) On "Looks correct — I'll remove `hill-ready` now":**

Wait for the operator to remove the label, then confirm. Once gone, proceed.

**3. Launch the harness**

```bash
bin/worktree harness "$1" ${2:+$2}
```

**4. Print a confirmation**

Print a summary including the worktree path, PR title/URL, and remote control name from `$remote_control`.
