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
| `bin/check-worktree` | Hook (PreToolUse) | Block edits on main branch (project-editable copy) |
| `bin/claude-code-web-setup` | Hook (PreToolUse, remote only) | Install deps before tool calls in web sessions |

Run `/cpb-dev-workflow:init` in a new project to scaffold `bin/check-worktree` and `bin/claude-code-web-setup` as editable starters.

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

### bin/check-worktree

The plugin ships a generic version that blocks `Edit`/`Write` on the `main` branch. After running `/cpb-dev-workflow:init`, edit `./bin/check-worktree` to customize the behavior (e.g., change the protected branch name or the error message).

### bin/claude-code-web-setup

The plugin ships a no-op skeleton. After running `/cpb-dev-workflow:init`, implement project-specific web session setup (e.g., `bundle install`, `hk install`). Use `tmp/claude-web-receipts/` for receipt-file caching to avoid re-running on every tool call.

## Hooks

The plugin registers three hooks via `hooks/hooks.json`:

| Hook | Trigger | Command |
|---|---|---|
| PreToolUse | All tools | `check-worktree` — blocks edits on main branch |
| PreToolUse | Edit/Write (remote only) | `claude-code-web-setup` — runs web session setup |
| PostToolUse | Edit/Write | `hk fix $CLAUDE_FILE_PATHS` — auto-formats edited files |

The PostToolUse `hk fix` hook requires [`hk`](https://github.com/jdx/hk) installed and a `.hk.toml` in the project root. Remove or override this hook in your project's `.claude/settings.json` if your project doesn't use `hk`.

## Worktree commands

`bin/worktree` is added to `PATH` when the plugin is active (callable as `worktree`). Skills call it as `bin/worktree` (relative path), so install it into your project's `bin/` via `/cpb-dev-workflow:init`.

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
