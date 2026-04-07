# Tools: Work Mode

## Git Workflow
- Remote: GitLab
- CLI: `glab`
- Branch naming: `feature/TICKET-123-description`, `fix/TICKET-456-description`
- MR workflow: Create MR, assign reviewers, wait for approval

## Issue Management
```bash
# Create issue
glab issue create --title "Title" --description "Description" --label "bug"

# List issues
glab issue list --assignee @me

# Create MR
glab mr create --fill --assignee @me
```

## CI/CD
- Pipeline must pass before merge
- Check pipeline: `glab ci status`
