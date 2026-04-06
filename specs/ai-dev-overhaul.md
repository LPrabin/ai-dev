# ai-dev Overhaul — Design Spec

**Status:** Approved  
**Date:** 2026-04-06  
**Author:** claude-code (with user)

---

## Overview

Overhaul the `ai-dev` system to fix MCP issues, add mode switching, enable agent hot-swapping, and make Obsidian the central memory for context continuity.

## Goals

1. **Fix MCP issues**: Codex uses TOML (not JSON), `ai-mcp disable` auto-redistributes
2. **Mode switching**: Work vs Self personas with full config swap
3. **Agent hot-swapping**: Switch between Claude Code, Codex, OpenCode without losing context
4. **Obsidian as center**: All significant events logged for continuity
5. **Multi-repo support**: Handle workspaces with multiple git repos

---

## Directory Structure

### Global (~/.ai/)

```
~/.ai/
├── modes/
│   ├── work/
│   │   ├── identity.md          # Work persona
│   │   ├── rules/               # Work-specific rules
│   │   └── tools.md             # glab, GitLab conventions
│   └── self/
│       ├── identity.md          # Personal persona
│       ├── rules/               # Personal rules
│       └── tools.md             # gh, GitHub conventions
├── context/                     # → symlink to active mode
├── mcp-servers.json             # Unified registry
└── bin/
    ├── ai-mode                  # Switch work/self
    ├── ai-init                  # Initialize project workspace
    ├── ai-mcp                   # MCP management (TOML fix)
    ├── ai-handoff               # Agent switch with context
    ├── build-context            # Assemble AGENTS.md
    ├── obs-read                 # Pull from Obsidian
    ├── obs-write                # Push to Obsidian
    ├── obs-commit               # Log commit to Obsidian
    └── adapters/
        ├── adapter-codex.sh     # TOML → ~/.codex/config.toml
        ├── adapter-claude-code.sh
        └── adapter-opencode.sh
```

### Obsidian Vault

```
Obsidian/
├── Work/
│   └── Projects/
│       └── {project}/
│           ├── changelog.md         # Session-by-session log
│           ├── commits.md           # Mirrored commit history
│           ├── progress.md          # Synced with project progress.md
│           ├── decisions/
│           │   ├── 001-tech-stack.md
│           │   ├── 002-auth-approach.md
│           │   └── ...
│           └── context.md           # Current state for handoffs
│
└── Self/
    └── Projects/
        └── {project}/
            ├── changelog.md
            ├── commits.md
            ├── decisions/
            └── context.md
```

### Project Workspace (Single Repo)

```
project/
├── AGENTS.md                    # Universal context
├── .claude/commands/            # Claude-specific
└── (source code)
```

### Project Workspace (Multi-Repo)

```
project-root/                    # NOT a git repo
├── AGENTS.md                    # → symlink to docs/workspace/AGENTS.md
├── .claude/
│   ├── commands/                # → symlink to docs/workspace/commands/
│   └── settings.json
├── {repo-1}/                    # git repo (user creates)
├── {repo-2}/                    # git repo (user creates)
├── taskboard/                   # git repo — GitLab issues (work mode only)
└── docs/                        # git repo — workspace config
    ├── workspace/
    │   ├── AGENTS.md
    │   ├── commands/
    │   └── agents/
    ├── progress/
    │   └── progress.md
    └── tests/e2e/
```

---

## Obsidian File Formats

### changelog.md

Auto-appended each session via `obs-write`:

```markdown
## 2026-04-06 14:30 — Session: ai-dev overhaul
Agent: claude-code | Mode: self | Duration: 45m

### Accomplished
- Designed multi-repo workspace structure
- Fixed MCP TOML issue for Codex

### Decisions made
- [[decisions/003-multi-repo-design|Multi-repo workspace design]]

### Open items
- Implement ai-init command
- Write tests for adapter scripts
```

### commits.md

Auto-appended via `obs-commit` (git post-commit hook):

```markdown
## 2026-04-06

| Time | Hash | Message | Files |
|------|------|---------|-------|
| 14:45 | `a1b2c3d` | feat(mcp): add TOML support for Codex | 3 |
| 15:20 | `e4f5g6h` | fix(adapter): correct config path | 1 |
```

### decisions/NNN-title.md

ADR format, created via `obs-decision`:

```markdown
# 001: Multi-repo Workspace Design

**Date:** 2026-04-06
**Status:** Accepted
**Agent:** claude-code

## Context
Need to support company's multi-repo project pattern.

## Decision
Workspace root is not a git repo. AGENTS.md symlinked from docs/workspace/.

## Alternatives Rejected
- Single monorepo: Too restrictive for existing projects
- Separate configs per repo: Duplication, drift risk

## Consequences
- Need ai-init --multi-repo command
- Symlink management required
```

### progress.md

Synced between project and Obsidian:

```markdown
# Progress: {project}

## Phase 1: Foundation
- [x] Project structure setup
- [x] Auth module
- [ ] API endpoints

## Phase 2: Features
- [ ] User dashboard
- [ ] Notifications
```

---

## Command Specifications

### ai-mode

Switch between work and self personas.

```bash
ai-mode work    # Switch to work mode
ai-mode self    # Switch to self mode
ai-mode         # Show current mode
```

**Behavior:**
1. Update symlink: `~/.ai/context/` → `~/.ai/modes/{mode}/`
2. Run `build-context` to regenerate AGENTS.md
3. Run `ai-mcp distribute` to update tool configs
4. Log mode change to Obsidian changelog

### ai-init

Initialize project workspace.

```bash
ai-init project-name              # Single repo
ai-init --multi-repo project-name # Multi-repo workspace
```

**Single repo behavior:**
1. Create `AGENTS.md` from template
2. Create `.claude/commands/` if Claude Code detected

**Multi-repo behavior:**
1. Create `docs/` repo structure:
   - `docs/workspace/AGENTS.md`
   - `docs/workspace/commands/`
   - `docs/workspace/agents/`
   - `docs/progress/progress.md`
2. Create symlinks at root:
   - `AGENTS.md` → `docs/workspace/AGENTS.md`
   - `.claude/commands/` → `docs/workspace/commands/`
3. If work mode: create `taskboard/` scaffold
4. Create Obsidian project folder with templates

### ai-mcp

MCP server management with auto-distribution.

```bash
ai-mcp list                  # List all servers and status
ai-mcp enable serena         # Enable server + redistribute
ai-mcp disable playwright    # Disable server + redistribute
ai-mcp distribute            # Manually redistribute to all tools
ai-mcp distribute codex      # Redistribute to specific tool
```

**Behavior:**
- Maintains unified registry at `~/.ai/context/mcp-servers.json`
- On enable/disable: update registry, then auto-redistribute
- Distribution calls adapters which output correct format per tool

### ai-handoff

Prepare context for agent switch.

```bash
ai-handoff              # Log context, show summary
ai-handoff codex        # Log context, show Codex-specific instructions
ai-handoff --resume     # Show context from last handoff
```

**Behavior:**
1. Log current context to Obsidian `context.md`
2. Display handoff summary (what was done, what's next)
3. If target agent specified, show agent-specific pickup instructions

### obs-read

Pull context from Obsidian.

```bash
obs-read                # Read context for current project
obs-read --all          # Read all related notes
obs-read --decisions    # Read recent decisions only
```

### obs-write

Push session summary to Obsidian.

```bash
obs-write                           # Interactive prompt for summary
obs-write "Summary of what was done" # Direct write
```

**Behavior:**
1. Append entry to `changelog.md`
2. If decisions made, prompt to create decision records
3. Sync `progress.md` if it exists in project

### obs-commit

Log commit to Obsidian (called from git hook).

```bash
obs-commit              # Log most recent commit
obs-commit abc123       # Log specific commit
```

### obs-decision

Create a decision record.

```bash
obs-decision "Tech stack choice"    # Interactive ADR creation
```

---

## Adapter Specifications

### adapter-codex.sh

**Input:** Reads `~/.ai/context/mcp-servers.json`

**Output:** Writes `~/.codex/config.toml`

```toml
# Auto-generated by ai-mcp distribute
# Do not edit manually

[mcp_servers.serena]
command = "uvx"
args = ["--from", "serena-mcp", "serena", "--project", "."]

[mcp_servers.playwright]
command = "npx"
args = ["@anthropic/mcp-playwright"]
```

### adapter-claude-code.sh

**Input:** Reads `~/.ai/context/mcp-servers.json`

**Output:** Updates `~/.claude/settings.json` mcpServers section

### adapter-opencode.sh

**Input:** Reads `~/.ai/context/mcp-servers.json`

**Output:** Updates OpenCode MCP config (format TBD based on OpenCode docs)

---

## Hooks

| Event | Trigger | Action |
|-------|---------|--------|
| Session start | AGENTS.md instructions | `obs-read` |
| Session end | Manual or agent prompt | `obs-write` |
| Commit | git post-commit hook | `obs-commit` |
| Decision made | Manual | `obs-decision` |
| Mode switch | `ai-mode` command | Log to changelog |
| Agent switch | `ai-handoff` command | Log to changelog + context.md |
| MCP change | `ai-mcp enable/disable` | Auto-redistribute |

---

## Migration Path

### Phase 1: Fix MCP (no breaking changes)
- Update `adapter-codex.sh` to output TOML
- Update `ai-mcp disable` to auto-redistribute
- Test with existing setup

### Phase 2: Add Mode Switching
- Create `~/.ai/modes/work/` and `~/.ai/modes/self/`
- Implement `ai-mode` command
- Migrate existing identity/rules to self mode

### Phase 3: Add Obsidian Integration
- Implement `obs-write`, `obs-commit`, `obs-decision`
- Update `obs-read` for new structure
- Create Obsidian templates

### Phase 4: Add Multi-Repo Support
- Implement `ai-init --multi-repo`
- Create workspace templates
- Document taskboard conventions (work mode)

### Phase 5: Add Agent Handoff
- Implement `ai-handoff`
- Update AGENTS.md template with handoff instructions
- Test cross-agent context continuity

---

## Open Questions (Resolved)

| Question | Resolution |
|----------|------------|
| Multi-repo init creates repos? | No, just workspace structure |
| Taskboard for GitHub too? | No, only GitLab (work mode) |
| Progress sync? | Both project and Obsidian |
| Templates location? | Obsidian only |
| Changelog format? | Full structure (changelog + commits + decisions) |

---

## Success Criteria

1. `ai-mcp distribute codex` writes valid TOML that Codex reads
2. `ai-mcp disable X` removes X from all tool configs automatically
3. `ai-mode work/self` switches full persona in <2 seconds
4. `ai-handoff` produces context that next agent can use without re-exploration
5. `obs-write` creates valid Obsidian markdown with working wikilinks
6. `ai-init --multi-repo` creates functional workspace with all symlinks
