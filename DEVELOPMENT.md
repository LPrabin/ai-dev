# ai-dev Development Guide

Architecture decisions, internal design, known issues, and roadmap for future development.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Design Decisions & Rationale](#2-design-decisions--rationale)
3. [Internal Component Map](#3-internal-component-map)
4. [Data Flow Diagrams](#4-data-flow-diagrams)
5. [Adding New Features](#5-adding-new-features)
6. [Known Issues & Tech Debt](#6-known-issues--tech-debt)
7. [Roadmap](#7-roadmap)
8. [Testing Strategy](#8-testing-strategy)

---

## 1. Architecture Overview

### 1.1 Core principle

**One source of truth, many consumers.**

```
                    ┌─────────────────────┐
                    │  ~/.ai/context/     │  ← source of truth
                    │  identity, rules,   │
                    │  skills, agents     │
                    └─────────┬───────────┘
                              │
                     build-context
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ~/.claude/CLAUDE.md  ~/.codex/      ~/.config/opencode/
     (adapter-claude)     instructions.md  system.md
                          (adapter-codex)  (adapter-opencode)
```

### 1.2 Layer model

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 0: Shell environment                                    │
│   AI_MODE, OBSIDIAN_VAULT, AI_PROJECT, AI_HOME               │
├──────────────────────────────────────────────────────────────┤
│ Layer 1: obs-lib (shared library)                             │
│   Transport detection, path resolution, read/write/search    │
├──────────────────────────────────────────────────────────────┤
│ Layer 2: obs-* commands (user-facing)                         │
│   obs-read, obs-write, obs-search                            │
├──────────────────────────────────────────────────────────────┤
│ Layer 3: ai-* commands (workflow)                             │
│   ai-start, ai-mode, ai-init, ai-review, ai-ship, ai-debug │
├──────────────────────────────────────────────────────────────┤
│ Layer 4: build-context + adapters (distribution)              │
│   Assembles context → pushes to each AI tool's config        │
├──────────────────────────────────────────────────────────────┤
│ Layer 5: AI tools (consumers)                                 │
│   Claude Code, Codex CLI, OpenCode                           │
├──────────────────────────────────────────────────────────────┤
│ Layer 6: Obsidian vault (persistence)                         │
│   Work/Projects/*, Self/Topics/*, Library/*                  │
└──────────────────────────────────────────────────────────────┘
```

### 1.3 File ownership

| Directory | Owner | Committed to git? |
|---|---|---|
| `~/.ai/context/` | User (global config) | Yes (ai-dev repo) |
| `~/.ai/skills/` | ai-dev + user extensions | Yes (ai-dev repo) |
| `~/.ai/bin/` | ai-dev (symlinks) | Yes (ai-dev repo) |
| `~/.ai/prompts/` | Generated (build-context) | No |
| `project/AGENTS.md` | User (per-project) | Yes (project repo) |
| `project/.ai/` | User (per-project) | Yes (project repo) |
| `Brain/` (vault) | User (Obsidian) | No (Obsidian Sync or git) |

---

## 2. Design Decisions & Rationale

### D1: Single vault over two vaults

**Decision**: One Obsidian vault with `Work/` and `Self/` top-level folders.

**Why**: Obsidian's value is the knowledge graph. Two vaults fragment the graph — a project can't link to a learning topic. Cross-linking is the differentiator over `.claude/projects/` auto-memory, which is a black box.

**Trade-off**: Larger vault, slightly more complex path logic. Worth it for graph connectivity.

**Alternatives considered**:
- Two vaults with manual cross-referencing → rejected: breaks graph, duplicates search
- Single vault with no folder separation → rejected: mode distinction is valuable for focus

### D2: Obsidian CLI primary, filesystem fallback

**Decision**: Use `obsidian` CLI commands when available, fall back to direct filesystem operations.

**Why**: CLI gives native search, tasks, daily notes, properties, backlinks. Filesystem gives zero-dependency baseline. Users shouldn't be blocked if they don't have the CLI (it's currently paid Early Access).

**Trade-off**: Two code paths to maintain in obs-lib. Mitigated by centralizing all transport logic in one file.

**When to revisit**: When Obsidian CLI becomes free for all users, consider making it the only transport and dropping filesystem fallback.

### D3: obs-lib shared library

**Decision**: Extract all transport logic into `obs-lib`, sourced by every obs-* and ai-* script.

**Why**: The previous version duplicated REST API + filesystem fallback code in every script (obs-read, obs-write, obs-search, build-context). 4x the same ~30 lines of curl/fallback code. obs-lib eliminates this.

**Pattern**: `source "${SCRIPT_DIR}/obs-lib"` at the top of every script that touches Obsidian.

### D4: AGENTS.md as universal project context

**Decision**: `AGENTS.md` is the single project context file. Claude Code reads it via `CLAUDE.md` symlink. Codex and OpenCode read it natively.

**Why**: Every AI tool needs a project context file. Different tools call it different things. AGENTS.md is the neutral name. Symlinks and native support handle the rest.

**Alternative considered**: Generate separate files per tool → rejected: drift risk, maintenance burden.

### D5: Adapter pattern for tool distribution

**Decision**: `build-context` assembles one prompt, then adapter scripts write tool-specific files.

**Why**: Adding a new AI tool means writing one adapter script (~40 lines). The context system doesn't need to know about tool-specific formats.

**How to add a new tool**: Copy an existing adapter, change the output path and format. Register it in `build-context`'s `--adapter all` case.

### D6: Session.md as human-AI task contract

**Decision**: Session.md contains task checkboxes that both the human and AI read/update.

**Why**: The AI needs to know what to work on. The human needs to know what's done. A shared checklist is the simplest contract. With Obsidian CLI, the AI can mark tasks done (`obsidian tasks daily todo`).

### D7: Timestamped append-only logs

**Decision**: Decisions, Skills, Errors are logged as timestamped lines appended to `log.md` files.

**Why**: Append-only is simple, doesn't lose data, and creates a natural timeline. Searching backward through decisions gives you project history. No schema, no database, just grep-friendly text.

### D8: Skills follow the Agent Skills spec

**Decision**: Bundled skills use `SKILL.md` with YAML frontmatter, matching the spec from kepano/obsidian-skills.

**Why**: Standard format means Claude Code, Codex, and OpenCode can all auto-discover and invoke them. No custom format to maintain.

---

## 3. Internal Component Map

### 3.1 obs-lib functions

| Function | Transport | Description |
|---|---|---|
| `obs_cli_available()` | — | Returns true if `obsidian` command exists |
| `obs_vault_set()` | — | Returns true if `OBSIDIAN_VAULT` is a directory |
| `obs_base_path()` | — | Returns mode-aware base path (`Work/Projects/X` or `Self/Topics/X`) |
| `obs_context_note()` | — | Returns path to main context note |
| `obs_session_note()` | — | Returns path to Session.md |
| `obs_sessions_dir()` | — | Returns path to Sessions/ directory |
| `obs_decisions_dir()` | — | Returns path to Decisions/ directory |
| `obs_skills_dir()` | — | Returns path to Skills/ directory |
| `obs_errors_dir()` | — | Returns path to Errors/ directory |
| `obs_session_file()` | — | Returns path to today's session file |
| `obs_read_note(path)` | CLI→FS | Read a note, return content or empty string |
| `obs_write_note(path, content)` | CLI→FS | Append content to a note (create if missing) |
| `obs_overwrite_note(path, content)` | CLI→FS | Overwrite a note entirely |
| `obs_search(query, limit)` | CLI→FS | Full-text search, returns results |
| `obs_tasks_today()` | CLI→FS | Show today's open tasks |
| `obs_task_add(task)` | CLI→FS | Add a task to daily note or Session.md |
| `obs_daily_read()` | CLI only | Read today's daily note |
| `obs_daily_append(content)` | CLI only | Append to daily note |
| `obs_property_set(note, key, value)` | CLI only | Set frontmatter property |
| `obs_backlinks(note)` | CLI→FS | Show backlinks to a note |
| `obs_list_dir(dir, limit)` | FS only | List files in a directory |
| `obs_ensure_dirs()` | FS | Create all subdirectories for current project |
| `obs_require_vault()` | — | Die if no transport is available |
| `obs_banner()` | — | Print mode/project header |
| `obs_footer()` | — | Print vault/mode/transport footer |

### 3.2 Adapter interface

Each adapter script receives one argument: the path to the assembled global.md file.

```bash
# Called by build-context:
bash adapter-claude-code.sh /path/to/global.md
```

The adapter reads global.md and writes to its tool-specific location. It can also generate additional files (e.g., command stubs for Claude Code).

### 3.3 Script dependency graph

```
obs-lib ◄──── obs-read
         ◄──── obs-write
         ◄──── obs-search
         ◄──── ai-mode
         ◄──── ai-start
         ◄──── ai-init ──── obs-write --init
         ◄──── build-context ──── adapter-claude-code.sh
                               ──── adapter-codex.sh
                               ──── adapter-opencode.sh

ai-review ──── (standalone, calls ruff/black/mypy/bandit + AI tool)
ai-ship   ──── obs-write --decision (optional)
ai-debug  ──── (standalone, calls pytest/pdb/profiler)
```

---

## 4. Data Flow Diagrams

### 4.1 Session start (`ai-start`)

```
User runs: ai-start
    │
    ├─ source obs-lib
    │   ├─ detect transport (CLI? filesystem?)
    │   └─ resolve paths (mode-aware)
    │
    ├─ obs_read_note(Session.md)
    │   ├─ CLI: obsidian read path="Work/Projects/X/Session.md"
    │   └─ FS:  cat $VAULT/Work/Projects/X/Session.md
    │
    ├─ obs_read_note(Decisions/log.md)
    │
    ├─ obs_daily_read() [CLI only]
    │
    └─ print status + quick reference
```

### 4.2 Context build (`build-context --obsidian --adapter all`)

```
User runs: build-context --obsidian --adapter all
    │
    ├─ assemble()
    │   ├─ cat identity.md
    │   ├─ cat rules/*.md
    │   ├─ summarize skills/*.md
    │   ├─ cat agents/*.md
    │   └─ get_obsidian_context()
    │       ├─ obs_read_note(README.md)
    │       ├─ obs_read_note(Session.md)
    │       └─ obs_read_note(Decisions/log.md)
    │
    ├─ write → ~/.ai/prompts/global.md
    │
    └─ for each adapter:
        ├─ adapter-claude-code.sh global.md → ~/.claude/CLAUDE.md
        ├─ adapter-codex.sh global.md      → ~/.codex/instructions.md
        └─ adapter-opencode.sh global.md   → ~/.config/opencode/system.md
```

### 4.3 Decision logging (`obs-write --decision "text"`)

```
User runs: obs-write --decision "Using JWT over sessions"
    │
    ├─ source obs-lib
    │
    ├─ format: "- [2026-03-31 14:30] Decision: Using JWT over sessions"
    │
    ├─ obs_write_note(Decisions/log.md, entry)
    │   ├─ CLI: obsidian append path="Work/Projects/X/Decisions/log.md" content="..." silent
    │   └─ FS:  echo "..." >> $VAULT/Work/Projects/X/Decisions/log.md
    │
    └─ obs_write_note(Sessions/2026-03-31.md, entry)
        └─ (same pattern — dual write for traceability)
```

---

## 5. Adding New Features

### 5.1 Add a new AI tool

1. Create `bin/adapters/adapter-newtool.sh`:
```bash
#!/usr/bin/env bash
# adapter-newtool.sh — write global context to newtool's config
GLOBAL_MD="$1"
OUTPUT="${HOME}/.newtool/system.md"
mkdir -p "$(dirname "$OUTPUT")"
cp "$GLOBAL_MD" "$OUTPUT"
echo "[adapter-newtool] wrote: ${OUTPUT}"
```

2. Register in `build-context`:
```bash
# In the case statement at the bottom:
case "$ADAPTER" in
  all) for t in claude-code codex opencode newtool; do run_adapter "$t"; done ;;
```

3. Add detection in `ai-init`:
```bash
command -v newtool &>/dev/null && "$build_script" --adapter newtool
```

### 5.2 Add a new context rule

1. Create `~/.ai/context/rules/newrule.md`:
```markdown
# New Rule Category

- Rule 1
- Rule 2
```

2. Rebuild: `build-context --adapter all`

No code changes needed — `build-context` auto-discovers `rules/*.md`.

### 5.3 Add a new skill

1. Create `~/.ai/skills/my-skill/SKILL.md`:
```markdown
---
name: my-skill
description: When to use this skill.
allowed-tools: Read, Write, Bash
---

# My Skill

Instructions for the AI agent...
```

2. Skills are auto-discovered by Claude Code, Codex, and OpenCode.

### 5.4 Add a new agent persona

1. Create `~/.ai/context/agents/newagent.md`:
```markdown
# New Agent

## Role
...

## Constraints
...
```

2. Rebuild: `build-context --adapter all`

### 5.5 Add a new obs-lib function

1. Add the function to `bin/obs-lib`
2. Follow the transport pattern:
```bash
obs_new_function() {
  local arg="$1"
  if obs_cli_available; then
    obsidian some-command arg="${arg}" 2>/dev/null
  elif obs_vault_set; then
    # filesystem fallback
  else
    echo "(requires CLI or vault)" >&2
  fi
}
```

3. Use it from any script that sources obs-lib.

### 5.6 Add a new vault subdirectory

1. Add path helper to `obs-lib`:
```bash
obs_new_dir() { echo "$(obs_base_path)/NewDir"; }
```

2. Add to `obs_ensure_dirs()`:
```bash
mkdir -p "${OBS_VAULT}/$(obs_new_dir)"
```

3. Add read/write commands to `obs-write` and `obs-read` as needed.

---

## 6. Known Issues & Tech Debt

### 6.1 Active issues

| ID | Severity | Issue | Location | Workaround |
|---|---|---|---|---|
| I1 | Medium | `ai-mode work/self` export doesn't persist across shell sessions | `ai-mode` | Use `ai-work`/`ai-self` aliases in `.zshrc` instead |
| I2 | Low | Temp files in `/tmp` not explicitly cleaned up | `obs-write`, `ai-debug` | OS handles cleanup; low risk |
| I3 | Low | `obs_read_note` returns empty string on both "not found" and "empty file" | `obs-lib` | Not distinguishable; benign in practice |
| I4 | Low | `ai-review` assumes `uv` is installed | `ai-review` | Install uv or modify to detect package manager |
| I5 | Low | Secret scan regex may have false positives | `ai-review:77-78` | Review flagged items manually |
| I6 | Low | `obs-search --cross` filesystem fallback is grep-based, slow on large vaults | `obs-search` | Use Obsidian CLI for better search |
| I7 | Low | No validation that `OBSIDIAN_VAULT` directory exists before operations | multiple | Set path correctly; `obs_require_vault` catches missing |

### 6.2 Tech debt

| Item | Effort | Impact | Description |
|---|---|---|---|
| **Test suite** | High | High | No automated tests for any shell scripts. See [Section 8](#8-testing-strategy). |
| **Error messages** | Low | Medium | Some error messages reference old env vars or paths |
| **`ai-review` package manager detection** | Low | Medium | Hardcoded to `uv`; should detect pip/poetry/uv |
| **Adapter for Cursor** | Medium | Medium | Cursor IDE is popular but has no adapter yet |
| **Adapter for Aider** | Medium | Low | Aider is a popular coding assistant |
| **`obs-lib` CLI command validation** | Low | Low | No check that specific CLI subcommands exist in user's version |
| **Shell completion** | Medium | Medium | No tab completion for ai-* and obs-* commands |
| **`ai-ship` configurable branch** | Low | Low | Hardcoded main/master check |

---

## 7. Roadmap

### Phase 1: Stability (current)

- [x] Single vault architecture
- [x] Obsidian CLI integration with filesystem fallback
- [x] obs-lib shared library
- [x] ai-start convenience command
- [x] Cross-linking between Work and Self
- [x] Bundled skills (obsidian-cli, obsidian-markdown)
- [x] Complete documentation (USAGE.md, DEVELOPMENT.md)
- [ ] Fix known issues I1-I7
- [ ] Basic test suite for obs-lib functions

### Phase 2: Intelligence

- [ ] **Auto-context selection**: `ai-start` automatically includes relevant decisions and skills based on current task, not just latest entries
- [ ] **Session continuity**: When resuming a session, show diff from last session (what changed, what's new)
- [ ] **Smart cross-linking**: After `obs-write --decision`, suggest related notes in other projects/topics
- [ ] **Task management**: Full task lifecycle using Obsidian CLI (`obs-task add/done/list`)
- [ ] **Daily note integration**: Auto-append session summaries to Obsidian daily note at session end
- [ ] **Property-based metadata**: Use Obsidian frontmatter properties for project status, priority, tags

### Phase 3: Multi-Agent Orchestration

- [ ] **Agent routing**: Based on task type, automatically suggest which AI tool to use (Claude for architecture, Codex for quick edits, etc.)
- [ ] **Agent handoff**: Pass context from one tool to another mid-session (e.g., plan with Claude, implement with Codex)
- [ ] **Parallel agents**: Run review agent and implementation agent simultaneously on different parts of a project
- [ ] **Agent memory sharing**: All agents read/write to the same Obsidian vault — shared memory across tools
- [ ] **Agent-specific skills**: Different skill sets loaded per tool (e.g., Claude gets planner skill, Codex gets speed-coder skill)

### Phase 4: Ecosystem

- [ ] **MCP server**: Expose obs-lib functions as an MCP server so AI tools can read/write Obsidian natively without shell commands
- [ ] **Obsidian plugin**: Sidebar panel in Obsidian showing current AI session, tasks, and recent decisions
- [ ] **GitHub Actions integration**: Auto-create Obsidian session notes from PR descriptions
- [ ] **Team vaults**: Shared vault for team decisions, with per-user session logs
- [ ] **Template library**: Pre-built AGENTS.md templates for common project types (FastAPI, React, CLI tool, etc.)
- [ ] **Metrics/analytics**: Track session duration, decisions per project, fix frequency — visible in Obsidian dashboard

### Phase 5: Language & Stack Expansion

- [ ] **TypeScript/JavaScript rules and skills**: Extend beyond Python-focused defaults
- [ ] **Rust rules and skills**
- [ ] **Go rules and skills**
- [ ] **`ai-review` stack detection**: Auto-detect language/framework and run appropriate linters
- [ ] **`ai-init` stack templates**: `ai-init --stack fastapi`, `ai-init --stack nextjs`, etc.

### Ideas (unvalidated)

- **Research mode**: A third mode alongside Work and Self for time-boxed investigations (compare libraries, prototype approaches). Would need its own vault folder and session structure.
- **Obsidian Sync as backup**: Use Obsidian Sync to replicate vault across machines — test if this works reliably with CLI writes.
- **Voice notes**: Integrate with Obsidian voice note plugins for quick decision logging via speech.
- **AI-generated weekly reports**: Aggregate Sessions/*.md into a weekly summary note automatically.
- **Dependency graph**: Track which projects depend on which topics. When a topic gets updated, flag dependent projects.

---

## 8. Testing Strategy

### 8.1 Current state

**No automated tests exist.** All scripts have been manually tested. This is the highest-priority tech debt item.

### 8.2 Recommended approach

Use [bats-core](https://github.com/bats-core/bats-core) (Bash Automated Testing System):

```bash
# Install
brew install bats-core

# Run
bats tests/
```

### 8.3 Test structure

```
tests/
├── test_obs_lib.bats        # obs-lib function tests
├── test_obs_read.bats       # obs-read integration tests
├── test_obs_write.bats      # obs-write integration tests
├── test_obs_search.bats     # obs-search integration tests
├── test_ai_mode.bats        # ai-mode tests
├── test_ai_start.bats       # ai-start tests
├── test_ai_init.bats        # ai-init scaffold tests
├── test_build_context.bats  # build-context assembly tests
├── fixtures/
│   └── mock_vault/          # test vault with known structure
└── helpers/
    └── setup.bash           # shared test helpers
```

### 8.4 Key test scenarios

**obs-lib (unit)**:
- `obs_base_path` returns correct path for work/self mode
- `obs_read_note` returns empty string for missing files
- `obs_write_note` creates parent directories
- `obs_write_note` appends (not overwrites) by default
- `obs_overwrite_note` overwrites existing content
- Transport detection: CLI present, CLI absent, vault set, vault unset

**obs-read (integration)**:
- `--session` reads only Session.md
- `--all` includes decisions, errors, skills, cross-refs
- Missing vault prints helpful error

**obs-write (integration)**:
- `--init` creates full directory structure
- `--decision` writes to both Decisions/log.md and Sessions/date.md
- `--fix` writes to both Errors/log.md and Sessions/date.md
- `--skill` writes to both Skills/log.md and Sessions/date.md
- `--link` adds wikilink to context note
- Pipe input works: `echo "test" | obs-write`

**ai-init (integration)**:
- Creates AGENTS.md with template substitution
- Creates CLAUDE.md symlink when claude is in PATH
- Creates .ai/rules/ and .ai/skills/ directories
- Skips existing files

**build-context (integration)**:
- Assembles all context files in correct order
- `--obsidian` includes vault content
- `--adapter all` runs all adapters
- Output file contains identity + rules + skills + agents

### 8.5 Test fixtures

Create a mock vault for testing:

```bash
# tests/helpers/setup.bash
setup() {
  export OBSIDIAN_VAULT="$(mktemp -d)"
  export AI_MODE="work"
  export AI_PROJECT="test-project"
  export AI_HOME="$(mktemp -d)"

  # Create mock vault structure
  mkdir -p "$OBSIDIAN_VAULT/Work/Projects/test-project"/{Sessions,Decisions,Skills,Errors}
  echo "# Test Project" > "$OBSIDIAN_VAULT/Work/Projects/test-project/README.md"
  echo "- [ ] Test task" > "$OBSIDIAN_VAULT/Work/Projects/test-project/Session.md"
}

teardown() {
  rm -rf "$OBSIDIAN_VAULT" "$AI_HOME"
}
```

---

## Contributing

### Code style

- Shell scripts: `set -euo pipefail` at the top
- Functions: `snake_case` with `obs_` or `ai_` prefix
- Error output: `>&2` for all errors
- Exit codes: 0 success, 1 error
- Comments: `# ── section ──` for visual structure

### Adding a feature

1. Check if it belongs in obs-lib (transport-level) or a command (user-facing)
2. Follow the transport pattern: CLI primary, filesystem fallback
3. Add to USAGE.md command reference
4. Add test scenarios to this file's testing section
5. Update DEVELOPMENT.md roadmap if it's a significant feature

### Commit message format

```
<type>: <description>

Types: add, update, fix, remove, refactor, docs, test
Examples:
  add: ai-start session startup command
  fix: obs-write --init creates missing parent dirs
  update: obs-lib CLI syntax for Obsidian v1.13
  docs: add testing strategy to DEVELOPMENT.md
```
