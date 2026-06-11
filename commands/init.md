---
description: Scaffold the project-specific hook scripts needed to use cpb in a new project
argument-hint: [--force]
---

Set up a new project to use the cpb plugin by creating editable hook scripts in bin/ and documenting what to customize.

## Steps

**1. Confirm this is a git repository**

```bash
git rev-parse --show-toplevel
```

Abort if this is not a git repository.

**2. Check for existing files**

Check which of these already exist: `bin/check-worktree`, `bin/claude-code-web-setup`, `bin/lint`.

If any exist and `--force` is not in `$ARGUMENTS`, print which ones exist and stop:
```
The following hook scripts already exist: <list>
Pass --force to overwrite them.
```

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

If `which check-worktree` fails, print a warning and skip:
```
Warning: check-worktree not found on PATH. The plugin's bin/ may not be active.
Run claude with --plugin-dir or install via /plugin install.
```

**5. Install bin/claude-code-web-setup**

Find the plugin's `claude-code-web-setup` binary on PATH:

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

**6. Create bin/lint**

Write the following boilerplate to `./bin/lint` and make it executable:

```bash
#!/usr/bin/env bash
# PostToolUse hook — called by cpb after every Edit/Write tool call.
# $CLAUDE_FILE_PATHS contains the space-separated paths of files that were modified.
# Implement your project's auto-formatting and linting here.
#
# Default: hk fix (https://github.com/jdx/hk)
#   Requires: hk installed, .hk.toml present at project root
#
# Other examples:
#   prettier --write $CLAUDE_FILE_PATHS
#   rubocop --autocorrect $CLAUDE_FILE_PATHS
#   ruff check --fix $CLAUDE_FILE_PATHS

HK_PKL_BACKEND=pklr hk fix $CLAUDE_FILE_PATHS
```

```bash
chmod +x bin/lint
```

**7. Check for bin/setup and bin/dev**

If `bin/setup` does not exist, print:
```
Warning: bin/setup is missing. bin/worktree add calls bin/setup --skip-server after
creating a new worktree. Create it to automate dependency installation and database setup.
```

If `bin/dev` does not exist, print:
```
Warning: bin/dev is missing. bin/worktree server calls bin/dev to start the development server.
Create it (e.g. exec rails server -b 0.0.0.0 -p $PORT) for worktree server commands to work.
```

**8. Print summary**

Print a checklist:

```
cpb project setup complete.

Created:
  bin/check-worktree        — blocks edits on main branch (edit to customize)
  bin/claude-code-web-setup — web session setup stub (add your dependency installs)
  bin/lint                  — PostToolUse formatter (configured for hk — edit for your stack)

Plugin hooks dispatch to these scripts when they exist. Edit each one to fit your project.

Required for worktree commands (create if missing):
  bin/setup                 — called by `bin/worktree add` to set up a new worktree
  bin/dev                   — called by `bin/worktree server` to start the dev server
```
