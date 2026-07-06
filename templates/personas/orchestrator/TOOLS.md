# Tools – Orchestrator Persona

## Required

- **`gh` CLI** (via Bash): Issue queries, label management, sub-issue creation, comments. Verify with `gh auth status`.

## Usage Patterns

All commands target the repo from config `github.repo` — pass `--repo <owner/name>` explicitly.

### Triage an issue

1. `gh issue list --assignee @me --state open --json number,title,labels` – fetch assigned issues (inbox scan)
2. `gh issue view N --json title,body,labels,comments` – read issue details
3. `gh issue edit N --add-label <persona>` – apply persona label
4. `gh issue comment N --body "..."` – post triage decision

### Decompose into sub-issues

1. `gh issue create --title "..." --body "..." --label <persona>` – create each child issue
2. `gh api repos/<owner>/<name>/issues/<parent>/sub_issues -f sub_issue_id=<id>` – attach it to the parent (ID from `gh api repos/<owner>/<name>/issues/<number> --jq .id`)
3. `gh issue comment <parent> --body "..."` – summarize decomposition on the parent

### Escalate to Board

1. `gh issue comment N --body "..."` – describe blocker, @-mention Board user (`github.board_user`)
2. `gh issue edit N --add-label agent-blocked` – apply the blocked label

## Not Used

The Orchestrator does not use repo tools (file read/write, git, etc.) — Bash is for `gh` commands only. It only reads the WoterClip config to understand available personas.
