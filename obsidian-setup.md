# Obsidian Vault Setup

Two-mode system: **Work** (project-based) and **Self** (topic-based).

---

## Shell Environment

Add to `~/.zshrc`:

```bash
# Mode selection
export AI_MODE="work"  # or "self"

# Vault paths
export OBSIDIAN_WORK_VAULT="$HOME/Documents/WorkVault"
export OBSIDIAN_SELF_VAULT="$HOME/Documents/SelfVault"

# Helpers
alias ai-work='export AI_MODE="work"'
alias ai-self='export AI_MODE="self"'
alias ai-mode='source ~/.ai/bin/ai-mode'

# API (same for both vaults)
export OBSIDIAN_API_KEY="your-key-from-plugin"
export OBSIDIAN_REST_PORT="27123"
```

Initialize vaults:
```bash
ai-mode --init
```

---

## Work Vault Structure

```
WorkVault/
└── Projects/
    └── project-a/
        ├── README.md           # project context (stack, conventions)
        ├── Session.md          # current tasks for AI
        ├── Sessions/
        │   └── 2026-03-30.md   # daily logs
        ├── Decisions/
        │   └── log.md          # project decisions
        ├── Skills/
        │   └── log.md          # project-specific skills
        └── Errors/
            └── log.md          # bug fixes
```

---

## Self Vault Structure

```
SelfVault/
└── Topics/
    └── python/
        ├── learnings.md        # consolidated topic notes
        ├── Session.md          # current tasks for AI
        ├── Sessions/
        │   └── 2026-03-30.md   # dated entries
        ├── Decisions/
        ├── Skills/
        └── Errors/
```

---

## Session.md Format

This is your **task file** — what you want the AI to work on:

```markdown
# Session — 2026-03-30

## Tasks
- [ ] Implement user authentication
- [ ] Add rate limiting
- [ ] Review PR #42

## Notes
- Use PostgreSQL for user data
- Follow existing service pattern
```

---

## Quick Reference

```bash
# Switch mode
ai-mode work     # or ai-work
ai-mode self     # or ai-self

# Read context
obs-read              # project/topic + session
obs-read --session    # session tasks only
obs-read --all        # full context + decisions

# Write notes
obs-write                  # interactive session log
obs-write --init           # create project/topic structure
obs-write --session "task" # add to Session.md
obs-write --decision "text" # log decision
obs-write --fix "text"      # log bug fix

# Search
obs-search "query"       # full-text
obs-search --project     # project context
obs-search --session     # current tasks
obs-search --recent 5    # recent sessions

# Build AI context (with Obsidian)
build-context --obsidian --adapter all
```

---

## Flow: AI Session

1. **Start session**
   ```bash
   ai-work  # or ai-self
   cd ~/work/project-a
   obs-read --session  # see what to work on
   ```

2. **AI works**
   - Reads `Session.md` for tasks
   - Reads `README.md` for context
   - Consults `Decisions/`, `Skills/`, `Errors/` as needed

3. **Log progress**
   ```bash
   obs-write --session "Completed auth, started rate limiting"
   obs-write --decision "Using JWT tokens instead of sessions"
   ```

4. **End session**
   ```bash
   obs-write  # full session log
   ```

---

## build-context with Obsidian

To include Obsidian context in AI prompts:

```bash
# Pull current project context + decisions from Obsidian
build-context --obsidian --adapter all
```

This adds to the AI context:
- Project/topic README.md
- Current Session.md (tasks)
- Recent decisions

---

## Notes

- **Mode auto-detection**: Based on `AI_MODE` env var
- **Project name**: Defaults to current directory basename, or `AI_PROJECT`
- **Topic name** (Self mode): Auto-detected from directory, e.g., `~/learning/python/` → topic=`python`
- **Decisions/Skills/Errors**: Stored per-project (Work) or per-topic (Self), NOT duplicated in `~/.ai/context/`
