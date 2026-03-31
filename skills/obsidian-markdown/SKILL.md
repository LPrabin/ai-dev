---
name: obsidian-markdown
description: Create and edit Obsidian Flavored Markdown with wikilinks, embeds, callouts, properties, and other Obsidian-specific syntax. Use when working with .md files in Obsidian, or when the user mentions wikilinks, callouts, frontmatter, tags, embeds, or Obsidian notes.
allowed-tools: Read, Write, Edit
---

# Obsidian Flavored Markdown

Create and edit valid Obsidian Flavored Markdown. Obsidian extends CommonMark and GFM with wikilinks, embeds, callouts, properties, and other syntax.

## Workflow: Creating an Obsidian Note

1. Add frontmatter with properties (title, tags, aliases) at the top
2. Write content using standard Markdown + Obsidian syntax below
3. Link related notes using `[[wikilinks]]` for internal vault connections
4. Embed content using `![[embed]]` syntax
5. Add callouts for highlighted information using `> [!type]` syntax

**Rule**: use `[[wikilinks]]` for vault-internal links, `[text](url)` for external URLs only.

## Internal Links (Wikilinks)

```
[[Note Name]]                    Link to note
[[Note Name|Display Text]]       Custom display text
[[Note Name#Heading]]            Link to heading
[[Note Name#^block-id]]          Link to block
[[#Heading in same note]]        Same-note heading link
```

## Cross-linking Work and Self

The ai-dev vault has Work/ and Self/ top-level folders. Cross-link freely:

```markdown
<!-- In Work/Projects/my-api/README.md -->
Related learning: [[Self/Topics/python/learnings]]

<!-- In Self/Topics/python/learnings.md -->
Applied in: [[Work/Projects/my-api/README]]
```

## Embeds

```
![[Note Name]]                   Embed full note
![[Note Name#Heading]]           Embed section
![[image.png]]                   Embed image
![[image.png|300]]               Image with width
```

## Callouts

```
> [!note]
> Basic callout.

> [!warning] Custom Title
> Callout with a custom title.

> [!faq]- Collapsed by default
> Foldable callout (- collapsed, + expanded).
```

Common types: note, tip, warning, info, example, quote, bug, danger, success, failure, question, abstract, todo.

## Properties (Frontmatter)

```yaml
---
title: My Note
date: 2024-01-15
tags:
  - project
  - active
aliases:
  - Alternative Name
---
```

## Tags

```
#tag                 Inline tag
#nested/tag          Nested tag with hierarchy
```

Tags can contain letters, numbers (not first character), underscores, hyphens, and forward slashes.

## Comments

```
This is visible %%but this is hidden%% text.
```

## ai-dev Note Conventions

When creating notes for ai-dev vault structure:

- **Session.md**: Use task lists (`- [ ]` / `- [x]`) for AI tasks
- **Decisions/log.md**: Prefix with date `- [YYYY-MM-DD HH:MM] Decision: ...`
- **Errors/log.md**: Prefix with date `- [YYYY-MM-DD HH:MM] Fix: ...`
- **Skills/log.md**: Prefix with date `- [YYYY-MM-DD HH:MM] Skill: ...`
- **Cross-link** between Work and Self whenever a project relates to a learning topic
