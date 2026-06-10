---
description: Elicit intent for new work, reach mutual understanding, and create a GitHub issue
argument-hint: [topic or seed idea]
---

Guide the user from a rough idea to a well-understood, documented GitHub issue using structured goal elicitation.

## Steps

**1. Check prerequisites**

```bash
gh auth status
```

Abort if not authenticated.

**2. Seed the conversation**

If `$ARGUMENTS` is non-empty, use it as the starting point. Otherwise start with an open question.

**3. Elicit the goal**

Use `AskUserQuestion` to reach mutual understanding. Aim for 2–4 focused rounds — stop as soon as you have enough to write a crisp issue. Cover these dimensions (combine into fewer questions when possible):

- **Problem / motivation** — What situation or pain prompted this? What goes wrong today without this?
- **Success criteria** — What does done look like? How will we know it's working?
- **Scope** — What's explicitly in scope? What's out of scope or saved for later?
- **Constraints** — Any technical, time, or stakeholder constraints worth capturing?

Adapt based on what the seed already answers. If $ARGUMENTS covers motivation clearly, skip that question.

**4. Confirm your understanding**

Before drafting, state your synthesis in 2–3 sentences and ask the user to confirm or correct it:

> "So the goal is: [one sentence]. Done means: [one sentence]. Out of scope: [one sentence]. Is that right?"

Use `AskUserQuestion` for this confirmation. Revise until the user confirms.

**5. Draft the issue**

Write a GitHub issue with this structure:

```markdown
## Goal

[One sentence: what we want to achieve and why.]

## Background

[2–4 sentences of context and motivation. What prompted this? What breaks or is painful today?]

## Acceptance criteria

- [ ] [Concrete, testable condition]
- [ ] [Concrete, testable condition]
- [ ] ...

## Out of scope

- [Item saved for later, if any]

## Notes

[Any constraints, open questions, or references worth capturing — omit section if empty.]
```

Derive the title from the goal sentence (concise, imperative, ≤ 72 characters).

Also infer whether this issue warrants a test-first hill:
- **Recommend yes** if the issue specifies new or corrected behavior expressible as assertions — features, behavioral bugs, protocol/API changes, UI interactions.
- **Recommend no** for chores, refactors, documentation updates, dependency bumps, configuration changes, or tooling work where behavior is not the primary deliverable.

Note the recommendation and a one-sentence rationale; show both with the draft in step 6.

**6. Show the draft and get approval**

Display the proposed title and body, then show the test-first recommendation:

> Test-first hill: **Yes** *(reason)* — or — **No** *(reason)*

Use `AskUserQuestion` to collect approval. Offer four options, putting the recommended test-first setting first:

- Approve (test-first: Yes) — create the issue with the `test-first` label
- Approve (test-first: No) — create the issue without the label
- Edit content — I'll describe what to change
- Start over

Revise and re-show until the user selects one of the Approve options. Record the final choice as `test_first` (true / false).

**7. Create the issue**

If `test_first` is true, ensure the label exists first:

```bash
gh label create "test-first" \
  --description "Requires a test-first failing-test hill before implementation" \
  --color "FF6B35" \
  --force
```

Then create the issue, including the label in the same command if needed:

```bash
# test_first = false:
gh issue create --title "<title>" --body "<body>"

# test_first = true:
gh issue create --title "<title>" --body "<body>" --label "test-first"
```

**8. Print a confirmation**

```
Created issue #<number>: <title>
<url>
<if test_first>Labels: test-first</if>

To start work on it now: /start-pr <number>
```
