---
description: Wait for CI, merge a PR, and clean up its worktree and tmux window
argument-hint: [pr-number]
---

Finish a PR: block until CI completes, merge on green, tear down the worktree, and close the tmux window.

## Steps

**1. Cleanup the PR**

```bash
bin/worktree cleanup $ARGUMENTS
```
