# ai-dev

Tool-agnostic AI coding setup. Works with Claude Code, Codex CLI, OpenCode, or any AI tool.

No slash commands. No `.claude/` lock-in. Plain markdown and shell scripts.

---

## What this is

A global config (`~/.ai/`) that gives any AI coding tool:
- Your identity, rules, and coding conventions (Python-focused, extends easily)
- Obsidian as a second brain: project context, decisions, and session logs
- Shell scripts for code review, pre-ship checks, and debugging
- A one-command project scaffold: `ai-init`

The source of truth is `~/.ai/context/`. A `build-context` script assembles it into a single prompt, then adapters write tool-specific files (`~/.claude/CLAUDE.md`, `~/.codex/instructions.md`, etc.) from that one source.

---

## Install

```bash
git clone https://github.com/yourname/ai-dev ~/.ai-dev
cd ~/.ai-dev
./bin/ai-init --global-install
source ~/.zshrc   # or ~/.bashrc
```

Then configure Obsidian (see [obsidian-setup.md](obsidian-setup.md)):
```bash
# Add to ~/.zshrc:
export OBSIDIAN_VAULT_PATH="$HOME/Documents/MyVault"
export OBSIDIAN_API_KEY="your-key-from-obsidian-local-rest-api-plugin"
```

Then rebuild context and push to your installed tools:
```bash
build-context --adapter all
```

---

## Scaffold a new project

```bash
cd my-project
ai-init                    # uses directory name as project name
# or
ai-init my-project-name
```

Creates:
```
AGENTS.md                  # universal AI context — edit this
CLAUDE.md -> AGENTS.md     # symlink (only if claude is installed)
.ai/rules/conventions.md   # project-specific rule overrides
.ai/skills/stack.md        # project stack notes
.gitignore                 # Python defaults added
```

And in Obsidian: `Projects/my-project.md` with a starter template.

---

## Daily workflow

```bash
# Session start
obs-read                   # load Obsidian context into terminal, feed to your AI tool

# During work
ai-review                  # linters + AI code review (before committing)
ai-debug --test path/to/test.py   # systematic debug session
obs-write --decision "text"       # quick decision log
obs-write --fix "text"            # quick bug fix log

# Session end
obs-write                  # full session log (opens $EDITOR)
ai-ship                    # pre-ship checklist before PR
```

---

## Tool compatibility

| Feature | Claude Code | Codex CLI | OpenCode |
|---|---|---|---|
| Project context | `AGENTS.md` (symlinked from CLAUDE.md) | `AGENTS.md` native | `AGENTS.md` native |
| Global rules | `~/.claude/CLAUDE.md` | `~/.codex/instructions.md` | `~/.config/opencode/system.md` |
| Shell commands | `ai-review`, `ai-ship`, etc. | same | same |
| Obsidian | `obs-read`, `obs-write` | same | same |

All shell commands work regardless of which AI tool you use. The AI tool is only invoked for the reasoning step — mechanical work (linting, testing, git) is always pure bash.

Set `AI_TOOL` to override auto-detection:
```bash
AI_TOOL=opencode ai-review
AI_TOOL=codex    ai-debug --test tests/test_auth.py
AI_TOOL=none     ai-ship    # run checklist without AI, print context for manual paste
```

---

## Structure

```
~/.ai/
├── context/
│   ├── identity.md          # who the AI is and how it works
│   ├── rules/
│   │   ├── general.md       # planning, naming, git, testing
│   │   ├── python.md        # types, imports, tooling, forbidden patterns
│   │   └── security.md      # injection, secrets, input validation
│   ├── skills/
│   │   ├── python.md        # project layout, Pydantic, service pattern
│   │   ├── tdd.md           # pytest patterns, fixtures, what to test
│   │   └── debugging.md     # protocol, tools, common Python bugs
│   └── agents/
│       ├── planner.md       # plan before code
│       └── reviewer.md      # review after code
├── bin/
│   ├── ai-init              # scaffold a project
│   ├── ai-review            # linters + AI review
│   ├── ai-ship              # pre-ship checklist
│   ├── ai-debug             # debug protocol
│   ├── build-context        # assemble context files into one prompt
│   ├── obs-read             # read Obsidian context
│   ├── obs-write            # write session notes to Obsidian
│   ├── obs-search           # search Obsidian vault
│   └── adapters/
│       ├── adapter-claude-code.sh
│       ├── adapter-codex.sh
│       └── adapter-opencode.sh
├── prompts/
│   └── global.md            # assembled output of build-context
└── template/
    └── AGENTS.md            # project scaffold template
```

Per-project (created by `ai-init`):
```
project/
├── AGENTS.md               # edit this — project context for all AI tools
├── CLAUDE.md -> AGENTS.md  # symlink for Claude Code
└── .ai/
    ├── rules/
    │   └── conventions.md  # project-specific rule overrides
    └── skills/
        └── stack.md        # chosen libraries, patterns, setup notes
```

---

## Editing global context

All source files are in `~/.ai/context/`. Edit them, then rebuild:

```bash
# Edit a rule
$EDITOR ~/.ai/context/rules/python.md

# Rebuild and push to all tools
build-context --adapter all
```

---

## Obsidian integration

Two modes, automatic fallback:
- **REST API** (primary): Obsidian must be open with the Local REST API plugin running.
- **Filesystem** (fallback): reads/writes `.md` files directly if Obsidian is closed.

See [obsidian-setup.md](obsidian-setup.md) for full setup instructions.

---

## Extending

**Add a new rule**: create `~/.ai/context/rules/myteam.md`, run `build-context --adapter all`.

**Add a new skill**: create `~/.ai/context/skills/fastapi.md`, same rebuild.

**Add a new AI tool**: copy an adapter from `bin/adapters/`, adjust the output path,
register it in `build-context`'s `--adapter all` case.

**Add Python stack skills**: edit `~/.ai/context/skills/python.md` with your
preferred frameworks (FastAPI, Django, Click, etc.).
