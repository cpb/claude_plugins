---
description: Write failing tests per layer, open draft PR for human hill review
argument-hint: <name>: "<spec>", <name2>: "<spec2>", ...
---

Orchestrate an outside-in test hill: create one worktree per test layer, spawn parallel subagents to write a single failing test each, merge the results, and open a draft PR for human review.

## Steps

**1. Validate prerequisites**

```bash
if [ -z "$ARGUMENTS" ]; then
  echo 'Usage: /hill-first name: "spec sentence", name2: "spec sentence", ...'
  exit 1
fi
```

**2. Parse layers**

Parse `$ARGUMENTS` as a comma-separated list of `name: "spec"` pairs. For each entry:
- Slug the name: lowercase, replace non-alphanumeric runs with `-`, strip leading/trailing hyphens.
- Store as an ordered list of `(slug, name, spec)` triples.

Example input:
```
controller-routing: "GET /fizzbuzz returns 200", model-validation: "FizzBuzz rejects blank input"
```

**3. Determine parent context**

```bash
parent_branch=$(git branch --show-current)
jq -r '.number' .worktree-session.json > /tmp/hill-parent-number.txt 2>/dev/null
grep -qE '[0-9]' /tmp/hill-parent-number.txt || echo "$parent_branch" | grep -oE '[0-9]+' | head -1 > /tmp/hill-parent-number.txt
```

Reference `$(cat /tmp/hill-parent-number.txt)` wherever `$parent_number` is used.

**4. Create a task list**

Use `TaskCreate` to create:
- One task per layer: `"Write failing test for layer: <name>"`
- One task: `"Merge all layer branches and run aggregate check"`
- One task: `"Push and open draft PR"`

**5. Per layer: create worktree and spawn subagent**

For each `(slug, name, spec)`:

```bash
branch="hill/$parent_branch/$slug"
bin/worktree add --parent "$parent_branch" "$branch"
git worktree list --porcelain \
  | grep -B2 "branch refs/heads/$branch" \
  | grep "^worktree" | sed 's/worktree //' > /tmp/hill-wt-path-$slug.txt
```

Reference `$(cat /tmp/hill-wt-path-$slug.txt)` wherever `$wt_path` is used (including in the subagent prompt).

Then use the `Agent` tool (`run_in_background: true`) with this prompt, substituting `<wt_path>`, `<name>`, `<spec>`, and `<slug>` with concrete values:

```
You are writing a single outside-in failing test for one hill layer.
Work exclusively inside the worktree at: <wt_path>
All file reads, writes, and Bash commands must use that path.

Layer name: <name>
Spec: <spec>

## Strict discipline — follow exactly, in order

1. Write ONE test in the appropriate test file that specifies the behavior described
   in Spec. Name the real class, route, or method you expect to exist — outside-in.
   Do not create any production code yet.

2. Run the test:
   ```bash
   cd <wt_path> && bin/rails test <test-file>
   ```
   The first failure will be a LOAD ERROR (NameError, NoMethodError, routing error,
   LoadError). This is expected — record the exact error message.

3. Add the MINIMUM stub the error message reveals:
   - An empty constant: `FooBar = nil` or `class FooBar; end`
   - An empty method: `def foo_bar; end`
   - A route entry that maps the path to a stub action
   No logic, no return values, no implementation.

4. Run the test again. Repeat steps 2–3 until the failure changes from a load error
   to an ASSERTION failure (assert_equal mismatch, Expected…got…, etc.).

5. Read the assertion failure message carefully. Confirm it expresses the intent of Spec.
   If the wording is misleading, adjust the test assertion — without adding any implementation.

6. STOP. Do not implement logic. Do not make the test pass.

7. Commit in two steps inside <wt_path>:
   a. Stage only stub files:
      git -C <wt_path> add <stub-files>
      git -C <wt_path> commit -m "stub: <slug> — minimum stubs for hill"
   b. Stage only the test file:
      git -C <wt_path> add <test-file>
      git -C <wt_path> commit -m "test: <slug> — failing test for hill"

8. Return your result as the very last lines of your response, in this exact format:
   HILL_RESULT: <slug>
   <verbatim assertion failure message, 1–3 lines copied exactly from test output>
   END_HILL_RESULT
```

Spawn all layer agents in parallel using `run_in_background: true`.

**6. Collect results**

Wait for all background agents to complete. For each agent result:
- Find the `HILL_RESULT: <slug>` … `END_HILL_RESULT` block.
- If any agent's result is missing or malformed, stop and report which layer failed.

Store results keyed by slug.

**7. Update layer tasks**

Call `TaskUpdate` on each layer task, marking it complete with the first line of its failure message as a note.

**8. Merge all layer branches**

```bash
for each layer:
  git merge --no-ff "hill/$parent_branch/$slug" \
    -m "hill: merge layer $slug"
```

On any merge conflict, stop and report which layer conflicted. Do not push.

**9. Run aggregate test check**

```bash
bin/rails test 2>&1 | tail -30
```

Confirm tests exit non-zero (failures expected). If all tests pass, something went wrong — stop and warn the operator.

Call `TaskUpdate` on the merge task, marking it complete.

**10. Push and open draft PR**

Ensure the `hill-ready` label exists:

```bash
gh label create "hill-ready" \
  --description "Hill under review — remove this label to signal approval and begin implementation" \
  --color "0E8A16" \
  --force
```

Push the branch and create the draft PR; capture the new PR number:

```bash
git push -u origin "$parent_branch"

hill_pr_url=$(gh pr create \
  --title "hill: failing tests for issue #$(cat /tmp/hill-parent-number.txt)" \
  --body "<body>" \
  --draft)

gh pr view --json number --jq '.number' > /tmp/hill-pr-number.txt
```

The PR body must include:

```markdown
## Hill: failing tests for issue #<parent_number>

This draft PR contains only failing tests — one commit of stubs and one of tests per
layer. CI should confirm failures. **Do not merge — implementation happens on this branch.**

## Expected failure messages

### <layer-name-1>
```
<verbatim failure message>
```

### <layer-name-2>
```
<verbatim failure message>
```

## Reviewer instructions

Review each expected failure message above. If they correctly specify the intended
behavior per the issue, **remove the `hill-ready` label** from this PR — that signals
the implementation session to begin. Run `/continue-pr <hill_pr_number>` to start it.
```

Apply `hill-ready` to signal the PR is ready for human review:

```bash
gh pr edit $(cat /tmp/hill-pr-number.txt) --add-label "hill-ready"
```

Call `TaskUpdate` on the draft-PR task, marking it complete.

**11. Tear down layer worktrees**

```bash
bin/worktree down-children "$parent_branch"
```

**12. Post instructions as a GitHub comment, then self-destruct**

Post a comment on the hill draft PR so the instructions survive after this session closes:

```bash
gh pr comment $(cat /tmp/hill-pr-number.txt) --body "$(cat <<'COMMENT'
## Hill ready for review

CI is running — wait for it to confirm the failures are **assertion failures** (not load errors).

**To proceed:**
1. Review the expected failure messages in the PR description.
2. Open `/continue-pr <hill_pr_number>` — that session will inspect CI and ask whether the hill is correctly specified.
3. If something is missing, describe it and the session will fix the hill.
4. When satisfied, **remove the `hill-ready` label** — `/continue-pr` will detect the removal and switch to implementation planning.
COMMENT
)"
```

Schedule the current tmux window to close 3 seconds from now:

```bash
tmux display-message -p "#{session_name}:#{window_index}" > /tmp/hill-current-window.txt
(sleep 3 && tmux kill-window -t "$(cat /tmp/hill-current-window.txt)") &
```

Print:

```
Hill draft PR: <hill_pr_url>
`hill-ready` applied. Instructions posted as a PR comment.
This session is closing in 3 seconds.
```

Then stop — make no further tool calls.
