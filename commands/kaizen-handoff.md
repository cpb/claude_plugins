---
description: Generate a narrative handoff prompt for a fresh session using automated context gathering
argument-hint: [pr-number]
---

Gather PR and git context via bin/worktree and synthesize a narrative handoff prompt to continue work in a fresh session.

## Steps

**1. Gather context**

```bash
bin/worktree handoff $ARGUMENTS
```

**2. Synthesize handoff**

Using the JSON output from step 1, generate a high-quality, narrative handoff prompt following this structure:
- "We're on branch [branch] accumulating fixes... [PR context]"
- "What's landed so far ([count] commits ahead of main): [list of commits]"
- "Key files: [list of modified files]"
- "To continue: [proactive next steps based on PR body and current state]"

Produce the handoff prompt inside a fenced code block for easy copying.
