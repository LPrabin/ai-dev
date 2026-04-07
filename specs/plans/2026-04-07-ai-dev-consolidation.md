# ai-dev Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate `~/.ai-dev` (source) and `~/.ai` (runtime) into a git-driven deploy workflow with rsync.

**Architecture:** Source lives in `~/.ai-dev` with `modes/` and `shared/` directories. Running `ai-deploy` rsyncs to `~/.ai` and rebuilds prompts. Git post-commit hook automates deployment.

**Tech Stack:** Bash, rsync, git hooks

**Spec:** `~/.ai-dev/specs/2026-04-07-ai-dev-consolidation-design.md`

---

## Task 1: Backup Current State

**Files:**
- None created/modified (safety checkpoint)

- [ ] **Step 1: Backup ~/.ai**

```bash
cp -r ~/.ai ~/.ai.bak.$(date +%s)
```

Expected: Directory copied with timestamp suffix

- [ ] **Step 2: Backup ~/.ai-dev**

```bash
cp -r ~/.ai-dev ~/.ai-dev.bak.$(date +%s)
```

Expected: Directory copied with timestamp suffix

- [ ] **Step 3: Verify backups exist**

```bash
ls -d ~/.ai.bak.* ~/.ai-dev.bak.*
```

Expected: Two backup directories listed

---

## Task 2: Create New Directory Structure in Source

**Files:**
- Create: `~/.ai-dev/modes/work/`
- Create: `~/.ai-dev/modes/self/`
- Create: `~/.ai-dev/shared/`
- Create: `~/.ai-dev/.githooks/`

- [ ] **Step 1: Create modes directories**

```bash
mkdir -p ~/.ai-dev/modes/work/rules
mkdir -p ~/.ai-dev/modes/self/rules
```

Expected: Directories created

- [ ] **Step 2: Create shared directory**

```bash
mkdir -p ~/.ai-dev/shared/agents
mkdir -p ~/.ai-dev/shared/skills
```

Expected: Directories created

- [ ] **Step 3: Create githooks directory**

```bash
mkdir -p ~/.ai-dev/.githooks
```

Expected: Directory created

- [ ] **Step 4: Verify structure**

```bash
ls -la ~/.ai-dev/modes ~/.ai-dev/shared ~/.ai-dev/.githooks
```

Expected: All directories exist with correct subdirectories

---

## Task 3: Migrate Context to Modes and Shared

**Files:**
- Move: `~/.ai-dev/context/identity.md` -> `~/.ai-dev/modes/work/identity.md`
- Move: `~/.ai-dev/context/rules/` -> `~/.ai-dev/modes/work/rules/`
- Move: `~/.ai-dev/context/agents/` -> `~/.ai-dev/shared/agents/`
- Move: `~/.ai-dev/context/skills/` -> `~/.ai-dev/shared/skills/`
- Move: `~/.ai-dev/context/mcp-servers.json` -> `~/.ai-dev/shared/mcp-servers.json`
- Copy: `~/.ai/modes/self/` -> `~/.ai-dev/modes/self/`

- [ ] **Step 1: Move identity.md to modes/work**

```bash
cp ~/.ai-dev/context/identity.md ~/.ai-dev/modes/work/identity.md
```

Expected: File copied

- [ ] **Step 2: Move rules to modes/work**

```bash
cp -r ~/.ai-dev/context/rules/* ~/.ai-dev/modes/work/rules/
```

Expected: Rules copied

- [ ] **Step 3: Copy tools.md from runtime to modes/work**

The runtime has `tools.md` that source doesn't. Copy it:

```bash
cp ~/.ai/modes/work/tools.md ~/.ai-dev/modes/work/tools.md
```

Expected: tools.md copied

- [ ] **Step 4: Move agents to shared**

```bash
cp -r ~/.ai-dev/context/agents/* ~/.ai-dev/shared/agents/
```

Expected: Agents copied

- [ ] **Step 5: Move skills to shared**

```bash
cp -r ~/.ai-dev/context/skills/* ~/.ai-dev/shared/skills/
```

Expected: Skills copied

- [ ] **Step 6: Move mcp-servers.json to shared**

```bash
cp ~/.ai-dev/context/mcp-servers.json ~/.ai-dev/shared/mcp-servers.json
```

Expected: mcp-servers.json copied

- [ ] **Step 7: Copy self mode from runtime**

```bash
cp -r ~/.ai/modes/self/* ~/.ai-dev/modes/self/
```

Expected: Self mode copied (identity.md, rules/, tools.md)

- [ ] **Step 8: Verify migration**

```bash
ls -la ~/.ai-dev/modes/work/
ls -la ~/.ai-dev/modes/self/
ls -la ~/.ai-dev/shared/
```

Expected: 
- modes/work: identity.md, rules/, tools.md
- modes/self: identity.md, rules/, tools.md
- shared: agents/, skills/, mcp-servers.json

- [ ] **Step 9: Remove old context directory**

Only after verifying migration:

```bash
rm -rf ~/.ai-dev/context
```

Expected: Old context directory removed

---

## Task 4: Create ai-deploy Script

**Files:**
- Create: `~/.ai-dev/bin/ai-deploy`

- [ ] **Step 1: Create the ai-deploy script**

Create file `~/.ai-dev/bin/ai-deploy`:

```bash
#!/usr/bin/env bash
# ai-deploy — deploy ~/.ai-dev to ~/.ai runtime
#
# Usage:
#   ai-deploy                deploy to ~/.ai
#   ai-deploy --dry-run      show what would be synced
#   ai-deploy --quiet        suppress output (for git hook)
#   ai-deploy --install-hook configure git to use .githooks/

set -euo pipefail

AI_DEV="${AI_DEV:-$HOME/.ai-dev}"
AI_HOME="${AI_HOME:-$HOME/.ai}"

DRY_RUN=false
QUIET=false
INSTALL_HOOK=false

while [[ $# -gt 0 ]]; do
  case "$1" in
    --dry-run)      DRY_RUN=true ;;
    --quiet|-q)     QUIET=true ;;
    --install-hook) INSTALL_HOOK=true ;;
    -h|--help)
      echo "Usage: ai-deploy [--dry-run] [--quiet] [--install-hook]"
      exit 0
      ;;
    *) echo "Unknown arg: $1" >&2; exit 1 ;;
  esac
  shift
done

log() {
  if [[ "$QUIET" != true ]]; then
    echo "$@"
  fi
}

# Install git hook
if [[ "$INSTALL_HOOK" == true ]]; then
  git -C "$AI_DEV" config core.hooksPath .githooks
  log "[ai-deploy] Git hook installed: .githooks/"
  exit 0
fi

# Verify source exists
if [[ ! -d "$AI_DEV" ]]; then
  echo "[ai-deploy] error: source not found: $AI_DEV" >&2
  exit 1
fi

# Check for required directories
for dir in modes shared bin template; do
  if [[ ! -d "$AI_DEV/$dir" ]]; then
    echo "[ai-deploy] error: missing directory: $AI_DEV/$dir" >&2
    exit 1
  fi
done

# Rsync options
RSYNC_OPTS="-a --delete"
RSYNC_OPTS+=" --exclude=.git"
RSYNC_OPTS+=" --exclude=.DS_Store"
RSYNC_OPTS+=" --exclude=.githooks"
RSYNC_OPTS+=" --exclude=prompts"
RSYNC_OPTS+=" --exclude=.deployed"
RSYNC_OPTS+=" --exclude=.env"
RSYNC_OPTS+=" --exclude=specs"
RSYNC_OPTS+=" --exclude=README.md"
RSYNC_OPTS+=" --exclude=DEVELOPMENT.md"
RSYNC_OPTS+=" --exclude=USAGE.md"
RSYNC_OPTS+=" --exclude=.claude"
RSYNC_OPTS+=" --exclude=*.pdf"

if [[ "$DRY_RUN" == true ]]; then
  RSYNC_OPTS+=" --dry-run"
  log "[ai-deploy] DRY RUN — no changes will be made"
fi

log "[ai-deploy] Deploying $AI_DEV -> $AI_HOME"

# Ensure target exists
mkdir -p "$AI_HOME"

# Sync each directory
for dir in bin modes shared template; do
  if [[ -d "$AI_DEV/$dir" ]]; then
    # shellcheck disable=SC2086
    rsync $RSYNC_OPTS "$AI_DEV/$dir/" "$AI_HOME/$dir/"
    count=$(find "$AI_DEV/$dir" -type f | wc -l | tr -d ' ')
    log "  $dir/    $count files"
  fi
done

if [[ "$DRY_RUN" == true ]]; then
  log "[ai-deploy] Would run: build-context --adapter all"
  exit 0
fi

# Create context symlink if missing
if [[ ! -L "$AI_HOME/context" ]]; then
  if [[ -d "$AI_HOME/context" ]]; then
    log "[ai-deploy] Backing up existing context directory"
    mv "$AI_HOME/context" "$AI_HOME/context.bak.$(date +%s)"
  fi
  ln -s "$AI_HOME/modes/work" "$AI_HOME/context"
  log "[ai-deploy] Created: context -> modes/work"
fi

# Run build-context
BUILD_CONTEXT="$AI_HOME/bin/build-context"
if [[ -x "$BUILD_CONTEXT" ]]; then
  log "[ai-deploy] Running build-context..."
  "$BUILD_CONTEXT" --adapter all 2>/dev/null || true
fi

# Write deployment metadata
COMMIT_HASH=$(git -C "$AI_DEV" rev-parse --short HEAD 2>/dev/null || echo "unknown")
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
echo "commit: $COMMIT_HASH" > "$AI_HOME/.deployed"
echo "timestamp: $TIMESTAMP" >> "$AI_HOME/.deployed"
echo "source: $AI_DEV" >> "$AI_HOME/.deployed"

log "[ai-deploy] Deployed: $COMMIT_HASH ($TIMESTAMP)"
```

- [ ] **Step 2: Make executable**

```bash
chmod +x ~/.ai-dev/bin/ai-deploy
```

Expected: Script is executable

- [ ] **Step 3: Verify script syntax**

```bash
bash -n ~/.ai-dev/bin/ai-deploy
```

Expected: No syntax errors

---

## Task 5: Create Git Hook

**Files:**
- Create: `~/.ai-dev/.githooks/post-commit`

- [ ] **Step 1: Create post-commit hook**

Create file `~/.ai-dev/.githooks/post-commit`:

```bash
#!/usr/bin/env bash
# Auto-deploy after commit

AI_DEV="${AI_DEV:-$HOME/.ai-dev}"
DEPLOY_SCRIPT="$AI_DEV/bin/ai-deploy"

if [[ -x "$DEPLOY_SCRIPT" ]]; then
  "$DEPLOY_SCRIPT" --quiet || echo "[post-commit] Warning: ai-deploy failed" >&2
fi
```

- [ ] **Step 2: Make executable**

```bash
chmod +x ~/.ai-dev/.githooks/post-commit
```

Expected: Hook is executable

---

## Task 6: Update build-context Script

**Files:**
- Modify: `~/.ai-dev/bin/build-context`

- [ ] **Step 1: Update get_skills_dir function**

Change line 62 from:
```bash
  echo "${AI_HOME}/context/skills"
```

To:
```bash
  echo "${AI_HOME}/shared/skills"
```

- [ ] **Step 2: Update get_agents_dir function**

Change line 74 from:
```bash
  echo "${AI_HOME}/context/agents"
```

To:
```bash
  echo "${AI_HOME}/shared/agents"
```

- [ ] **Step 3: Update MCP registry path**

Change line 193 from:
```bash
  local mcp_registry="${AI_HOME}/context/mcp-servers.json"
```

To:
```bash
  local mcp_registry="${AI_HOME}/shared/mcp-servers.json"
```

- [ ] **Step 4: Update skills echo message**

Change line 172 from:
```bash
  echo "Detailed skill files in Obsidian Library/Skills/. Reference them when relevant:"
```

To:
```bash
  echo "Detailed skill files in ~/.ai/shared/skills/. Reference them when relevant:"
```

- [ ] **Step 5: Update MCP registry echo**

Change line 203 from:
```bash
      echo "Registry: \`~/.ai/context/mcp-servers.json\`. Use \`ai-mcp list\` to see all."
```

To:
```bash
      echo "Registry: \`~/.ai/shared/mcp-servers.json\`. Use \`ai-mcp list\` to see all."
```

- [ ] **Step 6: Verify build-context syntax**

```bash
bash -n ~/.ai-dev/bin/build-context
```

Expected: No syntax errors

---

## Task 7: Update ai-mode Script

**Files:**
- Modify: `~/.ai-dev/bin/ai-mode`

- [ ] **Step 1: Remove build-context call from switch_mode**

In function `switch_mode()`, remove lines 54-58:

```bash
  # Rebuild context for all tools
  local build_script="${AI_HOME}/bin/build-context"
  if [[ -x "$build_script" ]]; then
    "$build_script" 2>/dev/null || true
  fi
```

- [ ] **Step 2: Remove MCP redistribution from switch_mode**

Remove lines 60-65:

```bash
  # Redistribute MCP servers
  local mcp_script
  mcp_script="$(dirname "${BASH_SOURCE[0]}")/ai-mcp"
  if [[ -x "$mcp_script" ]]; then
    "$mcp_script" distribute all 2>/dev/null || true
  fi
```

- [ ] **Step 3: Add note about instant switching**

After the symlink creation (line 49), add:

```bash
  # Note: Mode switching is instant. build-context runs during deploy, not here.
```

- [ ] **Step 4: Verify ai-mode syntax**

```bash
bash -n ~/.ai-dev/bin/ai-mode
```

Expected: No syntax errors

---

## Task 8: Clean Deploy to Runtime

**Files:**
- Modifies: `~/.ai/` (runtime)

- [ ] **Step 1: Preserve .env**

```bash
[[ -f ~/.ai/.env ]] && cp ~/.ai/.env /tmp/.ai-env-backup
```

Expected: .env backed up if exists

- [ ] **Step 2: Remove old runtime content**

```bash
rm -rf ~/.ai/bin ~/.ai/modes ~/.ai/shared ~/.ai/context ~/.ai/prompts ~/.ai/template
```

Expected: Old directories removed

- [ ] **Step 3: Run initial deploy**

```bash
~/.ai-dev/bin/ai-deploy
```

Expected output:
```
[ai-deploy] Deploying /Users/.../.ai-dev -> /Users/.../.ai
  bin/         XX files
  modes/        X files
  shared/       X files
  template/     X files
[ai-deploy] Created: context -> modes/work
[ai-deploy] Running build-context...
[ai-deploy] Deployed: XXXXXXX (YYYY-MM-DD HH:MM:SS)
```

- [ ] **Step 4: Restore .env**

```bash
[[ -f /tmp/.ai-env-backup ]] && mv /tmp/.ai-env-backup ~/.ai/.env
```

Expected: .env restored if it existed

- [ ] **Step 5: Verify deployment**

```bash
ls -la ~/.ai/
cat ~/.ai/.deployed
```

Expected: Shows bin/, modes/, shared/, template/, context symlink, .deployed file

---

## Task 9: Install Git Hook

**Files:**
- Modifies: `~/.ai-dev/.git/config`

- [ ] **Step 1: Install hook**

```bash
~/.ai-dev/bin/ai-deploy --install-hook
```

Expected: `[ai-deploy] Git hook installed: .githooks/`

- [ ] **Step 2: Verify hook configuration**

```bash
git -C ~/.ai-dev config --get core.hooksPath
```

Expected: `.githooks`

---

## Task 10: Verify Full Cycle

**Files:**
- None (verification only)

- [ ] **Step 1: Test mode switching**

```bash
~/.ai/bin/ai-mode work
~/.ai/bin/ai-mode self
~/.ai/bin/ai-mode work
~/.ai/bin/ai-mode
```

Expected: Mode switches without errors, shows current mode

- [ ] **Step 2: Test manual deploy**

```bash
~/.ai-dev/bin/ai-deploy --dry-run
```

Expected: Shows what would be synced without making changes

- [ ] **Step 3: Test auto-deploy (git hook)**

```bash
cd ~/.ai-dev
echo "" >> README.md
git add README.md
git commit -m "test: verify auto-deploy hook"
cat ~/.ai/.deployed
```

Expected: .deployed shows new commit hash

- [ ] **Step 4: Test build-context output**

```bash
head -30 ~/.ai/prompts/global.md
```

Expected: Valid assembled prompt with identity, rules, skills

- [ ] **Step 5: Verify adapters ran**

```bash
head -10 ~/.claude/CLAUDE.md 2>/dev/null || echo "Claude adapter not installed"
```

Expected: Shows Claude-specific config or appropriate message

---

## Task 11: Commit All Changes

**Files:**
- All modified files in ~/.ai-dev

- [ ] **Step 1: Stage changes**

```bash
cd ~/.ai-dev
git add -A
git status
```

Expected: Shows new files (modes/, shared/, .githooks/, bin/ai-deploy) and modified files (bin/build-context, bin/ai-mode)

- [ ] **Step 2: Commit**

```bash
git commit -m "feat: consolidate .ai-dev and .ai with rsync deploy

- Add modes/ and shared/ directories in source
- Create ai-deploy script with rsync
- Add post-commit hook for auto-deploy
- Update build-context to use shared/ paths
- Simplify ai-mode (no longer runs build-context)

Closes: consolidation design spec"
```

Expected: Commit succeeds, auto-deploy runs via hook

- [ ] **Step 3: Verify final state**

```bash
~/.ai/bin/ai-mode
cat ~/.ai/.deployed
```

Expected: Shows correct mode and latest commit hash

---

## Rollback (if needed)

If anything fails catastrophically:

```bash
# Find and restore backups
ls -d ~/.ai.bak.* ~/.ai-dev.bak.*

# Restore (use actual timestamps from ls output)
rm -rf ~/.ai ~/.ai-dev
mv ~/.ai.bak.TIMESTAMP ~/.ai
mv ~/.ai-dev.bak.TIMESTAMP ~/.ai-dev
```
