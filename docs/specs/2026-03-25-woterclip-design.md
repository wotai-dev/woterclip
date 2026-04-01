# WoterClip Design Spec

**Date:** 2026-03-25
**Status:** Draft
**Author:** Alex Kim + Claude
**Linear:** [WOT-79](https://linear.app/wotai/issue/WOT-79/design-woterclip-agent-orchestration-system)

## Overview

WoterClip is a Claude Code plugin that provides Linear-backed agent orchestration with persona-based task routing. It replaces Paperclip's agent system by using Linear as the task store and Claude Code's `/schedule` as the heartbeat runner.

**Core idea:** A single Claude Code instance per repo wears different "hats" (personas) based on Linear issue labels. A CEO persona triages and decomposes work. Worker personas (backend, frontend, etc.) execute it. The human is the Board — the ultimate escalation target.

## Key Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Agent model | Single Claude instance, multiple personas | Simpler than multiple processes; personas switch via skill files |
| Persona storage | Separate directory per persona (SOUL.md, TOOLS.md, config.yaml) | Matches proven Paperclip pattern; rich enough for identity + tools |
| Issue → persona matching | Linear labels | Simple, visible in Linear UI, no custom fields needed |
| Issue assignment | Board user's account + persona labels as signal | No fake bot accounts; agent picks up your issues that have persona labels |
| Checkout locking | `agent-working` Linear label | Keeps agent state separate from workflow status |
| Blocked state | `agent-blocked` label + comment + @-mention Board | Clear escalation path; dedup prevents repeated blocked comments |
| Communication | Structured comment on every heartbeat | Full audit trail with commits, duration, sub-issues, carry-forward |
| Hierarchy | Board (human) → CEO persona → worker personas | CEO orchestrates, workers execute, Board decides |
| Escalation | Comment + @-mention Board user | CEO escalates to Board; workers escalate to CEO |
| Delegation | Large work → Linear sub-issues; small → internal tasks | Visible decomposition for large scope; lightweight for small scope |
| Distribution | Claude Code plugin (own repo + marketplace) | Install once, works everywhere; community gets updates automatically |

## 1. Plugin Structure

```
woterclip/
  package.json                # Plugin manifest (name: "woterclip")
  commands/
    heartbeat.md              # /heartbeat slash command
    woterclip-init.md         # /woterclip-init slash command
    woterclip-status.md       # /woterclip-status slash command
  skills/
    heartbeat/
      SKILL.md                # Core heartbeat procedure
    init/
      SKILL.md                # Initialize WoterClip in a repo
    persona-create/
      SKILL.md                # Create a new persona
    persona-list/
      SKILL.md                # List configured personas
    persona-import/
      SKILL.md                # Import from Paperclip agent configs
    status/
      SKILL.md                # Show agent status
    heartbeat-log/
      SKILL.md                # Summarize recent heartbeat activity
  agents/
    ceo.md                    # CEO orchestration subagent
  hooks/
    hooks.json                # Optional hooks
  references/
    label-conventions.md      # Standard Linear label names and meanings
    comment-format.md         # Structured comment templates
    status-mapping.md         # Linear states <-> WoterClip states
  templates/
    config.yaml               # Default config template
    personas/
      ceo/
        SOUL.md               # Default CEO persona template
        TOOLS.md
        config.yaml
  README.md
```

**Per-repo scaffolding** (created by `/woterclip-init`):

```
<target-repo>/
  .woterclip/
    config.yaml             # Repo-level config
    personas/
      ceo/
        SOUL.md
        TOOLS.md
        config.yaml
      backend/              # User-created personas
        SOUL.md
        TOOLS.md
        config.yaml
```

## 2. Heartbeat Procedure

The core loop that runs on every `/schedule` trigger or manual `/heartbeat` invocation.

### Steps

1. **Load Config** — Read `.woterclip/config.yaml`. Verify Linear MCP is available. Check for lockfile (`.woterclip/.heartbeat-lock`). If lockfile exists and is less than `stale_lock_hours` old, skip this heartbeat ("previous heartbeat still active"). If stale, delete lockfile and proceed. Create lockfile with current timestamp. Delete lockfile on exit (including error paths).

2. **Check Inbox** — Query Linear via `mcp__claude_ai_Linear__list_issues` with `assignee: "me"`. The Linear MCP only accepts a single `state` filter, so fetch all issues for the assignee (no state filter), then filter and sort client-side. Sort by status (in_progress > todo), then by Linear priority (critical > low). Skip issues without a persona label. Skip `agent-blocked` issues unless new comments exist since last agent comment. Detect stale `agent-working` labels (older than `stale_lock_hours`) by checking comment timestamps — clean up stale lock, post a comment about it.

3. **Pick Issue** — First in_progress with persona label, then first todo. Respect `max_issues_per_heartbeat` limit. If `--dry-run`, report what would be picked and exit. If `--persona <name>`, only pick issues matching that persona's label. No matches → exit heartbeat.

4. **Resolve Persona** — Match issue label → persona directory via config. No persona label → load CEO persona. Load SOUL.md and TOOLS.md into context. Apply runtime config (model, thinking effort, max turns) from persona's config.yaml.

5. **Validate Tools** — Check persona's `required_tools` are available. The `required_tools` field uses **prefix matching** — `mcp__claude_ai_Linear` matches any `mcp__claude_ai_Linear__*` tool. Missing tool prefix → block immediately with specific error message.

6. **Lock** — Read the issue's current labels via `mcp__claude_ai_Linear__get_issue`. Append `agent-working` to the existing labels array. Save via `mcp__claude_ai_Linear__save_issue` with the full label set. If `agent-working` is already present (same agent, previous beat), proceed without re-saving. **Note:** Label updates are a read-modify-write cycle. This is acceptable because WoterClip runs as a single instance per repo — no concurrent writers.

7. **Understand Context** — Read issue title, description, comments. Read parent issue if exists. Check for new comments since last heartbeat. Parse the heartbeat counter from the last WoterClip comment (look for `Heartbeat #N` pattern). If no previous comment, start at #1.

8. **Do Work** — Follow persona instructions (SOUL.md). Use repo tools. For large scope: create Linear sub-issues via `mcp__claude_ai_Linear__save_issue` with `team` from repo config, `parentId` set to the current issue, and appropriate persona labels. For small scope: use internal Claude Code tasks. If Linear MCP becomes unavailable mid-work, stop work, leave `agent-working` label in place (will be cleaned as stale lock on next heartbeat), and exit with error log.

9. **Report** — Post structured comment on Linear issue via `mcp__claude_ai_Linear__save_comment`. Include heartbeat counter (incremented from step 7). Store heartbeat metadata to `.woterclip/heartbeat-log.jsonl` (timestamp, issue ID, persona, duration, status, heartbeat number) for `/woterclip-status --history`.

10. **Update State** — Read the issue's current labels, then modify:
    - **Done** → remove `agent-working` from labels array, save. Update issue state to the appropriate Linear status.
    - **Blocked** → remove `agent-working`, add `agent-blocked` to labels array, save. Post comment with blocker details. Include Board user's Linear display name in comment text (e.g., "**@Alex Kim** — please review"). Note: Linear comment @-mentions use display names in markdown; Linear may or may not trigger notifications for plain text mentions. For guaranteed notification, also consider assigning the issue back to the Board user or changing issue state.
    - **More work needed** → keep `agent-working`, comment with progress.

11. **Next Issue or Exit** — If under `max_issues_per_heartbeat`, go to step 2. Otherwise delete lockfile and exit.

### Blocked-Task Dedup

Before working an `agent-blocked` issue, read its comment thread. If the agent's last comment was a blocked update AND no new comments from humans exist since, skip the issue entirely. Only re-engage when new context appears.

### Heartbeat Counter Persistence

The heartbeat counter is **derived from Linear comments**, not stored locally. On each heartbeat, parse the last WoterClip comment on the issue for `Heartbeat #N` and increment. If comments are deleted, the counter resets — this is acceptable as the counter is informational, not functional.

### Overlap Prevention

A lockfile at `.woterclip/.heartbeat-lock` prevents concurrent heartbeats. Created at step 1, deleted at step 11 (and in error handlers). Stale lockfiles (older than `stale_lock_hours`) are auto-cleaned.

### Differences from Paperclip

| Paperclip | WoterClip |
|-----------|-----------|
| `POST /checkout` API | `agent-working` Linear label |
| `GET /api/agents/me` | Read config.yaml + persona files |
| `PATCH /api/issues/:id` | Linear MCP `save_issue` |
| `POST /api/issues/:id/comments` | Linear MCP `save_comment` |
| `409 Conflict` on checkout | Check if `agent-working` label present |
| Budget tracking via API | No budget tracking (relies on /schedule limits) |
| Chain of command from API | Defined in persona config (escalates_to field) |
| Separate agent processes | Single Claude instance, persona switching |
| Custom memory system (PARA) | Claude Code built-in memory |

## 3. Persona Structure

Each persona gets its own directory, inspired by Paperclip's agent structure.

### File Mapping from Paperclip

| Paperclip File | WoterClip Equivalent | Purpose |
|----------------|---------------------|---------|
| `SOUL.md` | `SOUL.md` | Identity, posture, voice, decision framework |
| `AGENTS.md` | `config.yaml` (repo-level) | Agent config, home refs, safety rules |
| `HEARTBEAT.md` | Plugin heartbeat skill (generic) | Execution procedure |
| `TOOLS.md` | `TOOLS.md` | Available tools and integrations |
| `life/`, `memory/` | Not needed | Claude Code has built-in memory |

### Persona config.yaml

```yaml
name: Backend Engineer
role: engineer
label: backend
escalates_to: ceo
required_tools:                # Prefix matching: "mcp__neon" matches any mcp__neon__* tool
  - mcp__claude_ai_Linear      # Required for all personas
  - mcp__neon                   # Neon database tools

runtime:
  model: opus                  # opus | sonnet | haiku
  thinking_effort: high        # high | medium | low
  max_turns: 300
  enable_chrome: false
  timeout: 0                   # 0 = no timeout
  extra_args: []               # e.g., ["--verbose"]
```

#### Runtime Config Per Persona

Different personas have different complexity needs. The heartbeat passes these as flags when dispatching work.

| Persona | Model | Thinking | Max Turns | Reasoning |
|---------|-------|----------|-----------|-----------|
| CEO | sonnet | medium | 100 | Triage/routing is lightweight — don't burn Opus tokens |
| Backend | opus | high | 300 | Deep implementation, complex code changes |
| Frontend | sonnet | medium | 200 | Component work, moderate complexity |
| Research | sonnet | medium | 150 | Analysis, not code — Chrome often enabled |
| QA | sonnet | medium | 200 | Test writing, moderate complexity |

These are defaults that ship with templates. Users customize per their needs and budget.

### SOUL.md Format

```markdown
# SOUL.md -- Backend Engineer Persona

You are the Backend Engineer...

## Technical Posture
- [Identity and decision-making principles]

## Voice and Tone
- [Communication style]

## Working Style
- [How to approach work — TDD, conventions, etc.]

## Boundaries
- [What NOT to work on — delegate to other personas]

## Quality Checklist
- [Pre-completion checks]
```

### CEO Persona (Special)

The CEO never writes code. It triages, decomposes, coordinates, and escalates.

- `escalates_to: board` (special value — @-mentions the Board user)
- `label: null` with `is_default: true` (fallback when no persona label matches)
- SOUL.md focuses on strategic posture, triage decisions, decomposition rules

## 4. Config & Initialization

### Repo-Level Config

```yaml
# .woterclip/config.yaml
version: 1

linear:
  user_name: "Alex Kim"          # Board user's display name (for comment mentions)
  team: "WotAI"                  # Default team for new issues/sub-issues
  project: null                  # Optional: scope to specific project
  # Note: assignee queries use "me" (resolved dynamically by Linear MCP)
  # user_id is NOT stored — avoids breakage if accounts change

heartbeat:
  max_issues_per_heartbeat: 2
  stale_lock_hours: 4
  quiet_hours:
    enabled: false
    timezone: "America/Los_Angeles"
    start: "22:00"
    end: "07:00"
    behavior: "skip"             # "skip" or "triage-only" (CEO triage but no code execution)
  cooldown:
    max_consecutive_failures: 3
    backoff_multiplier: 2

labels:
  group: "WoterClip"             # Parent label group in Linear (auto-created by init)
  working: "agent-working"
  blocked: "agent-blocked"

personas:
  ceo:
    path: "personas/ceo"
    label: null
    is_default: true
  backend:
    path: "personas/backend"
    label: "backend"
  frontend:
    path: "personas/frontend"
    label: "frontend"
```

#### Config Versioning

When the plugin updates the config schema, it checks the `version` field:
- **Same version** → proceed normally
- **Older version** → auto-migrate with backup (copy to `config.yaml.bak`), log what changed
- **Newer version** → refuse to run, prompt user to update plugin

#### Quiet Hours: Triage-Only Mode

When `behavior: "triage-only"`, the heartbeat runs CEO persona only:
- Triage unlabeled issues (apply labels, decompose)
- Skip code execution (no worker persona work)
- Post triage comments as normal
- Useful for overnight label routing without code changes

### `/woterclip-init` Flow

1. Check Linear MCP is available → fail with helpful error if not
2. Fetch Linear user info → auto-fill `user_id` and `user_name`
3. Fetch Linear teams → let user pick team
4. Ask which personas to scaffold (presets: "engineering", "marketing", "custom")
5. Create "WoterClip" label group in Linear, then create `agent-working` and `agent-blocked` as child labels, plus persona labels (e.g., `backend`, `frontend`)
6. Write `config.yaml` and persona directories with templates
7. Offer to set up recurring heartbeat schedule
8. Print summary of what was created

## 5. Comment Format

### Standard Template

```markdown
## Heartbeat #3 — 2026-03-25 10:15 UTC (12 min)

**Status:** In Progress | Completed | Blocked

### What was done
- [`a1b2c3d`](link) feat(api): add GET /api/bookings endpoint
- Added Zod validation schema for booking filters
- Unit tests passing (4/4)

### Created sub-issues
- [WOT-83](link) — Wire to dashboard page (frontend)

### What's next
- Integration tests for the endpoint

### Blockers
None

---
*WoterClip · backend · [WOT-79](link) · from [Heartbeat #2](link)*
```

### Blocked Template

```markdown
## Heartbeat #2 — 2026-03-25 09:00 UTC (3 min)

**Status:** Blocked

### Blocker
Missing `STRIPE_WEBHOOK_SECRET` in `.env.local`. Cannot test webhook
handler without it.

### Action needed
@Alex Kim — please add the webhook secret or provide instructions
for generating a new one.

### What was done before blocking
- Implemented webhook route handler
- Added signature verification logic

---
*WoterClip · backend · [WOT-82](link)*
```

### Comment Rules

- Always include heartbeat counter and timestamp with duration
- Always include persona name and issue link in footer
- Reference previous heartbeat comment for carry-forward
- Blocked comments must name who needs to act
- Completion comments must list shipped commits/PRs with links
- Use `⚠️` flag for uncertain work that needs manual verification
- Fast-path triage: `**Triage:** → backend` for obvious routing

## 6. CEO Triage Logic

### Decision Framework

1. **Single-persona work** — Apply persona label, leave triage comment
2. **Multi-persona work** — Decompose into sub-issues with persona labels, set dependencies
3. **Unclear scope** — Apply `ceo` label, mark blocked, ask Board
4. **Non-code work, no matching persona** — Escalate to Board
5. **Large scope (4+ sub-issues)** — Flag to Board before decomposing

### Triage Rules

- **One issue = one persona.** Never dual-label — always decompose
- **Sub-issues inherit parent priority.** Blocking sub-issues get +1 priority bump
- **Fast-path obvious routing.** `**Triage:** → backend` for clear cases
- **Check recent similar issues** for routing consistency before deciding
- **Parent completion check.** When a sub-issue completes, check if all siblings are done → close parent with summary

### Label Heuristics

| Signal in issue | Persona |
|-----------------|---------|
| API, endpoint, route, database, migration, query, webhook | `backend` |
| Component, UI, page, layout, styling, responsive, animation | `frontend` |
| Deploy, CI/CD, Docker, env vars, infrastructure | `infra` |
| Test, coverage, E2E, integration test, flaky | `qa` |
| No code signals, architecture, cross-cutting | `ceo` (keeps it) |

## 7. Schedule Integration

### Trigger Methods

```bash
/heartbeat                    # Manual: run one cycle now
/heartbeat --dry-run          # Manual: show what would be picked up
/heartbeat --persona backend  # Manual: force a specific persona
/schedule 30m /heartbeat      # Automatic: recurring heartbeat
```

### Recommended Cadences

| Workload | Cadence | Reasoning |
|----------|---------|-----------|
| Active sprint | Every 15-30 min | Fast iteration, quick unblocking |
| Steady state | Every 1-2 hours | Regular progress without burn |
| Background | Every 4-6 hours | Low-priority, overnight runs |
| Paused | No schedule | Manual `/heartbeat` only |

### Adaptive Cadence Suggestions

The heartbeat suggests cadence changes based on workload:

- 0 todo issues → suggest pausing schedule
- 3+ blocked issues → suggest Board attention, not more heartbeats
- Critical issue queued → suggest shortening cadence

### Error Cooldown

- 1 failure → normal schedule
- 2 consecutive failures → double interval, notify Board
- 3 consecutive failures → pause schedule, require manual restart

### Overlap Prevention

If a previous heartbeat is still active when the next triggers, skip the beat and log it.

### Quiet Hours

Configurable time window where heartbeats are skipped or limited to triage-only.

### `/woterclip-status` Output

```
WoterClip Status
────────────────
Schedule:     Every 30 min (active)
Last beat:    Heartbeat #7 — 12 min ago

Since last heartbeat:
  ✓ WOT-81  [backend]   Completed    "Add Zod schemas for bookings"
  → WOT-79  [backend]   In Progress  "Add booking API filters"
  ✗ WOT-82  [backend]   Blocked      "Missing STRIPE_WEBHOOK_SECRET"
  + WOT-83  [frontend]  Created      "Wire bookings to dashboard"

Queue (next heartbeat):
  WOT-84  [backend]  Todo  High   "Booking notification emails"
  WOT-85  [backend]  Todo  Medium "Add pagination to bookings list"

Blocked (needs Board):
  WOT-82  @Alex Kim — missing env var
```

### Heartbeat History

```
/woterclip-status --history

Heartbeat History (last 5)
──────────────────────────
#7  10:15  backend   WOT-79  In Progress  (12 min)
#6  09:45  backend   WOT-81  Completed    (8 min)
#5  09:15  ceo       WOT-80  Triaged → 3 sub-issues (4 min)
#4  08:45  frontend  WOT-78  Completed    (15 min)
#3  08:15  backend   WOT-82  Blocked      (3 min)
```

## 8. Portability

### For Users (Plugin Install)

1. Install WoterClip plugin from marketplace
2. Run `/woterclip-init` in any repo
3. Customize personas for that repo's stack
4. Run `/heartbeat` or set up `/schedule`

### For Persona Sharing

Persona directories are self-contained. Users can:
- Copy persona dirs between repos
- Share personas as gists or in community forums
- Use `/persona-import` to convert Paperclip agent configs

### Prerequisites

- Claude Code with Linear MCP connected
- Linear workspace with the target team
- `.mcp.json` in the repo pointing to the right Linear workspace (if not using global config)

## 9. Persona Import from Paperclip

The `/persona-import` skill converts Paperclip agent directories into WoterClip persona directories.

### Mapping

| Paperclip File | WoterClip Target | What's Imported |
|----------------|-----------------|-----------------|
| `SOUL.md` | `SOUL.md` | Copied as-is (identity, posture, voice) |
| `HEARTBEAT.md` | Skipped | Replaced by plugin's generic heartbeat. Role-specific responsibilities (bottom section) are appended to SOUL.md |
| `AGENTS.md` | `config.yaml` + `SOUL.md` | Safety rules → SOUL.md. Workspace paths → dropped. Memory refs → dropped |
| `TOOLS.md` | `TOOLS.md` | Copied as-is, with Paperclip API references replaced by Linear MCP equivalents |
| `life/`, `memory/` | Skipped | Claude Code has built-in memory |
| Playbook references | `references/` dir (optional) | If AGENTS.md references playbooks, note them in TOOLS.md for manual migration |

### What's NOT Imported

- **Budget tracking** — Paperclip budget fields have no WoterClip equivalent
- **Approval workflows** — Intentionally omitted from v1 (see Future Work)
- **PARA memory system** — Replaced by Claude Code built-in memory
- **Agent hiring** — Replaced by `/persona-create`

## 10. Runtime State & Storage

### Heartbeat Log

Each heartbeat appends a JSON line to `.woterclip/heartbeat-log.jsonl`:

```json
{"heartbeat": 7, "timestamp": "2026-03-25T10:15:00Z", "issue": "WOT-79", "persona": "backend", "duration_sec": 720, "status": "in_progress", "actions": ["committed a1b2c3d", "created sub-issue WOT-83"]}
```

This powers `/woterclip-status --history`. The file is append-only and can be safely truncated (informational, not functional).

### Lockfile

`.woterclip/.heartbeat-lock` contains the timestamp when the current heartbeat started. Auto-deleted on exit. Stale locks (older than `stale_lock_hours`) are auto-cleaned.

### Label State

All agent state labels (`agent-working`, `agent-blocked`) live in Linear. No local label state is cached.

## 11. Future Work (Intentionally Omitted from v1)

| Feature | Why Omitted | When to Add |
|---------|-------------|-------------|
| **Approval workflows** | Paperclip's approval system is tightly coupled to its API. Linear doesn't have native approval flows. Could be built with custom labels or Linear integrations | v2, if users request it |
| **Budget tracking** | Would need token counting per heartbeat. Claude Code doesn't expose cost data easily | v2, when `/schedule` exposes cost metrics |
| **Multi-agent concurrency** | v1 runs one instance per repo. Concurrency adds label race conditions | v2, with proper locking |
| **Cross-team delegation** | v1 assumes single team per repo. Cross-team needs multi-workspace support | v2, for monorepo/platform setups |

## Appendix: Comparison to Paperclip

| Capability | Paperclip | WoterClip |
|-----------|-----------|-----------|
| Task store | Paperclip API | Linear |
| Heartbeat runner | Paperclip scheduler | Claude Code `/schedule` |
| Agent identity | API records | Per-repo persona files (SOUL.md, TOOLS.md, config.yaml) |
| Checkout locking | `POST /checkout` + 409 | `agent-working` label (read-modify-write) |
| Chain of command | API-driven hierarchy | `escalates_to` in persona config |
| Escalation notifications | API-driven | Comment mention + optional issue reassignment |
| Memory | Custom PARA system | Claude Code built-in memory |
| Budget tracking | API-tracked spend | Not implemented v1 (future) |
| Approval workflows | API-driven governance | Not implemented v1 (future) |
| Playbooks/SOPs | Referenced in AGENTS.md | Optional `references/` dir per persona |
| Agent hiring | `paperclip-create-agent` | `/persona-create` |
| Runtime config | Per-agent (model, turns, etc.) | Per-persona config.yaml (model, thinking, turns) |
| Multi-agent concurrency | Separate processes | Single instance, persona switching |
| Overlap prevention | API-level checkout | Local lockfile |
| Issue management | Paperclip API (limited) | Linear (full-featured) |
| Distribution | Paperclip platform | Claude Code plugin marketplace |
| Heartbeat history | API runs endpoint | Local `.jsonl` log file |
