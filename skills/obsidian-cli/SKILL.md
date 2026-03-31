---
name: obsidian-cli
description: Interact with Obsidian vaults using the Obsidian CLI to read, create, search, and manage notes, tasks, properties, and more. Use when the user asks to interact with their Obsidian vault, manage notes, search vault content, or perform vault operations from the command line.
allowed-tools: Bash, Read, Grep, Glob
---

# Obsidian CLI

Use the `obsidian` CLI to interact with a running Obsidian instance. Requires Obsidian to be open.

## Command reference

Run `obsidian help` to see all available commands. Full docs: https://help.obsidian.md/cli

## Syntax

Parameters take a value with `=`. Quote values with spaces:

```
obsidian create name="My Note" content="Hello world"
```

Flags are boolean switches with no value:

```
obsidian create name="My Note" silent overwrite
```

For multiline content use `\n` for newline and `\t` for tab.

## File targeting

- `file=<name>` — resolves like a wikilink (name only, no path or extension needed)
- `path=<path>` — exact path from vault root, e.g. `folder/note.md`
- Without either, the active file is used.

## Vault targeting

Use `vault=<name>` as the first parameter to target a specific vault:

```
obsidian vault="My Vault" search query="test"
```

## Common patterns

```bash
obsidian read file="My Note"
obsidian read path="Work/Projects/my-project/README.md"
obsidian create name="New Note" content="# Hello" silent
obsidian append path="Work/Projects/my-project/Session.md" content="- [ ] New task" silent
obsidian search query="search term" limit=10
obsidian daily:read
obsidian daily:append content="- [ ] New task"
obsidian property:set name="status" value="done" file="My Note"
obsidian tasks daily todo
obsidian tags sort=count counts
obsidian backlinks file="My Note"
```

Use `--copy` on any command to copy output to clipboard.
Use `silent` to prevent files from opening.

## ai-dev integration

This skill works with the ai-dev obs-* commands:

```bash
obs-read              # read project context via CLI
obs-write --init      # create vault structure
obs-search "query"    # search across vault
ai-start              # full session startup
```

The obs-* scripts use the Obsidian CLI as primary transport with filesystem fallback.

Vault structure:
```
Brain/
├── Work/Projects/<name>/    # project-based work
│   ├── README.md            # project context
│   ├── Session.md           # current AI tasks
│   ├── Sessions/            # daily logs
│   ├── Decisions/log.md     # decision log
│   ├── Skills/log.md        # learned patterns
│   └── Errors/log.md        # bug fixes
├── Self/Topics/<name>/      # learning & exploration
│   ├── learnings.md
│   ├── Session.md
│   └── ...
└── Library/                 # cross-cutting knowledge
```
