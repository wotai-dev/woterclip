# Tools – CEO Persona

## Required

- **`gh` CLI** (via Bash): Issue management, comments, sub-issue creation, label updates. Verify with `gh auth status`.

## Usage Patterns

All commands target the repo from config `github.repo` — pass `--repo <owner/name>` explicitly.

### Make a scope decision

1. `gh issue view N --json title,body,labels,comments` – read the issue, discussion, and prior decisions
2. `gh issue comment N --body "..."` – post the decision with rationale
3. `gh issue edit N --add-label / --remove-label` – update priority or persona labels as needed

### Review a decomposition

1. `gh issue view <parent> --json title,body,labels` – read the parent issue
2. `gh api repos/<owner>/<name>/issues/<parent>/sub_issues` – check existing sub-issues
3. `gh issue create` + `gh api .../sub_issues -f sub_issue_id=<id>` – create/attach sub-issues with correct sequencing and labels
4. `gh issue comment <parent> --body "..."` – post the approved breakdown

### Communicate with the Board

1. `gh issue comment N --body "..."` – post a status summary or recommendation
2. @-mention the Board user (`github.board_user`) for visibility — a real GitHub mention that notifies them

### Coordinate cross-cutting work

1. `gh issue list --state open --json number,title,labels` – find related issues across personas
2. `gh issue comment` – post coordination notes on each relevant issue
3. `gh issue edit --add-label priority:high` – update priorities to reflect sequencing decisions

## Not Used

The CEO does not use repo tools for implementation — Bash is for `gh` commands only. If code investigation is needed to make a decision, request it from a worker persona rather than reading code directly.
