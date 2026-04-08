---
name: generate-ticket
description: Create GitLab issues on the Amnil chatbot task board in Jira-style format with proper labels. Use when the user wants to create a ticket, task, bug, user story, or any issue on the team board.
allowed-tools: Bash, Read, Write, Edit
---

# Generate Ticket

Create GitLab issues on the Amnil task board following the company Time Log Guidelines.

**Task board repo:** `task-board/amnil-chatbot-research-development-taskboard-2025-2026`
**Issue log:** `~/.ai-dev/skills/Generate-ticket/issue-log.md`

---

## Workflow

1. Gather info from the user (or infer from context)
2. Draft the ticket — show it to the user before creating
3. Create with `glab issue create`
4. Log the issue number and URL
5. Add an initial work comment if the user requests it

---

## Step 1: Gather Info

Ask the user (or infer) for:

| Field | Options |
|---|---|
| **Type** | User Story / Task / Bug |
| **Title** | Short description |
| **Details** | What needs to be done and why |
| **Acceptance criteria** | List of done conditions |
| **Status label** | Default: `TO DO` |
| **Category label** | Required — see table below |
| **Platform label** | Optional — API / UI / CSS / Design |
| **Linked story** | Issue # (required for Tasks and Bugs) |

---

## Step 2: Title Format

Company convention — prefix by type:

```
User Story>>As a user I can search previous conversations
Task>>Implement conversation history search endpoint
Bug>>Search returns empty results for queries with special chars (of ticket #42)
```

Special characters allowed in tickets: `! @ # $ % ^ & * ( ) _ + - = [ ] { } | \ ; ' : " , . / < > ?`

---

## Step 3: Label Rules

**One status label** (default: `TO DO`):

| Label | Meaning |
|---|---|
| `OPEN` | Backlog — not yet scheduled |
| `TO DO` | Ready to start, not started |
| `IN PROGRESS` | Started, not complete |
| `READY FOR QA` | Dev done, needs testing |
| `IN QA` | QA actively testing |
| `NEED DEPLOYMENT` | QA passed, awaiting deploy |
| `CLOSED` | Deployed and done |

**One category label** (required — pick best fit):

| Label | Use for |
|---|---|
| `R&D` | Research with a specific goal/product in mind |
| `STANDUP` | Daily standup task |
| `LEGACY` | Old code study |
| `CODE_REVIEW` | Code review by mentor |
| `DISCUSSION` | Internal team discussion |
| `BUG` | Bug report |
| `DOCUMENTATION` | Writing or updating docs |
| `SPRINT_PLANNING` | Sprint management, task breakdown |
| `LOAD_BALANCE` | Traffic distribution work |
| `OPTIMIZATION` | Code improvement or performance |
| `RE_OPEN` | Resuming a closed task |
| `DEFERRED` | Postponed work |
| `DUPLICATE` | Duplicate of existing task |
| `USER STORY` | User story category label |
| `CLIENT_MEETING` | Client meeting tasks |

**One platform label** (optional — only if applicable):

| Label | Use for |
|---|---|
| `API` | API development |
| `UI` | UI development |
| `CSS` | CSS/styling work |
| `Design` | Design work |

---

## Step 4: Issue Body (Jira-style)

Use this template for the `--description` field:

```markdown
## Summary
<one-line summary>

## Details
<full description of what needs to be done and why>

## Acceptance Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] <criterion 3>

## Linked Story
#<issue-number>
(Omit this section for User Stories)

## Notes
<additional context, constraints, references — or omit if none>
```

---

## Step 5: Create the Issue

```bash
glab issue create \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026 \
  --title "Task>>Your task title here" \
  --description "$(cat <<'BODY'
## Summary
...

## Details
...

## Acceptance Criteria
- [ ] ...

## Linked Story
#42
BODY
)" \
  --label "TO DO,R&D,API"
```

Capture the output — it contains the issue URL with the issue number.

---

## Step 6: Log the Issue

After creation, append to `~/.ai-dev/skills/Generate-ticket/issue-log.md`.

If the file does not exist, create it with this header:

```markdown
# Issue Log

| # | Title | Type | Labels | Created | Context |
|---|---|---|---|---|---|
```

Then append the new row:

```markdown
| [#N](https://gitlab.amniltech.com/task-board/amnil-chatbot-research-development-taskboard-2025-2026/-/issues/N) | Task>>Title here | Task | TO DO, R&D | YYYY-MM-DD | Brief context |
```

---

## Step 7: Add Work Comment (optional)

If the user wants to log work on the issue:

```bash
glab issue note <issue-number> \
  --repo task-board/amnil-chatbot-research-development-taskboard-2025-2026 \
  --message "Work log: <description of work done or started>"
```

Use this to comment progress, blockers, or completion notes on any existing issue.

---

## Rules

- Title **must** start with `User Story>>`, `Task>>`, or `Bug>>`
- Each issue gets **exactly one** status label and **exactly one** category label
- Platform label is optional but if used, use only one
- Tasks and Bugs **must** reference their linked user story `#N` in the body
- Always show the drafted ticket to the user for confirmation before creating
- Always log the issue number and URL after successful creation
- Never `cd` into the repo — always use `--repo` flag with glab
