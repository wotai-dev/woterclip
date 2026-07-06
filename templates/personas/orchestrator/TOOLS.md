# Tools – Orchestrator Persona

## Required

- **`gh` CLI** (via Bash): Issue queries, label management, sub-issue creation, comments. Verify with `gh auth status`.

## Usage Patterns

All commands target the repo from config `github.repo` — pass `--repo <owner/name>` explicitly.

### Triage an issue

1. `gh issue list --repo <owner/name> --assignee @me --state open --json number,title,labels` – fetch assigned issues (inbox scan)
2. `gh issue view N --repo <owner/name> --json title,body,labels,comments` – read issue details
3. `gh issue edit N --repo <owner/name> --add-label <persona>` – apply persona label
4. `gh issue comment N --repo <owner/name> --body "..."` – post triage decision

### Decompose into sub-issues

Follow `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md` (canonical create/attach/verify procedure): create each child with `--assignee @me`, persona label, and `Parent: #N` body reference; attach by issue **ID** with `-F sub_issue_id=`; verify the attach; summarize the decomposition on the parent.

### Escalate to Board

1. `gh issue comment N --repo <owner/name> --body "..."` – describe blocker, @-mention Board user (`github.board_user`)
2. `gh issue edit N --repo <owner/name> --add-label agent-blocked` – apply the blocked label

## Not Used

The Orchestrator does not use repo tools (file read/write, git, etc.) — Bash is for `gh` commands only. It only reads the WoterClip config to understand available personas.
