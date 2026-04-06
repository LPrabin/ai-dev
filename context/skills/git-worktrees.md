# Skill: Git Worktrees

Based on [superpowers/using-git-worktrees](https://github.com/obra/superpowers).

Isolated workspaces for parallel development without stashing or branch-switching.

## When to Use
- Spiking or prototyping without polluting the main branch
- Running a subagent on a separate task
- Testing a fix while keeping current work intact
- Parallel feature development

## Create a Worktree

```bash
# Pick a directory (check for existing .worktrees/ or worktrees/ first)
WORKTREE_DIR=".worktrees"
mkdir -p "$WORKTREE_DIR"

# Ensure it's gitignored
grep -qxF "$WORKTREE_DIR/" .gitignore 2>/dev/null || echo "$WORKTREE_DIR/" >> .gitignore

# Create worktree with new branch
PROJECT=$(basename "$(pwd)")
BRANCH="wt/${PROJECT}-$(date +%Y%m%d-%H%M)"
git worktree add "${WORKTREE_DIR}/${BRANCH##*/}" -b "$BRANCH"

# Set up the worktree (auto-detect package manager)
cd "${WORKTREE_DIR}/${BRANCH##*/}"
[[ -f package.json ]] && npm install
[[ -f pyproject.toml ]] && uv sync
[[ -f Cargo.toml ]] && cargo build
[[ -f go.mod ]] && go mod download
```

## Verify Clean Baseline

Always run tests in the new worktree before making changes:
```bash
uv run pytest -q  # or equivalent
```

## Finish and Clean Up

```bash
# From the main working directory
git worktree remove .worktrees/<name>
git branch -d wt/<branch-name>  # only if merged
```

## Rules
- Never create worktrees outside the project directory without asking.
- Always verify the worktree directory is gitignored.
- Run project setup and verify tests before starting work.
- Clean up worktrees when done — they hold refs.
