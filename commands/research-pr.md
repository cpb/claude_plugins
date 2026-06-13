---
description: Prime a worktree for a research-and-planning session; spawn parallel research agents, gate plan agents on their output, and open a draft PR of terrain-map documents
argument-hint: <issue-number> [--gemini]
---

Set up an isolated worktree for issue $ARGUMENTS and launch Claude (or Gemini if --gemini is specified) in plan mode, primed with the issue description and a research-and-planning brief that instructs the orchestrator to fan out parallel research agents, gate plan agents on their output, and open a draft PR of hierarchical markdown artifacts.

## Steps

**1. Validate arguments**

```bash
if [ -z "$ARGUMENTS" ]; then
  echo "Usage: /research-pr <issue-number> [--gemini]"
  exit 1
fi
```

**2. Prepare the worktree**

Write the following research brief to `/tmp/cpb-research-context.md`:

```
---

## Research-and-Planning Protocol

This is a **research-and-planning session**, not an implementation session. Your
deliverable is a hierarchical document tree that maps the terrain and proposes
grounded plans. Do not write application code.

### Document hierarchy

Create this exact structure under `docs/<issue>-<slug>/` where `<slug>` is derived
from the issue title using the same slug rules as branch names (lowercase,
non-alphanumeric runs replaced with `-`):

    docs/<issue>-<slug>/
      README.md                    ← entry point: what this PR answers, reading order
      research/
        README.md                  ← breadth-first summary of all research topics + TOC
        <topic>/
          README.md                ← summary + TOC for this topic
          <subtopic>.md            ← deep-dive leaf document
      plans/
        README.md                  ← breadth-first summary of all plans + TOC
        <nn>-<plan-slug>.md        ← individual plan (≤ 200 lines)

### Research doc rules

Research docs are **ground-truth terrain maps** readable independently by a human:

- **Current terrain** — observable facts about the system as built: stack versions,
  config values, file paths, measured numbers, embedded links to source.
- **New territory** — options not yet adopted: libraries, patterns, migration paths —
  compared factually (versions, benchmarks, trade-offs), not prescribed.
- Include **Mermaid diagrams** wherever they clarify structure (architecture maps,
  migration paths, decision trees, component relationships).
- Every claim is verifiable: include specific version numbers, file paths, line refs.

### Plan doc rules (≤ 200 lines each)

- Cross-link to relevant research sections.
- State: **Decision**, **Rationale**, **Steps**, **Open questions**.
- Ground every decision in research findings.

### Every README must include

- A **table of contents** linking to every child document.
- A one-sentence description of each child.

### Orchestration instructions

1. **Parse the issue** to identify 3–6 research topics — each a specific question
   one agent can answer in one worktree session.

2. **Create tasks** using `TaskCreate`:
   - One task per research topic: `"Research: <topic>"`
   - One task per plan: `"Plan: <plan-slug>"` (add after research topics are known)
   - One task: `"Write top-level READMEs, commit, and open draft PR"`

3. **Spawn research agents in parallel** using the `Agent` tool:
   - `isolation: "worktree"` and `run_in_background: true`
   - Each agent must: write its topic subtree under `docs/<issue>-<slug>/research/<topic>/`,
     write a topic README with TOC, commit with message `"research(<topic>): <summary>"`,
     and return this sentinel as the **very last lines** of its response:
       RESEARCH_RESULT: <topic-slug>
       <one-sentence summary of findings>
       END_RESEARCH_RESULT

4. **Collect all research results** before proceeding. If any result is missing or
   malformed, stop and report which agent failed.

5. **Update research task statuses** via `TaskUpdate`.

6. **Spawn plan agents** (after all research is complete). Each plan agent reads the
   research subtree, writes one plan doc, commits with `"plan(<slug>): <summary>"`,
   and returns:
       PLAN_RESULT: <plan-slug>
       <one-sentence summary>
       END_PLAN_RESULT

7. **Write top-level READMEs** assembling the full TOC tree:
   - `docs/<issue>-<slug>/README.md` (entry point with reading order)
   - `docs/<issue>-<slug>/research/README.md` (breadth-first summary + TOC)
   - `docs/<issue>-<slug>/plans/README.md` (breadth-first summary + TOC)
   Commit: `"docs(<issue>): top-level READMEs and entry point"`

8. **Open a draft PR**:
   ```
   git push -u origin HEAD
   gh pr create \
     --title "research(<issue>): <issue-title>" \
     --body "Research and planning artifacts for issue #<issue>. Entry point: docs/<issue>-<slug>/README.md" \
     --draft
   ```

### Quality signal

TOC completeness is the key quality indicator: every link in every README TOC must
resolve to an existing file; every leaf document must be referenced by at least one
TOC. A broken link means an artifact is missing; an orphaned doc means planning
coverage is incomplete.
```

Then prepare the worktree with the research brief appended to `pr_context.md`:

```bash
bin/worktree prepare "$1" --issue --append-context-file /tmp/cpb-research-context.md
```

Note the `worktree_path`, `title`, `url`, and `remote_control` from the JSON output.

**3. Launch the harness**

```bash
bin/worktree harness "$1" ${2:---}
```

**4. Print a confirmation**

Print:
- Worktree path
- Issue title and URL
- Remote control name
- Note that the orchestrator is in plan mode and will propose a research scope before spawning agents

## Test plan

- [ ] Running `/research-pr <n>` creates a worktree for issue `<n>`
- [ ] `pr_context.md` in the worktree contains the heading `## Research-and-Planning Protocol`
- [ ] `.worktree-session.json` in the worktree has `"type": "issue"` (confirming plan-mode launch)
- [ ] A tmux window named `issue-<n>` is open with a Claude session running
- [ ] (inspect only) Skill file defines the hierarchical folder structure `docs/<issue>-<slug>/research/` and `docs/<issue>-<slug>/plans/`
- [ ] (inspect only) Skill file instructs research agents to use `isolation: "worktree"` and `run_in_background: true`
- [ ] (inspect only) Skill file gates plan agents on all research agents completing (step 6 follows step 4 collect)
- [ ] (inspect only) Skill file instructs opening a draft PR after top-level READMEs are committed
- [ ] (inspect only) Research brief defines ground-truth terrain types (current terrain + new territory) and Mermaid diagram requirement
- [ ] (inspect only) Research brief requires every README to include a TOC linking all children
