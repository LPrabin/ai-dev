---
name: log-time
description: Log time spent on GitLab issues using /spend format. Use when the user wants to track time, log hours, or add time spent on a task.
allowed-tools: Bash, Read
---

# Log Time

Add time tracking comments to GitLab issues following the `/spend` format.

**Task board repo:** `task-board/amnil-chatbot-research-development-taskboard-2025-2026`

---

## Workflow

1. Identify the issue (ask if not clear)
2. Determine time spent (ask if not provided)
3. Write a descriptive comment explaining what was done
4. Add the comment with `/spend` directive

---

## Time Format

GitLab accepts these time units:
- `1h` — 1 hour
- `30m` — 30 minutes
- `1h30m` — 1 hour 30 minutes
- `1d` — 1 day (8 hours)
- `1w` — 1 week (5 days)

**Examples:**
- `/spend 2h` — 2 hours
- `/spend 1h30m` — 1 hour 30 minutes
- `/spend 45m` — 45 minutes

---

## Rules

### Character Safety (important)

- Time log comments must stay plain ASCII text.
- Do **not** use unicode arrow symbols (for example: `→`, `➡`, `⟶`) or emojis.
- If user-provided text contains emojis/arrows, rewrite to plain text before posting the note.
- Safe replacements:
  - `→`, `➡`, `⟶` -> `to`
  - `✅` -> `[done]`
  - `❌` -> `[blocked]`
  - `🔥` -> `[high-priority]`

### Maximum Time Per Comment

- **Maximum:** 4 hours per single `/spend` entry
- If the user reports more than 4 hours, **ask for clarification**:
  - "That's over 4 hours. Want me to split it across multiple entries?"
  - "Can you break down what was done in each chunk?"

### Comment Structure

Every time log comment **must** include:
1. **What was done** — specific tasks completed
2. **How it was done** — approach, tools, methods used
3. **The `/spend` directive** — at the end of the comment
4. **ASCII-only content** — no emoji and no unicode arrows

**Template:**
```
## Work Done
<description of what was accomplished>

## Approach
<how the work was done, tools used, decisions made>

/spend <time>
```

---

## Command

```bash
glab issue note <issue-number> \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026 \
  --message "$(cat <<'BODY'
## Work Done
<what was accomplished>

## Approach
<how it was done>

/spend 1h30m
BODY
)"
```

---

## Examples

### Example 1: Development work

```bash
glab issue note 42 \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026 \
  --message "$(cat <<'BODY'
## Work Done
Implemented the conversation history search endpoint with pagination support.

## Approach
- Added new `/api/conversations/search` endpoint in FastAPI
- Used Elasticsearch for full-text search with fuzzy matching
- Added unit tests for edge cases (empty results, special chars)

/spend 2h30m
BODY
)"
```

### Example 2: Research work

```bash
glab issue note 43 \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026 \
  --message "$(cat <<'BODY'
## Work Done
Researched vector database options for RAG implementation.

## Approach
- Compared Pinecone, Weaviate, and Qdrant on cost, performance, and ease of use
- Tested Qdrant locally with sample embeddings
- Documented findings in Obsidian for team review

/spend 3h
BODY
)"
```

### Example 3: Bug fix

```bash
glab issue note 44 \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026 \
  --message "$(cat <<'BODY'
## Work Done
Fixed issue where search returned empty results for queries containing special characters.

## Approach
- Traced the bug to unescaped regex characters in the search query
- Added input sanitization using `re.escape()` before passing to Elasticsearch
- Added test cases for all special characters mentioned in the ticket

/spend 1h
BODY
)"
```

---

## Gathering Info

If the user says "log time" without details, ask:

1. **Which issue?** — Issue number or let them describe it
2. **How much time?** — In hours/minutes
3. **What was done?** — Brief description of the work

If context is available (e.g., you just helped with a task), infer the details and confirm before logging.

---

## Viewing Time Spent

To check existing time on an issue:

```bash
glab issue view <issue-number> \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026
```

The time estimate and time spent will be shown in the issue details.
