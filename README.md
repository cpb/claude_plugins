# cpb-dev-workflow

A Claude Code plugin providing general-purpose GitHub development workflow skills and worktree tooling.

## Skills

All skills are namespaced as `/cpb-dev-workflow:<name>`.

| Skill | Purpose |
|---|---|
| `/cpb-dev-workflow:start-pr <n>` | Create worktree + plan-mode Claude session for a GitHub issue |
| `/cpb-dev-workflow:continue-pr <n>` | Create worktree + Claude session for an open PR |
| `/cpb-dev-workflow:finish-pr [n]` | CI gate → squash-merge → worktree teardown |
| `/cpb-dev-workflow:qa-pr [n]` | Router: detect PR mode, delegate to sub-commands, finish on pass |
| `/cpb-dev-workflow:qa-pr-skill <n>` | Skill-contribution walkthrough via automated Claude session |
| `/cpb-dev-workflow:qa-pr-app <n>` | App-change walkthrough via dev server + browser automation |
| `/cpb-dev-workflow:new-issue [topic]` | Elicit intent and create a GitHub issue |
| `/cpb-dev-workflow:hill-first <layers>` | Write failing tests per layer, open draft PR for review |
| `/cpb-dev-workflow:research-pr <n>` | Spawn parallel research agents, gate plan agents, open draft PR |
| `/cpb-dev-workflow:kaizen-handoff [n]` | Generate narrative handoff prompt for a fresh session |
| `/cpb-dev-workflow:verify [item]` | Verify a code change by running the app and observing behavior |
| `/cpb-dev-workflow:heal-ci <n>` | Poll CI, reproduce failures, classify root cause, fix and push until green |
| `/cpb-dev-workflow:init` | Scaffold project-specific files for a new project |

## Installation

### Via skills directory (simplest — no marketplace step)

```bash
git clone https://github.com/cpb/claude_plugins ~/.claude/skills/cpb-dev-workflow
```

Loads automatically as `cpb-dev-workflow@skills-dir` on the next Claude Code session.

### Via marketplace

```
/plugin marketplace add cpb/claude_plugins
/plugin install cpb-dev-workflow@cpb
```

### Local testing

```bash
claude --plugin-dir /path/to/cpb/claude_plugins
```

## Project setup

The workflow skills rely on three conventions your project must implement:

| Script | Called by | Purpose |
|---|---|---|
| `bin/setup` | `bin/worktree add` | Set up a new worktree (install deps, prepare DB, etc.) |
| `bin/dev` | `bin/worktree server` | Start the development server on `$PORT` |
| `bin/test` | `bin/worktree heal-reproduce` | Run a test by file:line — editable; plugin provides a default that calls `bin/rails test` |
| `bin/check-worktree` | Hook (PreToolUse) | Block edits on main branch — editable; plugin provides generic fallback |
| `bin/claude-code-web-setup` | Hook (PreToolUse, remote only) | Install deps in web sessions — editable; plugin provides no-op fallback |
| `bin/lint` | Hook (PostToolUse) | Auto-format edited files — editable; plugin provides no-op fallback |

Run `/cpb-dev-workflow:init` in a new project to scaffold `bin/check-worktree`, `bin/claude-code-web-setup`, `bin/lint`, and `bin/test` as editable starters.

### bin/setup

Called as `bin/setup --skip-server` after `worktree add` creates a new worktree. Minimum example:

```bash
#!/bin/bash
set -e
bundle install
bin/rails db:prepare
```

### bin/dev

Called by `worktree server` to start the dev server in the background. Must respect `$PORT`. Minimum example:

```bash
#!/bin/bash
exec bundle exec rails server -b 0.0.0.0 -p "${PORT:-3000}"
```

### bin/test

Called by `bin/worktree heal-reproduce` to run a single test. Receives the file:line argument. Default calls `bin/rails test`. Override to use a different runner:

```bash
#!/usr/bin/env bash
exec bin/rails test "$@"
```

### bin/check-worktree

The plugin ships a generic version that blocks `Edit`/`Write` on the `main` branch. After running `/cpb-dev-workflow:init`, edit `./bin/check-worktree` to customize the behavior (e.g., change the protected branch name or the error message).

### bin/claude-code-web-setup

The plugin ships a no-op skeleton. After running `/cpb-dev-workflow:init`, implement project-specific web session setup (e.g., `bundle install`, `hk install`). Use `tmp/claude-web-receipts/` for receipt-file caching to avoid re-running on every tool call.

## Hooks

The plugin registers three hooks via `hooks/hooks.json`, each dispatching to a project-owned script:

| Hook | Trigger | Project script | Behavior when missing |
|---|---|---|---|
| PreToolUse | All tools | `bin/check-worktree` | Falls back to plugin's generic version |
| PreToolUse | Edit/Write (remote only) | `bin/claude-code-web-setup` | Falls back to plugin's no-op skeleton |
| PostToolUse | Edit/Write | `bin/lint` | Falls back to plugin's no-op skeleton |

Each hook's only guard is whether the project script exists. No other conditions are checked — put project-specific guards (tool availability, config file presence, etc.) inside the script itself.

The plugin's `bin/` is added to the **Bash tool's PATH only** — hook processes use `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PROJECT_DIR}` for path resolution. Run `/cpb-dev-workflow:init` to install editable copies of all three scripts into your project's `./bin/`.

### bin/lint

`bin/lint` is the project's PostToolUse formatting hook. `$CLAUDE_FILE_PATHS` is available as an environment variable containing the space-separated paths of files modified by the tool call.

`/cpb-dev-workflow:init` creates this boilerplate:

```bash
#!/usr/bin/env bash
# Default: hk fix — replace with your project's formatter
HK_PKL_BACKEND=pklr hk fix $CLAUDE_FILE_PATHS
```

Replace the body with whatever your project uses: `prettier --write`, `rubocop --autocorrect`, `ruff check --fix`, etc.

## Session launch path

Skills follow a two-step pattern:

1. **`bin/worktree prepare <id>`** — fetches GitHub info, creates or locates the worktree, writes `.worktree-session.json` and `pr_context.md`, and prints session JSON.

2. **`bin/worktree harness <id>`** — reads the session JSON, creates a tmux window in the worktree, writes a launcher script to `/tmp/harness-<rc>.sh`, and sends `bash /tmp/harness-<rc>.sh` to the window.

Skills capture the prepare output into a shell variable so no `/tmp/*.json` intermediary is needed:

```bash
session=$(bin/worktree prepare "$1" --issue)
wt_path=$(echo "$session" | jq -r '.worktree_path')
```

### Harness flags

| Flag | Default | Description |
|---|---|---|
| `--prompt-file <path>` | `<wt_path>/pr_context.md` | Prompt file to pass to Claude/Gemini |
| `--window <name>` | `session["tmux_window"]` | Override the tmux window name |
| `--fresh` | false | Kill any existing window with that name before creating a new one |
| `--gemini` | false | Launch Gemini instead of Claude |

The launcher script avoids quoting breakage: instead of inlining the prompt through `tmux send-keys` (which can break on backticks, newlines, and quotes), the full command is written to a bash script and only `bash /tmp/harness-<rc>.sh` is sent through tmux.

## Worktree commands

`bin/worktree` is added to `PATH` when the plugin is active (callable as `worktree`). Skills call it as `bin/worktree` (relative path), so install it into your project's `bin/` via `/cpb-dev-workflow:init`.

### Session launch path

All session-launching skills (`start-pr`, `continue-pr`, `research-pr`, `qa-pr-skill`) share one launch path:

1. **`bin/worktree prepare <id>`** — fetches GitHub info, creates the worktree, writes `pr_context.md` and `.worktree-session.json`, and prints a JSON object on stdout. Skills capture this with `session=$(bin/worktree prepare …)` and read fields with `jq -r '.<field>'`.

2. **`bin/worktree harness <id> [flags]`** — finds or creates the tmux window, writes a `/tmp/harness-<rc>.sh` launcher script, and sends the script path via `tmux send-keys`. All flags:

   | Flag | Default | Description |
   |---|---|---|
   | `--prompt-file <path>` | `<worktree>/pr_context.md` | Prompt file to feed to the agent |
   | `--window <name>` | Session's `tmux_window` | Override the tmux window name |
   | `--fresh` | (reuse existing) | Kill any existing window with the target name, then create a clean one |
   | `--gemini` | (use Claude) | Launch Gemini instead of Claude |

   The launcher-script mechanism (`/tmp/harness-<rc>.sh`) avoids the shell re-parsing that occurs when long prompts are inlined through `tmux send-keys`. Prompts containing backticks, newlines, or quotes are handled correctly.

### Worktree directory naming

New worktrees are created adjacent to the main repo as `<app-name>_<branch>`, where `<app-name>` is derived from the git root directory name. For a repo at `~/projects/my-app`, a branch `feature-x` gets a worktree at `~/projects/my-app_feature-x`.

### Port management

Each worktree gets an auto-assigned port ≥ 3001, written to `.env.local`. Skills read the port from `.env.local` using:

```bash
grep "^PORT=" .env.local | cut -d= -f2
```

The health check endpoint is `GET /up` → 200.

## Migrating from standalone .claude/commands/

If you're migrating a project that already has these skills as standalone `.claude/commands/` files:

1. Install the plugin (see above)
2. Update cross-skill references in your local skill files to use the namespaced form, e.g.:
   - `invoke the \`qa-pr-skill\` skill` → `invoke the \`cpb-dev-workflow:qa-pr-skill\` skill`
3. Delete the local copies from `.claude/commands/` once confirmed working
4. Invocation syntax changes from `/start-pr` to `/cpb-dev-workflow:start-pr`

The standalone and plugin versions can coexist temporarily — the local `.claude/commands/` files take precedence over the plugin versions for same-named skills.
