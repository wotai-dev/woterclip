# Status Mapping

Maps between GitHub issue state and WoterClip agent states. GitHub issues are only
**open** or **closed** — the working-state distinctions Linear carried natively are
carried by **status labels** (`backlog`, `todo`, `in-progress`, `in-review`).

## GitHub State → WoterClip Behavior

| GitHub state | WoterClip Behavior |
|-------------|-------------------|
| Open + `backlog` label | Ignored — not in the inbox |
| Open, no status label (or `todo`) | In the inbox, eligible for pickup (lower priority than `in-progress`) |
| Open + `in-progress` label | In the inbox, priority pickup (agent or human started work) |
| Open + `in-review` label | Ignored — human is reviewing, agent should not touch |
| Closed (completed) | Ignored — completed |
| Closed (not planned) | Ignored — canceled |

## WoterClip Outcomes → GitHub State Changes

| Heartbeat Outcome | GitHub Transition | Labels Changed |
|-------------------|------------------------|----------------|
| **Work completed** | Close the issue (`gh issue close N --comment ...`), or swap to `in-review` if a PR was opened | Remove `agent-working` |
| **Work in progress** | Stays open, ensure `in-progress` label | Keep `agent-working` |
| **Blocked** | Stays open | Remove `agent-working`, add `agent-blocked` |
| **Triaged by Orchestrator** | Stays open | Add persona label, remove `agent-working` |
| **Decomposed** | Parent stays open (todo-tier, below its children in pick order), sub-issues created open | Add persona labels to sub-issues, remove `agent-working` and `in-progress` from parent |

## Inbox Query

The heartbeat fetches open issues assigned to the authenticated user, then filters
client-side:

```bash
gh issue list --repo <owner/name> --assignee @me --state open \
  --json number,title,labels,createdAt --limit 100
```

### Sort Order

1. **Status label**: `in-progress` > no status label / `todo` (`backlog` and `in-review` filtered out)
2. **Priority label**: `priority:high` > no priority label > `priority:low`

GitHub has no native priority field — priority is the optional `priority:high` /
`priority:low` label pair. Unlabeled issues sit between the two.

### Filter Rules

- Skip issues without a persona label (unless Orchestrator is default)
- Skip `agent-blocked` issues unless new human comments exist since last agent comment
- Skip issues labeled `backlog` or `in-review`
- If `--persona <name>` flag is set, only match that persona's label

## Stale Detection

- `agent-working` label with no heartbeat comment in the last `stale_lock_hours` → stale lock
- Heartbeat cleans stale locks: removes `agent-working` (`gh issue edit N --remove-label agent-working`), posts a comment explaining the cleanup
