# ai-dev Usage Guide

Complete reference for the ai-dev system — a git-driven workflow for AI-assisted development with Obsidian integration.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [First-Time Setup](#2-first-time-setup)
3. [Deployment Workflow](#3-deployment-workflow)
4. [Mode Switching](#4-mode-switching)
5. [Session Lifecycle](#5-session-lifecycle)
6. [Work Mode — Project Development](#6-work-mode--project-development)
7. [Self Mode — Learning & Exploration](#7-self-mode--learning--exploration)
8. [Cross-Linking Work and Self](#8-cross-linking-work-and-self)
9. [Command Reference](#9-command-reference)
10. [Context System Deep Dive](#10-context-system-deep-dive)
11. [Multi-Agent Workflow](#11-multi-agent-workflow)
12. [Obsidian Vault Management](#12-obsidian-vault-management)
13. [Real-World Flows](#13-real-world-flows)

---

## 1. Architecture Overview

ai-dev uses a **source → runtime** deployment model:

```
~/.ai-dev (source/git repo)           ~/.ai (runtime)
├── modes/                     ──→    ├── modes/
│   ├── work/                         │   ├── work/
│   │   ├── identity.md               │   │   ├── identity.md
│   │   ├── tools.md                  │   │   ├── tools.md
│   │   └── rules/                    │   │   └── rules/
│   └── self/                         │   └── self/
│       ├── identity.md               │       └── ...
│       ├── tools.md                  ├── shared/
│       └── rules/                    │   ├── agents/
├── shared/                           │   ├── skills/
│   ├── agents/                       │   └── mcp-servers.json
│   ├── skills/                       ├── bin/
│   └── mcp-servers.json              ├── template/
├── bin/                              ├── prompts/
├── template/                         │   └── global.md
├── .githooks/                        ├── context → modes/work (symlink)
│   └── post-commit                   └── .deployed
└── specs/
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Source** (`~/.ai-dev`) | Git repository. Edit files here. |
| **Runtime** (`~/.ai`) | Deployed files. AI tools read from here. |
| **Deploy** | `ai-deploy` rsyncs source → runtime, then runs `build-context` |
| **Auto-deploy** | Git post-commit hook triggers deploy automatically |
| **Modes** | `work` and `self` — different identities, tools, rules |
| **Shared** | Agents, skills, MCP servers — same across modes |

### Design Principles

1. **Git is the source of truth** — All config lives in `~/.ai-dev`, version controlled
2. **Instant mode switching** — Just a symlink change, no rebuild needed
3. **Auto-deploy on commit** — Push changes by committing; hook handles the rest
4. **Separation of concerns** — Mode-specific vs shared content clearly organized

---

## 2. First-Time Setup

### 2.1 Clone and deploy

```bash
# Clone the repository
git clone https://github.com/yourname/ai-dev ~/.ai-dev

# Initial deploy to runtime
~/.ai-dev/bin/ai-deploy

# Install the git hook for auto-deploy
~/.ai-dev/bin/ai-deploy --install-hook

# Add to PATH (in ~/.zshrc)
export PATH="$HOME/.ai/bin:$PATH"
source ~/.zshrc
```

### 2.2 Configure your Obsidian vault

Edit `~/.zshrc`:

```bash
export OBSIDIAN_VAULT="$HOME/Documents/Brain"
export AI_MODE="work"  # or "self"
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
    ├── Agents/
    └── AgentConfig/     # optional: Obsidian as source of truth
```

### 2.3 Verify installation

```bash
# Check deployment status
cat ~/.ai/.deployed

# Check mode status
ai-mode

# Test mode switching
ai-mode work
ai-mode self
ai-mode work
```

### 2.4 Transport detection

ai-dev automatically detects what's available:

| Transport | Detected when | Features |
|-----------|---------------|----------|
| **Obsidian CLI** | `obsidian` command exists | Full: search, tasks, daily notes, properties, backlinks |
| **Filesystem** | `OBSIDIAN_VAULT` directory exists | Basic: read, write, append, grep-based search |

You don't configure this. `obs-lib` handles it transparently.

---

## 3. Deployment Workflow

### 3.1 How deployment works

When you run `ai-deploy`:

1. **Rsync** copies `bin/`, `modes/`, `shared/`, `template/` to `~/.ai/`
2. **Symlink** ensures `~/.ai/context` points to current mode
3. **build-context** assembles prompts and pushes to AI tools
4. **Metadata** written to `~/.ai/.deployed`

### 3.2 Manual deploy

```bash
# Full deploy
ai-deploy

# Dry run (see what would change)
ai-deploy --dry-run

# Quiet mode (for scripts)
ai-deploy --quiet
```

### 3.3 Auto-deploy via git hook

After installing the hook (`ai-deploy --install-hook`), every commit triggers deployment:

```bash
cd ~/.ai-dev

# Edit a file
vim modes/work/rules/python.md

# Commit — auto-deploy runs
git add -A
git commit -m "feat: add new Python rule"

# Verify
cat ~/.ai/.deployed
# Shows: commit: abc1234, timestamp: 2026-04-07 17:30:00
```

### 3.4 What gets deployed

| Source | Destination | Notes |
|--------|-------------|-------|
| `bin/` | `~/.ai/bin/` | All scripts |
| `modes/` | `~/.ai/modes/` | Work and self mode configs |
| `shared/` | `~/.ai/shared/` | Agents, skills, MCP servers |
| `template/` | `~/.ai/template/` | Project scaffolding templates |

### 3.5 What's excluded from deploy

- `.git/` — Git internals
- `.githooks/` — Hooks stay in source only
- `specs/` — Design specs and plans
- `prompts/` — Generated at deploy time
- `.env` — Preserved in runtime
- `*.pdf`, `.DS_Store` — Non-essential files

### 3.6 Editing workflow

```bash
# 1. Edit in source
cd ~/.ai-dev
vim shared/skills/python.md

# 2. Test locally (optional)
ai-deploy --dry-run

# 3. Commit to deploy
git add -A
git commit -m "docs: improve Python skill"

# Done! Changes are now in ~/.ai and pushed to all AI tools
```

---

## 4. Mode Switching

### 4.1 Two modes

| Mode | Purpose | Identity | Tools |
|------|---------|----------|-------|
| **work** | Shipping code at company | Senior engineer, team-oriented | GitLab (`glab`), company conventions |
| **self** | Personal learning & projects | Curious explorer | GitHub (`gh`), experimentation |

### 4.2 Switch modes

```bash
# Switch to work mode
ai-mode work

# Switch to self mode  
ai-mode self

# Check current mode
ai-mode
```

Mode switching is **instant** — it just updates the `~/.ai/context` symlink:

```
~/.ai/context → ~/.ai/modes/work   (work mode)
~/.ai/context → ~/.ai/modes/self   (self mode)
```

### 4.3 Mode-specific files

Each mode has its own:

```
modes/work/
├── identity.md    # Who you are in this mode
├── tools.md       # Git remote, CLI tools
└── rules/
    ├── general.md
    ├── python.md
    └── security.md

modes/self/
├── identity.md
├── tools.md
└── rules/
    └── ...
```

### 4.4 Shared across modes

These files are the same regardless of mode:

```
shared/
├── agents/
│   ├── planner.md
│   └── reviewer.md
├── skills/
│   ├── debugging.md
│   ├── git-worktrees.md
│   ├── python.md
│   ├── security-audit.md
│   └── tdd.md
└── mcp-servers.json
```

### 4.5 Quick aliases

Add to `~/.zshrc`:

```bash
alias ai-work='ai-mode work'
alias ai-self='ai-mode self'
```

---

## 5. Session Lifecycle

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
│     → obs-write --skill "..."               │
├─────────────────────────────────────────────┤
│  4. Review before shipping                  │
│     → ai-review                             │
│     → ai-ship                               │
├─────────────────────────────────────────────┤
│  5. End session                             │
│     → obs-write  (full session log)         │
└─────────────────────────────────────────────┘
```

### 5.1 Starting a session

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

### 5.2 During a session

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

### 5.3 Code review and shipping

```bash
# Before committing — run linters + AI review
ai-review

# Lint only (no AI)
ai-review --lint-only

# Before PR — full pre-ship checklist
ai-ship
```

### 5.4 Ending a session

```bash
obs-write
```

Opens your `$EDITOR` with a pre-filled template. Fill it in, save, close.

---

## 6. Work Mode — Project Development

Work mode is for shipping code. Each project gets its own vault folder.

### 6.1 Scaffold a new project

```bash
ai-work                       # ensure work mode
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

### 6.2 Edit AGENTS.md

This is your highest-leverage file. Keep it under 200 lines. Include:

- Stack and versions
- Run/test/lint commands
- Architecture decisions (the "why")
- Conventions that differ from defaults
- Known gotchas

### 6.3 Edit Session.md

Session.md is your **task handoff** between you and the AI:

```markdown
# Session — 2026-04-07

## Tasks
- [ ] Implement rate limiting middleware
- [ ] Add Redis cache layer for /users endpoint
- [ ] Fix flaky test in test_auth.py

## Notes
- Rate limiting: use sliding window, not fixed window
- Redis: connect via connection pool, not per-request
```

---

## 7. Self Mode — Learning & Exploration

Self mode is for building knowledge. Each topic gets its own vault folder.

### 7.1 Create a new topic

```bash
ai-self                       # switch to self mode
cd ~/learning/rust
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

### 7.2 Self mode tasks are exploratory

```markdown
# Session — 2026-04-07

## Tasks
- [ ] Understand ownership and borrowing model
- [ ] Build a CLI tool with clap
- [ ] Compare error handling: Result vs anyhow vs thiserror

## Resources
- The Rust Book ch. 4
- Jon Gjengset's YouTube channel
```

---

## 8. Cross-Linking Work and Self

This is the killer feature of the single-vault design.

### 8.1 Link a project to a topic

```bash
# While in work mode, working on my-api
obs-write --link "Self/Topics/python"
obs-write --link "Self/Topics/security"
```

### 8.2 Discover cross-references

```bash
# Search across both Work and Self for a keyword
obs-search --cross "authentication"
```

### 8.3 Use Obsidian's graph view

Because everything is in one vault, you can visualize connections between projects and learning topics.

---

## 9. Command Reference

### Deployment

| Command | Description |
|---------|-------------|
| `ai-deploy` | Deploy source → runtime, rebuild prompts |
| `ai-deploy --dry-run` | Show what would be synced |
| `ai-deploy --quiet` | Suppress output (for git hook) |
| `ai-deploy --install-hook` | Configure git to use .githooks/ |

### Mode switching

| Command | Description |
|---------|-------------|
| `ai-mode` | Show current status (mode, vault, transport, tasks) |
| `ai-mode work` | Switch to work mode |
| `ai-mode self` | Switch to self mode |
| `ai-mode --init` | Create vault directory structure |
| `ai-mode --init-modes` | Create mode directories in ~/.ai/modes/ |
| `ai-mode --list` | List all projects and topics |

### Session management

| Command | Description |
|---------|-------------|
| `ai-start` | Begin session — shows mode, tasks, decisions, status |
| `ai-start work` | Switch to work mode and start |
| `ai-start self` | Switch to self mode and start |
| `ai-start --rebuild` | Start + rebuild context for all AI tools |

### Reading from Obsidian

| Command | Description |
|---------|-------------|
| `obs-read` | Read project/topic context |
| `obs-read --session` | Read Session.md tasks only |
| `obs-read --all` | Full context: main + session + decisions + errors + skills |

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
| `ai-review` | Linters + AI review |
| `ai-review --lint-only` | Mechanical checks only |
| `ai-ship` | Full pre-ship checklist |

### Debugging

| Command | Description |
|---------|-------------|
| `ai-debug` | Interactive debug protocol |
| `ai-debug --test path/to/test.py` | Run specific test in debug mode |
| `ai-debug --error "message"` | Analyze a specific error |

### Context building

| Command | Description |
|---------|-------------|
| `build-context` | Assemble prompts to ~/.ai/prompts/global.md |
| `build-context --obsidian` | Include current Obsidian project context |
| `build-context --adapter all` | Push to all detected AI tools |
| `build-context --adapter claude-code` | Push to Claude Code only |
| `build-context --stdout` | Print assembled context to stdout |

### MCP servers

| Command | Description |
|---------|-------------|
| `ai-mcp list` | List all registered MCP servers |
| `ai-mcp enable <name>` | Enable an MCP server |
| `ai-mcp disable <name>` | Disable an MCP server |
| `ai-mcp distribute all` | Push MCP config to all AI tools |

### Project scaffolding

| Command | Description |
|---------|-------------|
| `ai-init` | Scaffold current directory |
| `ai-init my-project` | Scaffold with explicit name |

---

## 10. Context System Deep Dive

### 10.1 Context layering

Context is assembled in layers, from broadest to most specific:

```
Layer 1: Identity       (~/.ai/modes/{mode}/identity.md)
   ↓
Layer 2: Tools          (~/.ai/modes/{mode}/tools.md)
   ↓
Layer 3: Mode rules     (~/.ai/modes/{mode}/rules/*.md)
   ↓
Layer 4: Shared skills  (~/.ai/shared/skills/*.md)
   ↓
Layer 5: Shared agents  (~/.ai/shared/agents/*.md)
   ↓
Layer 6: MCP servers    (~/.ai/shared/mcp-servers.json)
   ↓
Layer 7: Project rules  (.ai/rules/conventions.md)
   ↓
Layer 8: AGENTS.md      (project root — AI reads directly)
   ↓
Layer 9: Obsidian       (Session.md, Decisions — pulled by obs-read)
```

Layers 1-6 are assembled by `build-context` into `~/.ai/prompts/global.md`, then pushed to each tool's global config via adapters.

Layers 7-9 are read directly by the AI tool or pulled on-demand.

### 10.2 Editing global context

**Edit in source, commit to deploy:**

```bash
cd ~/.ai-dev

# Edit a mode-specific rule
vim modes/work/rules/python.md

# Edit a shared skill
vim shared/skills/debugging.md

# Add a new agent
vim shared/agents/architect.md

# Commit to deploy
git add -A
git commit -m "feat: add architect agent"
```

### 10.3 Context files reference

| File | Location | Purpose |
|------|----------|---------|
| `identity.md` | `modes/{mode}/` | Who the AI is in this mode |
| `tools.md` | `modes/{mode}/` | Git remote, CLI tools for this mode |
| `rules/*.md` | `modes/{mode}/` | Mode-specific coding rules |
| `agents/*.md` | `shared/` | Agent personas (planner, reviewer) |
| `skills/*.md` | `shared/` | Skill patterns (debugging, TDD) |
| `mcp-servers.json` | `shared/` | MCP server registry |

### 10.4 Where context goes

| Destination | Generated by | Content |
|-------------|--------------|---------|
| `~/.ai/prompts/global.md` | `build-context` | Full assembled context |
| `~/.claude/CLAUDE.md` | adapter-claude-code | Claude Code global config |
| `~/.codex/instructions.md` | adapter-codex | Codex CLI global config |
| `~/.config/opencode/system.md` | adapter-opencode | OpenCode global config |

---

## 11. Multi-Agent Workflow

### 11.1 Supported AI tools

ai-dev works with multiple AI coding assistants:

| Tool | Project context | Global context |
|------|-----------------|----------------|
| Claude Code | `CLAUDE.md` → `AGENTS.md` | `~/.claude/CLAUDE.md` |
| Codex CLI | `AGENTS.md` (native) | `~/.codex/instructions.md` |
| OpenCode | `AGENTS.md` (native) | `~/.config/opencode/system.md` |

### 11.2 Using different tools

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

### 11.3 Rebuilding for specific tools

```bash
# Rebuild for all tools
build-context --adapter all

# Rebuild for specific tool
build-context --adapter claude-code
```

### 11.4 MCP servers

MCP servers provide additional capabilities to AI tools:

```bash
# List available servers
ai-mcp list

# Enable a server
ai-mcp enable serena

# Distribute to all tools
ai-mcp distribute all
```

Registry location: `~/.ai/shared/mcp-servers.json`

---

## 12. Obsidian Vault Management

### 12.1 Vault structure

```
Brain/                              # OBSIDIAN_VAULT
├── Work/
│   └── Projects/
│       ├── my-api/
│       │   ├── README.md           # project context
│       │   ├── Session.md          # current AI tasks
│       │   ├── Sessions/           # daily session logs
│       │   ├── Decisions/log.md    # all decisions
│       │   ├── Skills/log.md       # patterns learned
│       │   └── Errors/log.md       # bug fixes
│       └── auth-service/
├── Self/
│   └── Topics/
│       ├── python/
│       │   ├── learnings.md
│       │   ├── Session.md
│       │   └── ...
│       └── security/
└── Library/                        # cross-cutting
    ├── Decisions/
    ├── Skills/
    ├── Agents/
    └── AgentConfig/                # optional: Obsidian as source
        ├── work/
        └── self/
```

### 12.2 Log file format

All log files use timestamped entries:

```markdown
- [2026-04-07 14:30] Decision: Using JWT over sessions — stateless, scales better
- [2026-04-07 15:15] Decision: Redis for rate limiting — need distributed counters
```

### 12.3 Using Obsidian features

Because everything is in one vault:

- **Graph view**: See connections between projects and topics
- **Search**: `ctrl+shift+F` for vault-wide search
- **Tags**: Add `#active`, `#archived`, `#blocked` to notes
- **Properties**: Add frontmatter for structured metadata
- **Templates**: Use Obsidian templates for consistent note creation
- **Daily notes**: Integrates with daily notes plugin

---

## 13. Real-World Flows

### 13.1 Flow: Making a config change

```bash
# 1. Edit in source
cd ~/.ai-dev
vim modes/work/rules/python.md

# 2. Commit (auto-deploys)
git add -A
git commit -m "feat(rules): add async guidelines"

# 3. Verify
cat ~/.ai/.deployed
# Shows new commit hash

# 4. Check it's in your AI tool
head -50 ~/.claude/CLAUDE.md
```

### 13.2 Flow: Adding a new skill

```bash
# 1. Create skill file
cd ~/.ai-dev
cat > shared/skills/docker.md << 'EOF'
# Docker Skill

## Dockerfile best practices
- Use multi-stage builds
- Pin base image versions
- Run as non-root user

## Common commands
- `docker build -t name .`
- `docker run --rm -it name`
EOF

# 2. Commit
git add -A
git commit -m "feat(skills): add Docker skill"

# 3. Skill is now available to all AI tools
```

### 13.3 Flow: Daily coding session

```bash
# 1. Start
cd ~/code/invoice-api
ai-start

# 2. Work with AI
claude

# 3. Log decisions
obs-write --decision "Using Stripe webhooks, not polling"

# 4. Before PR
ai-review
ai-ship

# 5. End session
obs-write
```

### 13.4 Flow: Switching between modes

```bash
# Working on company project
ai-work
cd ~/code/company-api
ai-start
# ... work ...
obs-write

# Switch to personal learning
ai-self
cd ~/learning/rust
ai-start
# ... learn ...
obs-write

# Back to work
ai-work
```

### 13.5 Flow: Setting up on a new machine

```bash
# 1. Clone
git clone git@github.com:yourname/ai-dev ~/.ai-dev

# 2. Deploy
~/.ai-dev/bin/ai-deploy
~/.ai-dev/bin/ai-deploy --install-hook

# 3. Add to PATH
echo 'export PATH="$HOME/.ai/bin:$PATH"' >> ~/.zshrc
echo 'export OBSIDIAN_VAULT="$HOME/Documents/Brain"' >> ~/.zshrc
echo 'export AI_MODE="work"' >> ~/.zshrc
source ~/.zshrc

# 4. Verify
ai-mode
ai-deploy --dry-run
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OBSIDIAN_VAULT` | Yes | — | Path to your Obsidian vault |
| `AI_MODE` | No | `"work"` | Current mode: `"work"` or `"self"` |
| `AI_PROJECT` | No | `basename $PWD` | Override project/topic name |
| `AI_HOME` | No | `$HOME/.ai` | Location of runtime config |
| `AI_DEV` | No | `$HOME/.ai-dev` | Location of source repo |
| `AI_TOOL` | No | auto-detect | Force specific AI tool |
| `EDITOR` | No | `vi` | Editor for interactive obs-write |

---

## Troubleshooting

### Deploy not working

```bash
# Check source exists
ls ~/.ai-dev/modes ~/.ai-dev/shared

# Check for syntax errors
bash -n ~/.ai-dev/bin/ai-deploy

# Run with output
ai-deploy  # should show file counts
```

### Hook not running

```bash
# Verify hook is installed
git -C ~/.ai-dev config --get core.hooksPath
# Should show: .githooks

# Verify hook is executable
ls -la ~/.ai-dev/.githooks/post-commit
# Should show: -rwxr-xr-x
```

### Mode switch not taking effect

```bash
# Check symlink
ls -la ~/.ai/context
# Should show: context -> /path/to/.ai/modes/{mode}

# Rebuild manually
build-context --adapter all
```

### AI tool not seeing changes

```bash
# Force rebuild
ai-deploy

# Check specific tool config
cat ~/.claude/CLAUDE.md | head -20
cat ~/.codex/instructions.md | head -20
```
