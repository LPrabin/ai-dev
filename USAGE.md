# ai-dev Usage Guide

Complete reference for every command, flow, and pattern in ai-dev.

---

## Table of Contents

1. [First-Time Setup](#1-first-time-setup)
2. [Session Lifecycle](#2-session-lifecycle)
3. [Work Mode — Project Development](#3-work-mode--project-development)
4. [Self Mode — Learning & Exploration](#4-self-mode--learning--exploration)
5. [Cross-Linking Work and Self](#5-cross-linking-work-and-self)
6. [Command Reference](#6-command-reference)
7. [Obsidian Vault Management](#7-obsidian-vault-management)
8. [Multi-Agent Workflow](#8-multi-agent-workflow)
9. [Context System Deep Dive](#9-context-system-deep-dive)
10. [Real-World Flows](#10-real-world-flows)

---

## 1. First-Time Setup

### 1.1 Install ai-dev globally

```bash
git clone https://github.com/yourname/ai-dev ~/.ai-dev
cd ~/.ai-dev
./bin/ai-init --global-install
source ~/.zshrc
```

This installs:
- `~/.ai/context/` — identity, rules, skills, agents
- `~/.ai/skills/` — bundled skills (obsidian-cli, obsidian-markdown)
- `~/.ai/bin/` — all commands symlinked to PATH
- Shell environment block in `~/.zshrc`

### 1.2 Configure your vault

Edit `~/.zshrc` and set your vault path:

```bash
export OBSIDIAN_VAULT="$HOME/Documents/Brain"
```

Then initialize the vault structure:

```bash
source ~/.zshrc
ai-mode --init
```

This creates:
```
Brain/
├── Work/Projects/       # your coding projects
├── Self/Topics/         # your learning areas
└── Library/             # cross-cutting shared knowledge
    ├── Decisions/
    ├── Skills/
    └── Errors/
```

### 1.3 Verify everything works

```bash
ai-mode              # shows current status, vault path, transport
ai-mode --list       # lists all projects and topics (empty at first)
```

### 1.4 Transport detection

ai-dev automatically detects what's available:

| Transport | Detected when | Features |
|---|---|---|
| **Obsidian CLI** | `obsidian` command exists | Full: search, tasks, daily notes, properties, backlinks |
| **Filesystem** | `OBSIDIAN_VAULT` directory exists | Basic: read, write, append, grep-based search |

You don't configure this. `obs-lib` handles it transparently.

---

## 2. Session Lifecycle

Every AI coding session follows this cycle:

```
┌─────────────────────────────────────────────┐
│  1. ai-start                                │
│     → shows mode, project, tasks, decisions │
│     → pulls context from Obsidian           │
├─────────────────────────────────────────────┤
│  2. Work with your AI tool                  │
│     → Claude Code / Codex / OpenCode        │
│     → reads AGENTS.md + global context      │
├─────────────────────────────────────────────┤
│  3. Log as you go                           │
│     → obs-write --decision "..."            │
│     → obs-write --fix "..."                 │
│     → obs-write --skill "..."              │
├─────────────────────────────────────────────┤
│  4. Review before shipping                  │
│     → ai-review                             │
│     → ai-ship                               │
├─────────────────────────────────────────────┤
│  5. End session                             │
│     → obs-write  (full session log)         │
└─────────────────────────────────────────────┘
```

### 2.1 Starting a session

```bash
# Basic — uses current AI_MODE and directory name
ai-start

# Switch mode at startup
ai-start work
ai-start self

# Also rebuild context and push to all AI tools
ai-start --rebuild
```

`ai-start` shows:
- Current mode and project
- Transport (CLI or filesystem)
- Open and completed tasks from Session.md
- Last 5 decisions
- Today's daily note (if CLI available)
- Quick reference of available commands

### 2.2 During a session

**Log decisions as you make them:**
```bash
obs-write --decision "Using JWT over sessions — stateless, scales horizontally"
obs-write --decision "PostgreSQL over SQLite — need concurrent writes"
```

**Log bug fixes:**
```bash
obs-write --fix "user.email was null when OAuth provider omits it — added fallback"
obs-write --fix "race condition in session refresh — added mutex"
```

**Log skills and patterns you learn:**
```bash
obs-write --skill "Pydantic model_validator(mode='after') for cross-field validation"
obs-write --skill "pytest.mark.parametrize with indirect fixtures for DB variants"
```

**Update session tasks:**
```bash
obs-write --session "Completed auth module, starting rate limiting next"
```

All logs are timestamped and written to both the category log (`Decisions/log.md`, etc.) and the daily session file (`Sessions/YYYY-MM-DD.md`).

### 2.3 Code review and shipping

```bash
# Before committing — run linters + AI review
ai-review

# Lint only (no AI)
ai-review --lint-only

# Before PR — full pre-ship checklist
ai-ship
```

`ai-review` runs: ruff, black (check), mypy, bandit, secret scan, then asks your AI tool for a review.

`ai-ship` runs: pytest, mypy, ruff, bandit, pip-audit, secret scan, branch check, and logs the ship event to Obsidian.

### 2.4 Debugging

```bash
# Interactive debug session
ai-debug

# Run specific test in debug mode
ai-debug --test tests/test_auth.py

# Analyze a specific error
ai-debug --error "TypeError: 'NoneType' object is not subscriptable"

# Profile performance
ai-debug --profile module.function
```

### 2.5 Ending a session

```bash
obs-write
```

Opens your `$EDITOR` with a pre-filled template:

```markdown
## [14:30] my-project — work session

### Decisions
-

### Fixes / discoveries
-

### Skills learned
-

### Open
-

### Notes
-
```

Fill it in, save, close. It's written to `Sessions/YYYY-MM-DD.md` in your vault.

You can also pipe content:

```bash
echo "Quick note: auth module complete" | obs-write
```

---

## 3. Work Mode — Project Development

Work mode is for shipping code. Each project gets its own vault folder.

### 3.1 Scaffold a new project

```bash
ai-work                      # ensure work mode
cd ~/code/my-api
ai-init                       # uses directory name: "my-api"
# or
ai-init my-custom-name        # explicit name
```

Creates in your project directory:
```
my-api/
├── AGENTS.md                 # edit this — your project's AI context
├── CLAUDE.md -> AGENTS.md    # symlink (Claude Code only)
└── .ai/
    ├── rules/conventions.md  # project-specific rules
    └── skills/stack.md       # project stack notes
```

Creates in your Obsidian vault:
```
Brain/Work/Projects/my-api/
├── README.md                 # project context
├── Session.md                # current AI tasks
├── Sessions/                 # daily logs
├── Decisions/log.md          # decision trail
├── Skills/log.md             # patterns learned
└── Errors/log.md             # bug fixes
```

### 3.2 Edit AGENTS.md

This is your highest-leverage file. Keep it under 200 lines. Include:

- Stack and versions
- Run/test/lint commands
- Architecture decisions (the "why")
- Conventions that differ from defaults
- Known gotchas

### 3.3 Edit Session.md

Session.md is your **task handoff** between you and the AI. Before starting a session, update it:

```markdown
# Session — 2026-03-31

## Tasks
- [ ] Implement rate limiting middleware
- [ ] Add Redis cache layer for /users endpoint
- [ ] Fix flaky test in test_auth.py

## Notes
- Rate limiting: use sliding window, not fixed window
- Redis: connect via connection pool, not per-request
```

### 3.4 List all projects

```bash
ai-mode --list
```

Shows all projects with open task counts.

---

## 4. Self Mode — Learning & Exploration

Self mode is for building knowledge. Each topic gets its own vault folder.

### 4.1 Create a new topic

```bash
ai-self                       # switch to self mode
cd ~/learning/rust            # or any directory — name is auto-detected
ai-init rust                  # creates topic structure
```

Creates in your vault:
```
Brain/Self/Topics/rust/
├── learnings.md              # consolidated topic notes
├── Session.md                # current exploration tasks
├── Sessions/
├── Decisions/log.md
├── Skills/log.md
└── Errors/log.md
```

### 4.2 Self mode tasks are exploratory

```markdown
# Session — 2026-03-31

## Tasks
- [ ] Understand ownership and borrowing model
- [ ] Build a CLI tool with clap
- [ ] Compare error handling: Result vs anyhow vs thiserror

## Resources
- The Rust Book ch. 4
- Jon Gjengset's YouTube channel
```

### 4.3 Log learnings

```bash
obs-write --skill "Rust lifetimes: 'a means 'at least as long as a'"
obs-write --decision "Using anyhow for applications, thiserror for libraries"
```

---

## 5. Cross-Linking Work and Self

This is the killer feature of the single-vault design.

### 5.1 Link a project to a topic

```bash
# While in work mode, working on my-api
obs-write --link "Self/Topics/python"
obs-write --link "Self/Topics/security"
```

This adds `[[Self/Topics/python]]` wikilinks to your project's README.md.

### 5.2 Discover cross-references

```bash
# Search across both Work and Self for a keyword
obs-search --cross "authentication"
```

Output:
```
=== Cross-search: "authentication" (both Work and Self) ===

## Work
  - Work/Projects/my-api/Decisions/log.md
    Decision: Using JWT tokens for authentication
  - Work/Projects/auth-service/README.md

## Self
  - Self/Topics/security/learnings.md
    OWASP authentication cheat sheet
```

### 5.3 Manual wikilinks in notes

In any Obsidian note, use wikilinks:

```markdown
<!-- In Work/Projects/my-api/README.md -->
Auth patterns from: [[Self/Topics/security/learnings]]

<!-- In Self/Topics/python/learnings.md -->
Applied in: [[Work/Projects/my-api/README|my-api project]]
```

Obsidian tracks renames automatically, so links don't break when you reorganize.

### 5.4 Backlinks

With Obsidian CLI:

```bash
obsidian backlinks path="Work/Projects/my-api/README.md"
```

Or from obs-lib:

```bash
# In obs-read --all, cross-references are shown automatically
obs-read --all
```

---

## 6. Command Reference

### Session management

| Command | Description |
|---------|-------------|
| `ai-start` | Begin session — shows mode, tasks, decisions, status |
| `ai-start work` | Switch to work mode and start |
| `ai-start self` | Switch to self mode and start |
| `ai-start --rebuild` | Start + rebuild context for all AI tools |

### Mode switching

| Command | Description |
|---------|-------------|
| `ai-work` | Quick alias: `export AI_MODE="work"` |
| `ai-self` | Quick alias: `export AI_MODE="self"` |
| `ai-mode` | Show current status (mode, vault, transport, tasks) |
| `ai-mode work` | Switch to work mode |
| `ai-mode self` | Switch to self mode |
| `ai-mode --init` | Create vault directory structure |
| `ai-mode --list` | List all projects and topics |

### Reading from Obsidian

| Command | Description |
|---------|-------------|
| `obs-read` | Read project/topic context (README.md or learnings.md) |
| `obs-read --session` | Read Session.md tasks only |
| `obs-read --all` | Full context: main + session + decisions + errors + skills + cross-refs |

### Writing to Obsidian

| Command | Description |
|---------|-------------|
| `obs-write` | Interactive session log (opens $EDITOR) |
| `obs-write --init` | Create project/topic vault structure |
| `obs-write --session "text"` | Update Session.md |
| `obs-write --decision "text"` | Log a decision (timestamped) |
| `obs-write --fix "text"` | Log a bug fix (timestamped) |
| `obs-write --skill "text"` | Log a learned pattern (timestamped) |
| `obs-write --link "path"` | Add cross-link to another note |
| `echo "text" \| obs-write` | Pipe content as session log |

### Searching Obsidian

| Command | Description |
|---------|-------------|
| `obs-search "query"` | Full-text search across vault |
| `obs-search --project` | Show project/topic context |
| `obs-search --session` | Show current Session.md |
| `obs-search --decisions` | Show decisions log |
| `obs-search --recent 5` | Show last N session logs |
| `obs-search --cross "query"` | Search across both Work and Self |

### Code quality

| Command | Description |
|---------|-------------|
| `ai-review` | Linters (ruff, black, mypy, bandit) + AI review |
| `ai-review --lint-only` | Mechanical checks only, no AI |
| `ai-ship` | Full pre-ship checklist (tests, types, lint, audit, secrets) |

### Debugging

| Command | Description |
|---------|-------------|
| `ai-debug` | Interactive debug protocol |
| `ai-debug --test path/to/test.py` | Run specific test in debug mode |
| `ai-debug --error "message"` | Analyze a specific error |
| `ai-debug --profile module.func` | Profile function performance |

### Context building

| Command | Description |
|---------|-------------|
| `build-context` | Assemble ~/.ai/context/ into ~/.ai/prompts/global.md |
| `build-context --obsidian` | Include current Obsidian project context |
| `build-context --adapter all` | Push to all detected AI tools |
| `build-context --adapter claude-code` | Push to Claude Code only |
| `build-context --stdout` | Print assembled context to stdout |

### Project scaffolding

| Command | Description |
|---------|-------------|
| `ai-init` | Scaffold current directory (uses dirname as project name) |
| `ai-init my-project` | Scaffold with explicit name |
| `ai-init --global-install` | One-time: install ~/.ai, PATH, shell config |

---

## 7. Obsidian Vault Management

### 7.1 Vault structure

```
Brain/                              # OBSIDIAN_VAULT
├── Work/
│   └── Projects/
│       ├── my-api/
│       │   ├── README.md           # project context
│       │   ├── Session.md          # current AI tasks
│       │   ├── Sessions/
│       │   │   ├── 2026-03-30.md   # daily session logs
│       │   │   └── 2026-03-31.md
│       │   ├── Decisions/
│       │   │   └── log.md          # all decisions, timestamped
│       │   ├── Skills/
│       │   │   └── log.md          # patterns learned
│       │   └── Errors/
│       │       └── log.md          # bug fixes
│       └── auth-service/
│           └── ...
├── Self/
│   └── Topics/
│       ├── python/
│       │   ├── learnings.md
│       │   ├── Session.md
│       │   └── ...
│       └── security/
│           └── ...
└── Library/                        # cross-cutting (manual use)
    ├── Decisions/
    ├── Skills/
    └── Errors/
```

### 7.2 Log file format

All log files use the same timestamp format:

```markdown
- [2026-03-31 14:30] Decision: Using JWT over sessions — stateless, scales better
- [2026-03-31 15:15] Decision: Redis for rate limiting — need distributed counters
```

### 7.3 Session file format

Daily session files in `Sessions/YYYY-MM-DD.md`:

```markdown
## [14:30] my-api — work session

### Decisions
- Chose sliding window for rate limiting

### Fixes / discoveries
- Fixed race condition in token refresh

### Skills learned
- Redis MULTI/EXEC for atomic counter operations

### Open
- Need to benchmark under load

### Notes
- Pair-programmed with Claude Code, 2hr session
```

### 7.4 Using Obsidian features

Because everything is in one vault, you can use Obsidian's full power:

- **Graph view**: See connections between projects and topics
- **Search**: `ctrl+shift+F` for vault-wide search
- **Tags**: Add `#active`, `#archived`, `#blocked` to notes
- **Properties**: Add frontmatter for structured metadata
- **Templates**: Use Obsidian templates for consistent note creation
- **Daily notes**: Obsidian CLI integrates with daily notes plugin

---

## 8. Multi-Agent Workflow

### 8.1 Using different AI tools

ai-dev is tool-agnostic. The same session works across tools:

```bash
# Start session (works for any tool)
ai-start

# Use Claude Code
claude

# Or use Codex CLI
codex

# Or use OpenCode
opencode
```

All tools read from:
- `AGENTS.md` in project root (project-specific context)
- Their global config file (generated by adapters from `~/.ai/context/`)

### 8.2 Switching tools mid-session

```bash
# Rebuild context for a specific tool
build-context --obsidian --adapter claude-code

# Or rebuild for all
build-context --obsidian --adapter all
```

### 8.3 Tool-specific overrides

Set `AI_TOOL` to override auto-detection in shell commands:

```bash
AI_TOOL=codex ai-review       # use Codex for the AI review step
AI_TOOL=none ai-ship           # run checklist without AI, print for manual paste
```

### 8.4 Where each tool reads context

| Tool | Project context | Global context |
|---|---|---|
| Claude Code | `CLAUDE.md` → `AGENTS.md` (symlink) | `~/.claude/CLAUDE.md` |
| Codex CLI | `AGENTS.md` (native) | `~/.codex/instructions.md` |
| OpenCode | `AGENTS.md` (native) | `~/.config/opencode/system.md` |

### 8.5 Bundled skills

ai-dev ships two skills compatible with the [Agent Skills spec](https://github.com/kepano/obsidian-skills):

| Skill | Location | Purpose |
|---|---|---|
| `obsidian-cli` | `~/.ai/skills/obsidian-cli/SKILL.md` | Teaches AI agents Obsidian CLI commands |
| `obsidian-markdown` | `~/.ai/skills/obsidian-markdown/SKILL.md` | Teaches AI agents Obsidian-flavored markdown |

These are auto-invoked by Claude Code and Codex when the task involves Obsidian files.

---

## 9. Context System Deep Dive

### 9.1 Context layering

Context is assembled in layers, from broadest to most specific:

```
Layer 1: Identity       (~/.ai/context/identity.md)
   ↓
Layer 2: Global rules   (~/.ai/context/rules/*.md)
   ↓
Layer 3: Global skills  (~/.ai/context/skills/*.md)
   ↓
Layer 4: Agent personas (~/.ai/context/agents/*.md)
   ↓
Layer 5: Project rules  (.ai/rules/conventions.md)
   ↓
Layer 6: Project skills (.ai/skills/stack.md)
   ↓
Layer 7: AGENTS.md      (project root — the AI reads this directly)
   ↓
Layer 8: Obsidian       (Session.md, Decisions, Skills — pulled by obs-read)
```

Layers 1-4 are assembled by `build-context` into `~/.ai/prompts/global.md`, then pushed to each tool's global config via adapters.

Layers 5-7 are read directly by the AI tool from the project directory.

Layer 8 is pulled on-demand by `obs-read` or `ai-start`.

### 9.2 Editing global context

```bash
# Edit a rule
$EDITOR ~/.ai/context/rules/python.md

# Add a new rule
$EDITOR ~/.ai/context/rules/myteam.md

# Rebuild and push to all tools
build-context --adapter all
```

### 9.3 Editing project context

```bash
# Project-specific rules
$EDITOR .ai/rules/conventions.md

# Project-specific stack notes
$EDITOR .ai/skills/stack.md

# Main AI context
$EDITOR AGENTS.md
```

No rebuild needed — AI tools read these directly.

### 9.4 Context files reference

| File | Purpose | Example content |
|---|---|---|
| `identity.md` | Who the AI is, how it works | Senior Python engineer persona, session cycle |
| `rules/general.md` | Planning, code quality, git | Plan before code, test after, never commit secrets |
| `rules/python.md` | Python-specific rules | Python 3.12, uv, types everywhere, forbidden patterns |
| `rules/security.md` | Security rules | Parameterized queries, no secrets in code, input validation |
| `skills/python.md` | Python patterns | Project layout, Pydantic, service pattern, subprocess |
| `skills/tdd.md` | Testing patterns | Red-green-refactor, pytest, conftest, what to test |
| `skills/debugging.md` | Debug protocol | 8-step process, pdb, profiling, common bugs |
| `agents/planner.md` | Plan-before-code persona | Asks clarifying questions, assesses complexity |
| `agents/reviewer.md` | Code review persona | Correctness, types, tests, style scoring |

---

## 10. Real-World Flows

### 10.1 Flow: Starting a new project

```bash
# 1. Create project directory
mkdir ~/code/invoice-api && cd ~/code/invoice-api
git init

# 2. Scaffold
ai-init

# 3. Edit AGENTS.md with your stack
$EDITOR AGENTS.md

# 4. Set up Obsidian session
obs-write --session "Initial setup: project structure, FastAPI boilerplate, DB schema"

# 5. Build context
build-context --obsidian --adapter all

# 6. Start coding with your AI tool
ai-start
claude  # or codex / opencode
```

### 10.2 Flow: Daily coding session

```bash
# 1. Start
cd ~/code/invoice-api
ai-start

# 2. Review tasks from yesterday's session
obs-read --session

# 3. Work with AI tool
claude

# 4. Log decisions as you go
obs-write --decision "Using Stripe webhooks, not polling — real-time, less infra"
obs-write --fix "PDF generation OOM on large invoices — streaming with reportlab"

# 5. Before committing
ai-review

# 6. Before PR
ai-ship

# 7. End session
obs-write
```

### 10.3 Flow: Debugging a production issue

```bash
# 1. Start debug session
ai-start
ai-debug --error "ConnectionResetError in /api/invoices after 30s"

# 2. Log what you find
obs-write --fix "DB connection pool exhausted — increased max_size from 5 to 20"
obs-write --decision "Adding connection pool monitoring via prometheus"

# 3. Verify fix
ai-debug --test tests/test_db_pool.py

# 4. Ship
ai-review
ai-ship
```

### 10.4 Flow: Learning a new topic

```bash
# 1. Switch to self mode
ai-self
mkdir ~/learning/kubernetes && cd ~/learning/kubernetes
ai-init kubernetes

# 2. Set exploration tasks
obs-write --session "Understand pods, services, deployments. Build a local cluster."

# 3. Work with AI
ai-start
claude  # ask questions, build examples

# 4. Log what you learn
obs-write --skill "kubectl apply -f creates/updates, kubectl create only creates"
obs-write --skill "Services use label selectors, not direct pod references"
obs-write --decision "Using k3s for local dev — lighter than minikube"

# 5. Cross-link to a project that needs K8s
obs-write --link "Work/Projects/invoice-api"

# 6. End session
obs-write
```

### 10.5 Flow: Switching between projects

```bash
# Working on project A
ai-work
cd ~/code/invoice-api
ai-start
# ... work ...
obs-write                    # end session log

# Switch to project B
cd ~/code/auth-service
ai-start                     # auto-detects project name from directory
# ... work ...
obs-write

# Switch to learning
ai-self
cd ~/learning/rust
ai-start
# ... learn ...
obs-write
```

### 10.6 Flow: Weekly review

```bash
# See recent sessions across a project
obs-search --recent 7

# Search for all decisions this week
obs-search --decisions

# Cross-search to find patterns
obs-search --cross "performance"
obs-search --cross "security"

# Review in Obsidian graph view for visual connections
```

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `OBSIDIAN_VAULT` | Yes | — | Path to your single Obsidian vault |
| `AI_MODE` | No | `"work"` | Current mode: `"work"` or `"self"` |
| `AI_PROJECT` | No | `basename $PWD` | Override project/topic name |
| `AI_HOME` | No | `$HOME/.ai` | Location of ai-dev config |
| `AI_TOOL` | No | auto-detect | Force specific AI tool for shell commands |
| `OBS_VAULT_NAME` | No | — | Vault name for CLI targeting (multi-vault setups) |
| `EDITOR` | No | `vi` | Editor for interactive obs-write |
