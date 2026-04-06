# ai-dev Overhaul Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Overhaul ai-dev to fix MCP auto-redistribute, add full mode switching with persona files, enhanced Obsidian logging (changelog, commits, decisions), multi-repo workspace support, and agent handoff.

**Architecture:** Mode switching via symlinked `~/.ai/context/` pointing to `~/.ai/modes/{work,self}/`. Obsidian integration enhanced with changelog.md, commits.md, and decisions/ ADR files. MCP `enable/disable` auto-redistributes. Multi-repo workspaces use `docs/workspace/` with symlinks to root.

**Tech Stack:** Bash scripts, jq for JSON, sed/awk for TOML, Obsidian markdown

---

## File Structure

### Files to Create
- `~/.ai/modes/work/identity.md` — Work persona
- `~/.ai/modes/work/tools.md` — glab/GitLab conventions
- `~/.ai/modes/work/rules/` — Work-specific rules (copy from existing)
- `~/.ai/modes/self/identity.md` — Self persona
- `~/.ai/modes/self/tools.md` — gh/GitHub conventions
- `~/.ai/modes/self/rules/` — Self-specific rules
- `~/.ai-dev/bin/ai-handoff` — Agent handoff script
- `~/.ai-dev/bin/obs-commit` — Log commits to Obsidian
- `~/.ai-dev/bin/obs-decision` — Create ADR-style decision records

### Files to Modify
- `~/.ai-dev/bin/ai-mcp` — Add auto-redistribute on enable/disable
- `~/.ai-dev/bin/ai-mode` — Full mode switching with symlink + rebuild
- `~/.ai-dev/bin/ai-init` — Add `--multi-repo` flag
- `~/.ai-dev/bin/obs-lib` — Add changelog/commits/decisions helpers
- `~/.ai-dev/bin/obs-write` — Add `--changelog` flag, update structure
- `~/.ai-dev/bin/build-context` — Read from active mode symlink

---

## Phase 1: Fix MCP Auto-Redistribute

### Task 1: Add auto-redistribute to ai-mcp enable/disable

**Files:**
- Modify: `~/.ai-dev/bin/ai-mcp:49-73`

- [ ] **Step 1: Read current enable function**

Current `cmd_enable` only updates registry, doesn't redistribute.

- [ ] **Step 2: Add redistribute call to cmd_enable**

```bash
cmd_enable() {
  local name="${1:?usage: ai-mcp enable <name>}"
  require_jq
  require_registry

  jq --arg n "$name" '
    if .servers[$n] then .servers[$n].enabled = true
    else error("server not found: " + $n)
    end
  ' "$REGISTRY" > "${REGISTRY}.tmp" && mv "${REGISTRY}.tmp" "$REGISTRY"
  echo "[ai-mcp] enabled: ${name}"
  
  # Auto-redistribute to all tools
  cmd_distribute all
}
```

- [ ] **Step 3: Add redistribute call to cmd_disable**

```bash
cmd_disable() {
  local name="${1:?usage: ai-mcp disable <name>}"
  require_jq
  require_registry

  jq --arg n "$name" '
    if .servers[$n] then .servers[$n].enabled = false
    else error("server not found: " + $n)
    end
  ' "$REGISTRY" > "${REGISTRY}.tmp" && mv "${REGISTRY}.tmp" "$REGISTRY"
  echo "[ai-mcp] disabled: ${name}"
  
  # Auto-redistribute to all tools
  cmd_distribute all
}
```

- [ ] **Step 4: Test enable auto-redistribute**

```bash
ai-mcp disable serena
ai-mcp status  # Verify serena removed from all tools
ai-mcp enable serena
ai-mcp status  # Verify serena added to all tools
```

- [ ] **Step 5: Commit**

```bash
git add bin/ai-mcp
git commit -m "fix(ai-mcp): auto-redistribute on enable/disable"
```

---

## Phase 2: Mode Switching Infrastructure

### Task 2: Create modes directory structure

**Files:**
- Create: `~/.ai/modes/work/identity.md`
- Create: `~/.ai/modes/work/tools.md`
- Create: `~/.ai/modes/self/identity.md`
- Create: `~/.ai/modes/self/tools.md`

- [ ] **Step 1: Create modes directories**

```bash
mkdir -p ~/.ai/modes/work/rules
mkdir -p ~/.ai/modes/self/rules
```

- [ ] **Step 2: Create work identity.md**

Write to `~/.ai/modes/work/identity.md`:

```markdown
# Identity: Work Mode

Senior engineer at company. Pragmatic, security-conscious, team-oriented.

## Communication
- Professional but direct
- Document decisions for team visibility
- Follow company conventions and processes

## Priorities
1. Security and compliance
2. Team alignment
3. Maintainability
4. Delivery

## Tools
- GitLab (`glab`) for issues, MRs, CI/CD
- Company-specific tooling as defined in project AGENTS.md
```

- [ ] **Step 3: Create work tools.md**

Write to `~/.ai/modes/work/tools.md`:

```markdown
# Tools: Work Mode

## Git Workflow
- Remote: GitLab
- CLI: `glab`
- Branch naming: `feature/TICKET-123-description`, `fix/TICKET-456-description`
- MR workflow: Create MR, assign reviewers, wait for approval

## Issue Management
```bash
# Create issue
glab issue create --title "Title" --description "Description" --label "bug"

# List issues
glab issue list --assignee @me

# Create MR
glab mr create --fill --assignee @me
```

## CI/CD
- Pipeline must pass before merge
- Check pipeline: `glab ci status`
```

- [ ] **Step 4: Create self identity.md**

Write to `~/.ai/modes/self/identity.md`:

```markdown
# Identity: Self Mode

Independent developer. Curious, experimental, learning-focused.

## Communication
- Casual and exploratory
- Document learnings for future self
- Experiment freely

## Priorities
1. Learning and growth
2. Clean, maintainable code
3. Fun and exploration
4. Shipping personal projects

## Tools
- GitHub (`gh`) for repos, issues, PRs
- Personal tooling preferences
```

- [ ] **Step 5: Create self tools.md**

Write to `~/.ai/modes/self/tools.md`:

```markdown
# Tools: Self Mode

## Git Workflow
- Remote: GitHub
- CLI: `gh`
- Branch naming: `feature/description`, `fix/description`
- PR workflow: Create PR, self-review, merge

## Issue Management
```bash
# Create issue
gh issue create --title "Title" --body "Description"

# List issues
gh issue list

# Create PR
gh pr create --fill
```

## Publishing
- Open source by default
- Document for others to use
```

- [ ] **Step 6: Copy existing rules to self mode**

```bash
cp -r ~/.ai/context/rules/* ~/.ai/modes/self/rules/
cp -r ~/.ai/context/rules/* ~/.ai/modes/work/rules/
```

- [ ] **Step 7: Verify structure**

```bash
ls -la ~/.ai/modes/work/
ls -la ~/.ai/modes/self/
```

Expected:
```
~/.ai/modes/work/
  identity.md
  tools.md
  rules/

~/.ai/modes/self/
  identity.md
  tools.md
  rules/
```

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(modes): create work and self mode directories"
```

### Task 3: Update ai-mode for full switching

**Files:**
- Modify: `~/.ai-dev/bin/ai-mode`

- [ ] **Step 1: Add symlink switching function**

Add after the `source obs-lib` line:

```bash
AI_HOME="${AI_HOME:-$HOME/.ai}"
CONTEXT_DIR="${AI_HOME}/context"
MODES_DIR="${AI_HOME}/modes"

switch_mode() {
  local mode="$1"
  local mode_dir="${MODES_DIR}/${mode}"
  
  if [[ ! -d "$mode_dir" ]]; then
    echo "[ai-mode] error: mode directory not found: ${mode_dir}" >&2
    echo "Run: ai-mode --init-modes" >&2
    return 1
  fi
  
  # Remove existing symlink or directory
  if [[ -L "$CONTEXT_DIR" ]]; then
    rm "$CONTEXT_DIR"
  elif [[ -d "$CONTEXT_DIR" ]]; then
    echo "[ai-mode] warning: ${CONTEXT_DIR} is a directory, backing up to ${CONTEXT_DIR}.bak"
    mv "$CONTEXT_DIR" "${CONTEXT_DIR}.bak"
  fi
  
  # Create symlink
  ln -s "$mode_dir" "$CONTEXT_DIR"
  
  # Update shell env (for current session)
  export AI_MODE="$mode"
  
  # Rebuild context for all tools
  local build_script="${AI_HOME}/bin/build-context"
  if [[ -x "$build_script" ]]; then
    "$build_script" 2>/dev/null || true
  fi
  
  # Redistribute MCP servers
  local mcp_script
  mcp_script="$(dirname "${BASH_SOURCE[0]}")/ai-mcp"
  if [[ -x "$mcp_script" ]]; then
    "$mcp_script" distribute all 2>/dev/null || true
  fi
  
  # Log to Obsidian
  if obs_vault_set || obs_cli_available; then
    local log_entry="- [$(date '+%Y-%m-%d %H:%M')] Mode switch: ${mode}"
    obs_write_note "$(obs_session_note)" "$log_entry" 2>/dev/null || true
  fi
  
  echo "[ai-mode] Switched to: ${mode}"
  echo "  Context: ${CONTEXT_DIR} -> ${mode_dir}"
}
```

- [ ] **Step 2: Update case statement**

Replace the existing case statement:

```bash
case "${1:-}" in
  work)
    switch_mode "work"
    ;;
  self)
    switch_mode "self"
    ;;
  --init-modes)
    echo "Creating mode directories..."
    mkdir -p "${MODES_DIR}/work/rules"
    mkdir -p "${MODES_DIR}/self/rules"
    echo "Created: ${MODES_DIR}/work/"
    echo "Created: ${MODES_DIR}/self/"
    echo ""
    echo "Next: Add identity.md and tools.md to each mode directory"
    ;;
  -i|--init)
    init_vault
    ;;
  -l|--list)
    list_all
    ;;
  "")
    show_status
    ;;
  -h|--help)
    echo "Usage: ai-mode [work|self|--init|--init-modes|--list]"
    echo "  (no args)    show current status"
    echo "  work         switch to work mode"
    echo "  self         switch to self mode"
    echo "  --init       create vault directories"
    echo "  --init-modes create mode directories"
    echo "  --list       list all projects and topics"
    ;;
  *)
    echo "Unknown mode: $1 (use 'work' or 'self')" >&2
    exit 1
    ;;
esac
```

- [ ] **Step 3: Update show_status to show symlink**

Update `show_status()`:

```bash
show_status() {
  echo "ai-dev status"
  echo "─────────────────────────────────────"
  echo "  Mode:      ${OBS_MODE}"
  echo "  Project:   ${OBS_PROJECT}"
  echo "  Vault:     ${OBS_VAULT:-<not set>}"
  
  # Show context symlink status
  local ctx_dir="${AI_HOME:-$HOME/.ai}/context"
  if [[ -L "$ctx_dir" ]]; then
    local target
    target=$(readlink "$ctx_dir")
    echo "  Context:   ${ctx_dir} -> ${target}"
  elif [[ -d "$ctx_dir" ]]; then
    echo "  Context:   ${ctx_dir} (directory, not symlinked)"
  else
    echo "  Context:   ${ctx_dir} (not found)"
  fi
  echo ""
  
  # ... rest of function unchanged
```

- [ ] **Step 4: Test mode switching**

```bash
ai-mode self
ls -la ~/.ai/context  # Should be symlink to ~/.ai/modes/self
ai-mode work
ls -la ~/.ai/context  # Should be symlink to ~/.ai/modes/work
ai-mode               # Should show current status with symlink
```

- [ ] **Step 5: Commit**

```bash
git add bin/ai-mode
git commit -m "feat(ai-mode): full mode switching with symlinks and rebuild"
```

---

## Phase 3: Enhanced Obsidian Logging

### Task 4: Add changelog/commits/decisions helpers to obs-lib

**Files:**
- Modify: `~/.ai-dev/bin/obs-lib`

- [ ] **Step 1: Add new path helpers**

Add after existing path helpers (around line 79):

```bash
# ── new path helpers for enhanced logging ───────────────────────────────────

obs_changelog_note()  { echo "$(obs_base_path)/changelog.md"; }
obs_commits_note()    { echo "$(obs_base_path)/commits.md"; }
obs_context_file()    { echo "$(obs_base_path)/context.md"; }
obs_progress_note()   { echo "$(obs_base_path)/progress.md"; }

obs_next_decision_number() {
  local decisions_dir
  decisions_dir=$(obs_decisions_dir)
  if obs_vault_set && [[ -d "${OBS_VAULT}/${decisions_dir}" ]]; then
    local max=0
    for f in "${OBS_VAULT}/${decisions_dir}/"*.md; do
      [[ -f "$f" ]] || continue
      local num
      num=$(basename "$f" | grep -oE '^[0-9]+' || echo 0)
      [[ "$num" -gt "$max" ]] && max="$num"
    done
    printf "%03d" $((max + 1))
  else
    echo "001"
  fi
}
```

- [ ] **Step 2: Add changelog write helper**

```bash
obs_log_changelog() {
  local session_title="$1"
  local accomplished="$2"
  local decisions="$3"
  local open_items="$4"
  local agent="${5:-unknown}"
  local duration="${6:-}"
  
  local entry="## $(date '+%Y-%m-%d %H:%M') — Session: ${session_title}
Agent: ${agent} | Mode: ${OBS_MODE}${duration:+ | Duration: ${duration}}

### Accomplished
${accomplished}

### Decisions made
${decisions}

### Open items
${open_items}

---
"
  obs_write_note "$(obs_changelog_note)" "$entry"
}
```

- [ ] **Step 3: Add commit log helper**

```bash
obs_log_commit() {
  local hash="$1"
  local message="$2"
  local files_changed="$3"
  local time
  time=$(date '+%H:%M')
  local date
  date=$(date '+%Y-%m-%d')
  
  # Check if we need to add date header
  local changelog
  changelog=$(obs_read_note "$(obs_commits_note)")
  if ! echo "$changelog" | grep -q "^## ${date}$"; then
    obs_write_note "$(obs_commits_note)" "
## ${date}

| Time | Hash | Message | Files |
|------|------|---------|-------|"
  fi
  
  local entry="| ${time} | \`${hash:0:7}\` | ${message} | ${files_changed} |"
  obs_write_note "$(obs_commits_note)" "$entry"
}
```

- [ ] **Step 4: Add decision create helper**

```bash
obs_create_decision() {
  local title="$1"
  local context="$2"
  local decision="$3"
  local alternatives="$4"
  local consequences="$5"
  local agent="${6:-unknown}"
  
  local num
  num=$(obs_next_decision_number)
  local slug
  slug=$(echo "$title" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g')
  local filename="${num}-${slug}.md"
  local filepath="$(obs_decisions_dir)/${filename}"
  
  local content="# ${num}: ${title}

**Date:** $(date '+%Y-%m-%d')
**Status:** Accepted
**Agent:** ${agent}

## Context
${context}

## Decision
${decision}

## Alternatives Rejected
${alternatives}

## Consequences
${consequences}
"
  obs_overwrite_note "$filepath" "$content"
  echo "$filepath"
}
```

- [ ] **Step 5: Commit**

```bash
git add bin/obs-lib
git commit -m "feat(obs-lib): add changelog, commits, decisions helpers"
```

### Task 5: Create obs-commit script

**Files:**
- Create: `~/.ai-dev/bin/obs-commit`

- [ ] **Step 1: Create the script**

Write to `~/.ai-dev/bin/obs-commit`:

```bash
#!/usr/bin/env bash
# obs-commit — log git commits to Obsidian
#
# Usage:
#   obs-commit              # log most recent commit
#   obs-commit abc123       # log specific commit
#   obs-commit --install    # install git post-commit hook
#
# Called automatically via git post-commit hook

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=obs-lib
source "${SCRIPT_DIR}/obs-lib"

install_hook() {
  local git_dir
  git_dir=$(git rev-parse --git-dir 2>/dev/null) || {
    echo "[obs-commit] error: not in a git repository" >&2
    return 1
  }
  
  local hook_file="${git_dir}/hooks/post-commit"
  
  if [[ -f "$hook_file" ]]; then
    if grep -q "obs-commit" "$hook_file"; then
      echo "[obs-commit] hook already installed"
      return 0
    fi
    # Append to existing hook
    echo "" >> "$hook_file"
    echo "# Log commit to Obsidian" >> "$hook_file"
    echo 'obs-commit 2>/dev/null || true' >> "$hook_file"
  else
    cat > "$hook_file" <<'EOF'
#!/usr/bin/env bash
# Log commit to Obsidian
obs-commit 2>/dev/null || true
EOF
  fi
  
  chmod +x "$hook_file"
  echo "[obs-commit] installed hook: ${hook_file}"
}

log_commit() {
  local commit_hash="${1:-HEAD}"
  
  # Get commit info
  local hash message files_changed
  hash=$(git rev-parse --short "$commit_hash" 2>/dev/null) || return 1
  message=$(git log -1 --format='%s' "$commit_hash" 2>/dev/null) || return 1
  files_changed=$(git diff-tree --no-commit-id --name-only -r "$commit_hash" 2>/dev/null | wc -l | tr -d ' ')
  
  # Log to Obsidian
  if obs_vault_set || obs_cli_available; then
    obs_log_commit "$hash" "$message" "$files_changed"
    echo "[obs-commit] logged: ${hash} — ${message}"
  fi
}

case "${1:-}" in
  --install|-i)
    install_hook
    ;;
  --help|-h)
    echo "Usage: obs-commit [commit-hash | --install]"
    echo "  (no args)   log most recent commit"
    echo "  --install   install git post-commit hook"
    ;;
  "")
    log_commit HEAD
    ;;
  *)
    log_commit "$1"
    ;;
esac
```

- [ ] **Step 2: Make executable**

```bash
chmod +x ~/.ai-dev/bin/obs-commit
```

- [ ] **Step 3: Test the script**

```bash
cd ~/.ai-dev
obs-commit  # Should log most recent commit
cat ~/.ai/context/../../../Documents/Brain/Self/Topics/ai-dev/commits.md 2>/dev/null || echo "Check Obsidian vault"
```

- [ ] **Step 4: Commit**

```bash
git add bin/obs-commit
git commit -m "feat(obs-commit): add script to log commits to Obsidian"
```

### Task 6: Create obs-decision script

**Files:**
- Create: `~/.ai-dev/bin/obs-decision`

- [ ] **Step 1: Create the script**

Write to `~/.ai-dev/bin/obs-decision`:

```bash
#!/usr/bin/env bash
# obs-decision — create ADR-style decision records in Obsidian
#
# Usage:
#   obs-decision "Title"              # interactive ADR creation
#   obs-decision --quick "Title"      # quick one-liner decision
#
# Creates numbered decision files: decisions/001-title.md

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=obs-lib
source "${SCRIPT_DIR}/obs-lib"

obs_require_vault

quick_decision() {
  local title="$1"
  local filepath
  filepath=$(obs_create_decision \
    "$title" \
    "Quick decision during development." \
    "$title" \
    "N/A" \
    "To be determined." \
    "${AI_AGENT:-cli}")
  echo "[obs-decision] created: ${filepath}"
}

interactive_decision() {
  local title="$1"
  
  echo "Creating decision record: ${title}"
  echo ""
  
  local tmpfile
  tmpfile=$(mktemp /tmp/obs-decision-XXXXXX.md)
  
  cat > "$tmpfile" <<EOF
# Decision: ${title}

## Context
<!-- What is the issue/question that requires a decision? -->


## Decision
<!-- What did you decide? -->


## Alternatives Rejected
<!-- What other options did you consider? Why were they rejected? -->
- 

## Consequences
<!-- What are the implications of this decision? -->
- 
EOF

  "${EDITOR:-vi}" "$tmpfile"
  
  # Parse the file
  local context decision alternatives consequences
  context=$(sed -n '/^## Context/,/^## Decision/p' "$tmpfile" | grep -v '^##' | grep -v '^<!--' | sed '/^$/d')
  decision=$(sed -n '/^## Decision/,/^## Alternatives/p' "$tmpfile" | grep -v '^##' | grep -v '^<!--' | sed '/^$/d')
  alternatives=$(sed -n '/^## Alternatives/,/^## Consequences/p' "$tmpfile" | grep -v '^##' | grep -v '^<!--' | sed '/^$/d')
  consequences=$(sed -n '/^## Consequences/,/^$/p' "$tmpfile" | grep -v '^##' | grep -v '^<!--' | sed '/^$/d')
  
  rm -f "$tmpfile"
  
  if [[ -z "$context" ]] && [[ -z "$decision" ]]; then
    echo "[obs-decision] cancelled (empty content)"
    return 1
  fi
  
  local filepath
  filepath=$(obs_create_decision \
    "$title" \
    "${context:-No context provided.}" \
    "${decision:-No decision documented.}" \
    "${alternatives:-None documented.}" \
    "${consequences:-None documented.}" \
    "${AI_AGENT:-cli}")
  
  echo "[obs-decision] created: ${filepath}"
  
  # Also log to changelog
  local log_entry="- [[${filepath}|${title}]]"
  obs_write_note "$(obs_changelog_note)" "Decision recorded: ${log_entry}"
}

case "${1:-}" in
  --quick|-q)
    shift
    [[ -z "${1:-}" ]] && { echo "Usage: obs-decision --quick \"Title\"" >&2; exit 1; }
    quick_decision "$1"
    ;;
  --help|-h)
    echo "Usage: obs-decision [--quick] \"Title\""
    echo "  (no flag)   interactive ADR creation"
    echo "  --quick     quick one-liner decision"
    ;;
  "")
    echo "Usage: obs-decision \"Title\"" >&2
    exit 1
    ;;
  *)
    interactive_decision "$1"
    ;;
esac
```

- [ ] **Step 2: Make executable**

```bash
chmod +x ~/.ai-dev/bin/obs-decision
```

- [ ] **Step 3: Test the script**

```bash
obs-decision --quick "Test decision"
# Verify file created in Obsidian
```

- [ ] **Step 4: Commit**

```bash
git add bin/obs-decision
git commit -m "feat(obs-decision): add ADR-style decision records"
```

### Task 7: Update obs-write with --changelog flag

**Files:**
- Modify: `~/.ai-dev/bin/obs-write`

- [ ] **Step 1: Add --changelog case**

Add to the case statement (around line 105):

```bash
  --changelog)
    shift
    local title="${1:-Session}"
    shift || true
    
    # Interactive changelog entry
    local tmpfile
    tmpfile=$(mktemp /tmp/obs-changelog-XXXXXX.md)
    
    cat > "$tmpfile" <<EOF
# Session: ${title}

## Accomplished
- 

## Decisions made
- 

## Open items
- 
EOF
    
    "${EDITOR:-vi}" "$tmpfile"
    
    local accomplished decisions open_items
    accomplished=$(sed -n '/^## Accomplished/,/^## Decisions/p' "$tmpfile" | grep -v '^##' | sed '/^$/d')
    decisions=$(sed -n '/^## Decisions/,/^## Open/p' "$tmpfile" | grep -v '^##' | sed '/^$/d')
    open_items=$(sed -n '/^## Open items/,/^$/p' "$tmpfile" | grep -v '^##' | sed '/^$/d')
    
    rm -f "$tmpfile"
    
    obs_log_changelog "$title" "$accomplished" "$decisions" "$open_items" "${AI_AGENT:-cli}" ""
    echo "Logged to: $(obs_changelog_note)"
    exit 0
    ;;
```

- [ ] **Step 2: Update help text**

```bash
  -h|--help)
    echo "Usage: obs-write [--init | --session TEXT | --decision TEXT | --fix TEXT | --skill TEXT | --link PATH | --changelog TITLE]"
    echo "  (no flag)     interactive session log"
    echo "  --init        create project/topic vault structure"
    echo "  --session     write to Session.md"
    echo "  --decision    log a decision (simple)"
    echo "  --fix         log a bug fix"
    echo "  --skill       log a learned skill/pattern"
    echo "  --link        cross-link to another note"
    echo "  --changelog   write structured changelog entry"
    exit 0
    ;;
```

- [ ] **Step 3: Test**

```bash
obs-write --changelog "Test session"
# Edit the template, save
# Verify changelog.md updated
```

- [ ] **Step 4: Commit**

```bash
git add bin/obs-write
git commit -m "feat(obs-write): add --changelog for structured session logging"
```

---

## Phase 4: Agent Handoff

### Task 8: Create ai-handoff script

**Files:**
- Create: `~/.ai-dev/bin/ai-handoff`

- [ ] **Step 1: Create the script**

Write to `~/.ai-dev/bin/ai-handoff`:

```bash
#!/usr/bin/env bash
# ai-handoff — prepare context for agent switch
#
# Usage:
#   ai-handoff              # log context, show summary
#   ai-handoff codex        # log context, show Codex pickup instructions
#   ai-handoff claude       # log context, show Claude pickup instructions
#   ai-handoff opencode     # log context, show OpenCode pickup instructions
#   ai-handoff --resume     # show context from last handoff
#
# Logs current state to Obsidian context.md for continuity

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=obs-lib
source "${SCRIPT_DIR}/obs-lib"

show_resume() {
  local context
  context=$(obs_read_note "$(obs_context_file)")
  if [[ -n "$context" ]]; then
    echo "=== Last Handoff Context ==="
    echo ""
    echo "$context"
  else
    echo "(No handoff context found)"
  fi
}

write_handoff_context() {
  local from_agent="${AI_AGENT:-unknown}"
  local project="${OBS_PROJECT}"
  
  # Get recent session info
  local session_tasks
  session_tasks=$(obs_read_note "$(obs_session_note)" | grep -E "^\s*- \[" | head -10)
  
  # Get recent decisions
  local recent_decisions
  recent_decisions=$(obs_read_note "$(obs_decisions_dir)/log.md" | tail -5)
  
  # Get git status summary
  local git_status=""
  if git rev-parse --git-dir &>/dev/null; then
    local branch
    branch=$(git branch --show-current 2>/dev/null || echo "unknown")
    local dirty=""
    git diff --quiet 2>/dev/null || dirty=" (uncommitted changes)"
    git_status="Branch: ${branch}${dirty}"
  fi
  
  local content="# Handoff Context

**From:** ${from_agent}
**To:** (next agent)
**Project:** ${project}
**Mode:** ${OBS_MODE}
**Time:** $(date '+%Y-%m-%d %H:%M')

## Git Status
${git_status:-Not in a git repo}

## Current Tasks
${session_tasks:-(no tasks)}

## Recent Decisions
${recent_decisions:-(none)}

## What Was Being Worked On
<!-- Fill in before handing off -->


## What Needs to Happen Next
<!-- Fill in before handing off -->


## Blockers or Concerns
<!-- Fill in if any -->

"
  
  obs_overwrite_note "$(obs_context_file)" "$content"
  echo "[ai-handoff] Context saved to: $(obs_context_file)"
}

show_pickup_instructions() {
  local agent="$1"
  
  echo ""
  echo "=== Pickup Instructions for ${agent} ==="
  echo ""
  
  case "$agent" in
    codex)
      echo "In Codex, run:"
      echo '  obs-read --all'
      echo ""
      echo "Or start with:"
      echo '  "Read AGENTS.md and run obs-read to get context from the previous session."'
      ;;
    claude|claude-code)
      echo "In Claude Code, the CLAUDE.md file will be read automatically."
      echo ""
      echo "Start with:"
      echo '  "Run obs-read --all to get context from the previous session."'
      ;;
    opencode)
      echo "In OpenCode, run:"
      echo '  obs-read --all'
      echo ""
      echo "Or start with:"
      echo '  "Read AGENTS.md and run obs-read to get context from the previous session."'
      ;;
    *)
      echo "Run: obs-read --all"
      echo "This will show the current project context and session state."
      ;;
  esac
  
  echo ""
  echo "Context file: $(obs_context_file)"
}

log_handoff_to_changelog() {
  local to_agent="${1:-unknown}"
  local entry="- [$(date '+%Y-%m-%d %H:%M')] Handoff: ${AI_AGENT:-unknown} → ${to_agent}"
  obs_write_note "$(obs_changelog_note)" "$entry"
}

case "${1:-}" in
  --resume|-r)
    show_resume
    ;;
  --help|-h)
    echo "Usage: ai-handoff [agent | --resume]"
    echo "  (no args)   save context and show summary"
    echo "  codex       save context, show Codex pickup instructions"
    echo "  claude      save context, show Claude pickup instructions"
    echo "  opencode    save context, show OpenCode pickup instructions"
    echo "  --resume    show context from last handoff"
    ;;
  "")
    write_handoff_context
    echo ""
    echo "Edit the context file to add what you were working on and what's next."
    echo "Then tell the next agent to run: obs-read --all"
    ;;
  codex|claude|claude-code|opencode)
    write_handoff_context
    log_handoff_to_changelog "$1"
    show_pickup_instructions "$1"
    ;;
  *)
    echo "[ai-handoff] unknown agent: $1" >&2
    echo "Known agents: codex, claude, opencode" >&2
    exit 1
    ;;
esac
```

- [ ] **Step 2: Make executable**

```bash
chmod +x ~/.ai-dev/bin/ai-handoff
```

- [ ] **Step 3: Test**

```bash
ai-handoff         # Should save context
ai-handoff codex   # Should save context + show Codex instructions
ai-handoff --resume  # Should show saved context
```

- [ ] **Step 4: Commit**

```bash
git add bin/ai-handoff
git commit -m "feat(ai-handoff): add agent handoff with context logging"
```

---

## Phase 5: Multi-Repo Support

### Task 9: Add --multi-repo flag to ai-init

**Files:**
- Modify: `~/.ai-dev/bin/ai-init`

- [ ] **Step 1: Add multi-repo scaffold function**

Add after `scaffold_project()` function:

```bash
scaffold_multi_repo() {
  local project_name="$1"
  local project_dir="$PWD"
  local date_str
  date_str=$(date +%Y-%m-%d)
  
  echo ""
  echo "Scaffolding multi-repo workspace: ${project_name}"
  echo "Directory: ${project_dir}"
  echo "Mode: ${OBS_MODE}"
  echo ""
  
  # ── docs/ repo structure ────────────────────────────────────────────────────
  mkdir -p docs/workspace/{commands,agents}
  mkdir -p docs/progress
  mkdir -p docs/tests/e2e
  
  # Create AGENTS.md in docs/workspace
  cat > docs/workspace/AGENTS.md <<EOF
# ${project_name} — Multi-Repo Workspace

## Overview
This is a multi-repo workspace. The project root is NOT a git repo.

## Repositories
<!-- List your repos here -->
- \`repo-1/\` — Description
- \`repo-2/\` — Description
- \`docs/\` — Workspace config, E2E tests, progress tracking

## Workflow
1. Run Claude/Codex/OpenCode from project root
2. AGENTS.md (this file) provides workspace-wide context
3. Each repo has its own tests; E2E tests live in \`docs/tests/e2e/\`

## Progress
See \`docs/progress/progress.md\` for current phase status.

## Commands
Custom slash commands in \`.claude/commands/\` (symlinked from \`docs/workspace/commands/\`)

## Conventions
- Commit messages: conventional commits format
- Branch naming: \`feature/\`, \`fix/\`, \`chore/\`

## Obsidian
\`$(obs_base_path)\` — run \`ai-start\` at session start.
EOF
  success "created docs/workspace/AGENTS.md"
  
  # Create progress.md
  cat > docs/progress/progress.md <<EOF
# Progress: ${project_name}

## Phase 1: Setup
- [ ] Initialize repositories
- [ ] Set up CI/CD
- [ ] Configure workspace

## Phase 2: Core Features
- [ ] Feature 1
- [ ] Feature 2

## Phase 3: Testing
- [ ] Unit tests
- [ ] Integration tests
- [ ] E2E tests
EOF
  success "created docs/progress/progress.md"
  
  # ── Symlinks at root ────────────────────────────────────────────────────────
  if [[ ! -L "AGENTS.md" ]] && [[ ! -f "AGENTS.md" ]]; then
    ln -s docs/workspace/AGENTS.md AGENTS.md
    success "created AGENTS.md -> docs/workspace/AGENTS.md"
  else
    skip "AGENTS.md already exists"
  fi
  
  # Claude Code symlinks
  if command -v claude &>/dev/null; then
    mkdir -p .claude
    if [[ ! -L ".claude/commands" ]] && [[ ! -d ".claude/commands" ]]; then
      ln -s ../docs/workspace/commands .claude/commands
      success "created .claude/commands -> docs/workspace/commands"
    else
      skip ".claude/commands already exists"
    fi
    
    if [[ ! -L "CLAUDE.md" ]] && [[ ! -f "CLAUDE.md" ]]; then
      ln -s AGENTS.md CLAUDE.md
      success "created CLAUDE.md -> AGENTS.md"
    fi
  fi
  
  # ── Taskboard (work mode only) ──────────────────────────────────────────────
  if [[ "$OBS_MODE" == "work" ]]; then
    if [[ ! -d "taskboard" ]]; then
      mkdir -p taskboard/.gitlab/issue_templates
      cat > taskboard/README.md <<EOF
# Taskboard

This repo holds GitLab issues and milestones for ${project_name}.
No code here — just issue tracking.

## Usage
\`\`\`bash
# Create milestone
glab milestone create --title "Phase 1" --repo taskboard

# Create issue
glab issue create --title "Task" --milestone "Phase 1" --repo taskboard

# List issues
glab issue list --milestone "Phase 1" --repo taskboard
\`\`\`
EOF
      success "created taskboard/ scaffold"
    else
      skip "taskboard/ already exists"
    fi
  fi
  
  # ── Obsidian ────────────────────────────────────────────────────────────────
  if obs_cli_available || obs_vault_set; then
    info "creating Obsidian vault structure..."
    AI_PROJECT="$project_name" obs_ensure_dirs
    AI_PROJECT="$project_name" "${SCRIPT_DIR}/obs-write" --init 2>/dev/null \
      && success "Obsidian structure created at $(obs_base_path)" \
      || skip "Obsidian not available"
  fi
  
  # ── Summary ─────────────────────────────────────────────────────────────────
  echo ""
  echo "Multi-repo workspace created: ${project_name}"
  echo ""
  echo "  Structure:"
  echo "    AGENTS.md              -> docs/workspace/AGENTS.md"
  echo "    .claude/commands/      -> docs/workspace/commands/"
  echo "    docs/workspace/        — workspace configuration"
  echo "    docs/progress/         — progress tracking"
  echo "    docs/tests/e2e/        — E2E tests"
  [[ "$OBS_MODE" == "work" ]] && echo "    taskboard/             — GitLab issues (work mode)"
  echo ""
  echo "  Next steps:"
  echo "    1. Create/clone your repos in this directory"
  echo "    2. Edit docs/workspace/AGENTS.md with repo descriptions"
  echo "    3. ai-start to begin a session"
}
```

- [ ] **Step 2: Update entry point case statement**

Replace the case statement at the end:

```bash
case "${1:-}" in
  --global-install|-g)
    global_install
    ;;
  --multi-repo|-m)
    shift
    scaffold_multi_repo "${1:-$(basename "$PWD")}"
    ;;
  "")
    scaffold_project "$(basename "$PWD")"
    ;;
  --*)
    echo "Unknown flag: $1" >&2
    echo "Usage: ai-init [project-name] | --multi-repo [name] | --global-install" >&2
    exit 1
    ;;
  *)
    scaffold_project "$1"
    ;;
esac
```

- [ ] **Step 3: Test multi-repo scaffold**

```bash
mkdir -p /tmp/test-multi-repo
cd /tmp/test-multi-repo
ai-init --multi-repo test-project
ls -la
ls -la docs/workspace/
cat AGENTS.md
```

- [ ] **Step 4: Cleanup test**

```bash
rm -rf /tmp/test-multi-repo
```

- [ ] **Step 5: Commit**

```bash
git add bin/ai-init
git commit -m "feat(ai-init): add --multi-repo workspace scaffolding"
```

---

## Phase 6: Update build-context for Mode Symlinks

### Task 10: Update build-context to read from symlinked context

**Files:**
- Modify: `~/.ai-dev/bin/build-context`

- [ ] **Step 1: Read current build-context**

Review the file to understand current behavior.

- [ ] **Step 2: Add symlink resolution**

Ensure the script resolves symlinks when reading from `~/.ai/context/`:

```bash
# At the top of the script, after AI_HOME definition
CONTEXT_DIR="${AI_HOME}/context"

# Resolve symlink if present
if [[ -L "$CONTEXT_DIR" ]]; then
  CONTEXT_DIR=$(readlink -f "$CONTEXT_DIR")
  echo "[build-context] Using mode context: ${CONTEXT_DIR}"
fi
```

- [ ] **Step 3: Test with mode switch**

```bash
ai-mode self
build-context
# Verify it reads from ~/.ai/modes/self/

ai-mode work
build-context
# Verify it reads from ~/.ai/modes/work/
```

- [ ] **Step 4: Commit**

```bash
git add bin/build-context
git commit -m "feat(build-context): resolve mode symlinks"
```

---

## Phase 7: Documentation and Final Testing

### Task 11: Update README with new features

**Files:**
- Modify: `~/.ai-dev/README.md`

- [ ] **Step 1: Add mode switching section**

- [ ] **Step 2: Add Obsidian logging section**

- [ ] **Step 3: Add multi-repo section**

- [ ] **Step 4: Add agent handoff section**

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: update README with new features"
```

### Task 12: End-to-end testing

- [ ] **Step 1: Test MCP auto-redistribute**

```bash
ai-mcp disable playwright
ai-mcp status  # Verify removed from all tools
ai-mcp enable playwright
ai-mcp status  # Verify added to all tools
```

- [ ] **Step 2: Test mode switching**

```bash
ai-mode self
ai-mode  # Verify status shows symlink
ai-mode work
ai-mode  # Verify status shows symlink
```

- [ ] **Step 3: Test Obsidian logging**

```bash
obs-write --changelog "Test session"
obs-decision --quick "Test decision"
obs-commit
```

- [ ] **Step 4: Test agent handoff**

```bash
ai-handoff codex
ai-handoff --resume
```

- [ ] **Step 5: Test multi-repo init**

```bash
mkdir -p /tmp/test-workspace
cd /tmp/test-workspace
ai-init --multi-repo test-project
ls -la
rm -rf /tmp/test-workspace
```

- [ ] **Step 6: Final commit**

```bash
git add -A
git commit -m "chore: final testing and cleanup"
```

---

## Summary

| Phase | Tasks | Description |
|-------|-------|-------------|
| 1 | 1 | Fix MCP auto-redistribute |
| 2 | 2-3 | Mode switching infrastructure |
| 3 | 4-7 | Enhanced Obsidian logging |
| 4 | 8 | Agent handoff |
| 5 | 9 | Multi-repo support |
| 6 | 10 | Build-context mode symlinks |
| 7 | 11-12 | Documentation and testing |

**Total: 12 tasks, ~45 steps**

**Estimated time:** 2-3 hours for implementation + testing
