---
description: Scaffold the project-specific files needed to use cpb-dev-workflow in a new project
argument-hint: [--force]
---

Set up a new project to use the cpb-dev-workflow plugin by creating the required bin/ scripts and documenting what to customize.

## Steps

**1. Confirm this is a git repository**

```bash
git rev-parse --show-toplevel
```

Abort if this is not a git repository.

**2. Check for existing files**

Check which of these already exist: `bin/worktree`, `bin/check-worktree`, `bin/claude-code-web-setup`, `bin/setup`, `bin/dev`.

If `bin/worktree` already exists and `--force` is not in `$ARGUMENTS`, print:
```
bin/worktree already exists. Pass --force to overwrite.
```
and stop.

**3. Ensure bin/ directory exists**

```bash
mkdir -p bin
```

**4. Install bin/check-worktree**

Find the plugin's `check-worktree` binary on PATH:

```bash
which check-worktree
```

Copy it into `./bin/check-worktree`:

```bash
cp "$(which check-worktree)" bin/check-worktree
chmod +x bin/check-worktree
```

If `which check-worktree` fails, print a warning:
```
Warning: check-worktree not found on PATH. The plugin's bin/ may not be active.
Run claude with --plugin-dir or install via /plugin install.
```

**5. Install bin/claude-code-web-setup**

Find the plugin's `claude-code-web-setup` binary:

```bash
which claude-code-web-setup
```

Copy it into `./bin/claude-code-web-setup`:

```bash
cp "$(which claude-code-web-setup)" bin/claude-code-web-setup
chmod +x bin/claude-code-web-setup
```

Tell the operator:
```
bin/claude-code-web-setup installed as a no-op skeleton.
Edit it to add your project's web-session setup (bundle install, npm install, etc.).
```

**6. Check for bin/setup and bin/dev**

If `bin/setup` does not exist, print:
```
Warning: bin/setup is missing. bin/worktree add will call bin/setup --skip-server after
creating a new worktree. Create bin/setup to automate dependency installation and database
setup for new worktrees.
```

If `bin/dev` does not exist, print:
```
Warning: bin/dev is missing. bin/worktree server calls bin/dev to start the development server.
Create bin/dev (e.g. exec rails server -b 0.0.0.0 -p $PORT) for the server commands to work.
```

**7. Print summary**

Print a checklist:

```
cpb-dev-workflow project setup complete.

Created:
  bin/check-worktree        — enforces worktree-first development (edit to customize)
  bin/claude-code-web-setup — web session setup skeleton (edit with your project's dependencies)

Required (create if missing):
  bin/setup                 — called by `bin/worktree add` to set up a new worktree
  bin/dev                   — called by `bin/worktree server` to start the dev server

Plugin hooks (active when plugin is loaded):
  PreToolUse  → check-worktree          blocks edits on main branch
  PreToolUse  → claude-code-web-setup   runs project setup in web sessions
  PostToolUse → hk fix                  formats edited files (requires hk + .hk.toml)

To use worktree commands, add the plugin's bin/ to your project PATH, or call:
  worktree add <branch>
  worktree server <branch>
  worktree list
```
