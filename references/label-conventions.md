# Label Conventions

WoterClip uses GitHub issue labels for persona routing and agent state tracking.

## Label Namespace

GitHub has no label groups — WoterClip labels are **flat**, created by `/woterclip-init`
directly on the repo. Keep the names below verbatim so they match config defaults.

## State Labels

| Label | Purpose | Applied by | Removed by |
|-------|---------|------------|------------|
| `agent-working` | Agent is actively working this issue | Heartbeat (step 6) | Heartbeat (step 10) on done/blocked/triaged/decomposed |
| `agent-blocked` | Agent is blocked, needs Board attention | Heartbeat (step 10) | Board (manually) or heartbeat when new context appears |

### State Label Rules

- `agent-working` and `agent-blocked` are mutually exclusive — never both present
- Labels are changed with **atomic operations**: `gh issue edit N --add-label X --remove-label Y` — never rewrite the full label set, so concurrent human labeling is never clobbered
- Stale `agent-working` labels (older than `stale_lock_hours`) are auto-cleaned by the heartbeat
- `agent-blocked` issues are skipped unless new human comments exist since the last agent comment

## Persona Labels

Persona labels route issues to the right persona. Created by `/woterclip-init`.

| Label | Persona | Typical signals |
|-------|---------|-----------------|
| `backend` | Backend Engineer | API, endpoint, route, database, migration, query, webhook |
| `frontend` | Frontend Engineer | Component, UI, page, layout, styling, responsive, animation |
| `infra` | Infra Engineer | Deploy, CI/CD, Docker, env vars, infrastructure |
| `qa` | QA Engineer | Test, coverage, E2E, integration test, flaky |
| `ceo` | CEO | Strategy, prioritization, roadmap, architecture, cross-cutting |
| *(none)* | Orchestrator (default) | Unlabeled issues – routed by the Orchestrator |

### Persona Label Rules

- **One persona label per issue.** Never dual-label — decompose into sub-issues instead.
- Labels are applied by the Orchestrator during triage, or manually by the Board.
- Custom persona labels are added via `/persona-create` and registered in `config.yaml`.

## Status Labels

`backlog`, `todo`, `in-progress`, `in-review` carry the working-state distinctions GitHub's
open/closed state can't (see `status-mapping.md`). They split into two groups:

- **Agent-written (required):** `in-progress` and `in-review` — the heartbeat filters on
  and writes these, so `/woterclip-init` creates them unconditionally.
- **Board-managed (optional):** `backlog` and `todo` — humans position open work with
  these; the heartbeat only reads them, and repos may skip them entirely (unlabeled open
  issues are eligible for pickup).

## Label Lifecycle

```
New issue (no labels)
  → Orchestrator triages → adds persona label (e.g., "backend")
  → Heartbeat picks up → adds "agent-working"
  → Work completes → removes "agent-working", closes issue (or labels "in-review")
  → Or blocked → removes "agent-working", adds "agent-blocked"
  → Board unblocks → removes "agent-blocked"
  → Next heartbeat picks up again
```

## Label Operations

GitHub label changes are atomic per operation — no read-modify-write needed:

```bash
gh issue edit 42 --repo <owner/name> --add-label agent-working
gh issue edit 42 --repo <owner/name> --remove-label agent-working --add-label agent-blocked
```

Make mutually-exclusive transitions (working ↔ blocked) in **one combined** `gh issue edit` with both `--add-label` and `--remove-label`, never two separate calls — a failure between split calls would leave both labels present.

If `--add-label` fails because the label doesn't exist on the repo (`could not add label: '<name>' not found`), create it first (`gh label create <name> --repo <owner/name>`) and retry the edit — `gh issue edit` never auto-creates labels.

To read current labels (for state checks, not for writes):

```bash
gh issue view 42 --repo <owner/name> --json labels --jq '.labels[].name'
```

## Concurrency Assumption

WoterClip assumes a **single agent runner per repo**. The heartbeat lockfile (`.woterclip/.heartbeat-lock`) only guards one checkout — it does not coordinate multiple checkouts or machines operating on the same GitHub repo. Do not run scheduled heartbeats for the same repo from more than one place; labels are a shared surface and two runners would race past each other's `agent-working` markers.
