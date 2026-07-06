# Brainstorm: swap Linear for GitHub Issues (#1)

**Date:** 2026-07-06 · **Tier:** safety-critical · **Issue:** [#1](https://github.com/wotai-dev/woterclip/issues/1)

## Problem

WoterClip is Linear-backed: the heartbeat reads its inbox from Linear MCP tools, personas
route on Linear labels, reports post as Linear comments, and init scaffolds Linear label
groups. Swap the tracker backend to GitHub Issues so the plugin has no Linear (or MCP)
dependency at all.

## Core decision: `gh` CLI, not GitHub MCP

The plugin talks to GitHub through the **`gh` CLI**, not an MCP server.

- No MCP dependency: works in scheduled/headless heartbeats where interactively-authenticated
  MCP servers may be absent.
- `gh` is already the conventional GitHub driver in this repo's process (AGENTS.md).
- Tool validation becomes a simple availability check: `gh auth status` exits 0.

`required_tools` in persona configs changes meaning: from MCP tool prefixes to executable
checks (`gh`), verified via Bash.

## Concept mapping

| Linear concept | GitHub equivalent | Notes |
|---|---|---|
| `mcp__claude_ai_Linear__*` tools | `gh issue list/view/edit/comment/close`, `gh api` | No MCP |
| Team (`linear.team`) | Repo (`github.repo`, `owner/name`) | Derived from `gh repo view` at init |
| Board user display name (`linear.user_name`) | GitHub login (`github.board_user`) | `@login` mentions actually notify |
| Issue ref `WOT-XX` | `#N` | Repo-scoped number |
| States (Backlog/Todo/In Progress/In Review/Done/Canceled) | open/closed + status labels | See status model below |
| Priority (Urgent/High/Medium/Low) | `priority:high` / `priority:low` labels | 3-level: high > none > low |
| Sub-issues (`parentId`) | Native sub-issues via REST (`gh api .../issues/{n}/sub_issues`) | Create issue, then attach by issue **ID** |
| Label group ("WoterClip" parent) | Flat labels | GitHub has no label groups — `labels.group` dropped |
| Comments (`save_comment` / `list_comments`) | `gh issue comment` / `gh issue view --json comments` | Heartbeat counter still parsed from comments |

## Status model (the biggest semantic gap)

GitHub issues are only open/closed; Linear's six states collapse to labels:

| Linear state | GitHub representation | Heartbeat behavior |
|---|---|---|
| Backlog | open + `backlog` label | Ignored |
| Todo | open (no status label, or `todo`) | Eligible for pickup |
| In Progress | open + `in-progress` label | Priority pickup |
| In Review | open + `in-review` label | Ignored — human reviewing |
| Done | closed (completed) | Ignored |
| Canceled | closed (not planned) | Ignored |

Transitions: work completed → close the issue (or swap to `in-review` label when a PR was
opened); blocked → stays open, `agent-blocked` added. The `agent-working`/`agent-blocked`
mutual exclusion and stale-label cleanup carry over unchanged.

## Label operations

Linear's read-modify-write array dance is replaced by GitHub's **atomic** label operations:
`gh issue edit N --add-label X --remove-label Y`. This is strictly safer than the Linear
pattern (no full-set overwrite, no clobber risk) — the state machine keeps its rules
(mutual exclusion, one persona label per issue) but the write primitive improves.

## Config schema v2

```yaml
version: 2
github:
  board_user: "{{USER_NAME}}"   # GitHub login (@-mentions notify)
  repo: "{{REPO}}"              # owner/name
labels:
  working: "agent-working"
  blocked: "agent-blocked"      # group: dropped — no label groups on GitHub
```

- Placeholder change: `{{TEAM}}` → `{{REPO}}` (CLAUDE.md convention note updated).
- Migration v1→v2 in the init skill: `linear.user_name` → ask the user for their GitHub
  login (a Linear display name is not a GitHub login — do not guess), `linear.team` →
  discarded (repo derived from the checkout), `labels.group` → dropped, `version` → 2.

## Out of scope

- `docs/specs/*` — dated historical design records; left as-is.
- Migrating existing Linear issue *data* into GitHub.
- GitHub Projects board integration (plain Issues suffice; board support could be a follow-up).
- Plugin version bump to 0.2.0 — happens in a release PR per AGENTS.md, not this feature PR.
