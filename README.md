# WoterClip

Linear-backed agent orchestration for Claude Code. A single Claude instance wears different "hats" (personas) based on Linear issue labels — a CEO triages work, worker personas execute it.

## How It Works

```
Linear Issues → /heartbeat → Persona Matching → Work → Report Back
```

1. **Issues** live in Linear with persona labels (`backend`, `frontend`, etc.)
2. **Heartbeat** picks the highest-priority issue and resolves which persona handles it
3. **Personas** (CEO, Backend, Frontend, ...) define identity, tools, and runtime config
4. **Reports** are structured comments on the Linear issue with progress, commits, and blockers
5. **Schedule** runs heartbeats automatically via Claude Code's `/schedule`

The human is the **Board** — the ultimate escalation target when the agent is blocked.

## Install

```bash
claude plugin add /path/to/woterclip
```

### Prerequisites

- [Claude Code](https://claude.ai/code) installed
- [Linear MCP](https://linear.app/docs/mcp) connected — add to your `.mcp.json` or global MCP config:
  ```json
  {
    "mcpServers": {
      "linear": {
        "type": "url",
        "url": "https://mcp.linear.app/sse"
      }
    }
  }
  ```
- Linear workspace with at least one team
- Issues assigned to your Linear account (the heartbeat queries `assignee: "me"`)

## Quick Start

```bash
# 1. Initialize WoterClip in your repo (creates config + personas + Linear labels)
/woterclip-init

# 2. Run a single heartbeat cycle
/heartbeat

# 3. Or schedule recurring heartbeats
/schedule 30m /heartbeat

# 4. Check status
/woterclip-status
```

## The Heartbeat Loop

Each `/heartbeat` runs an 11-step cycle:

1. **Load config** — read `.claude/woterclip/config.yaml`, check lockfile
2. **Check inbox** — query Linear for assigned issues, filter and sort
3. **Pick issue** — highest priority In Progress, then Todo
4. **Resolve persona** — match issue label → persona directory, load SOUL.md + TOOLS.md
5. **Validate tools** — check required MCP tools are available
6. **Lock issue** — add `agent-working` label
7. **Understand context** — read issue, comments, parent, heartbeat counter
8. **Do work** — follow persona instructions (CEO triages, workers implement)
9. **Report** — post structured comment with progress, commits, sub-issues
10. **Update state** — manage labels based on outcome (done/blocked/continuing)
11. **Next or exit** — pick another issue or clean up and stop

Use `--dry-run` to see what would be picked without doing work. Use `--persona backend` to force a specific persona.

## Personas

Each persona gets its own directory with three files:

| File | Purpose |
|------|---------|
| `SOUL.md` | Identity, posture, voice, decision framework |
| `TOOLS.md` | Available tools and integrations |
| `config.yaml` | Runtime config (model, thinking effort, max turns) |

### Default Personas

| Persona | Role | Model | Turns | Label |
|---------|------|-------|-------|-------|
| CEO | Triage, decompose, coordinate | Sonnet | 100 | *(default — no label)* |
| Backend | API, database, server-side | Opus | 300 | `backend` |
| Frontend | UI, components, styling | Sonnet | 200 | `frontend` |

Create custom personas with `/persona-create` or copy directories between repos.

### Per-Repo Structure

After `/woterclip-init`, your repo gets:

```
.claude/woterclip/
├── config.yaml              # Linear settings, heartbeat behavior, persona routing
├── heartbeat-log.jsonl      # Append-only heartbeat history (created at runtime)
└── personas/
    ├── ceo/
    │   ├── SOUL.md
    │   ├── TOOLS.md
    │   └── config.yaml
    ├── backend/
    │   ├── SOUL.md
    │   ├── TOOLS.md
    │   └── config.yaml
    └── frontend/
        ├── SOUL.md
        ├── TOOLS.md
        └── config.yaml
```

## Commands

| Command | Description |
|---------|-------------|
| `/heartbeat` | Run one heartbeat cycle |
| `/heartbeat --dry-run` | Show what would be picked up |
| `/heartbeat --persona backend` | Force a specific persona |
| `/woterclip-init` | Initialize WoterClip in a repo |
| `/woterclip-status` | Current state, queue, blocked issues |
| `/woterclip-status --history` | Recent heartbeat history |
| `/persona-create` | Create a new persona interactively |
| `/persona-list` | List configured personas |

## Label System

WoterClip uses Linear labels for state management:

| Label | Purpose |
|-------|---------|
| `agent-working` | Agent is actively working this issue |
| `agent-blocked` | Agent is blocked, needs Board attention |
| `backend`, `frontend`, etc. | Routes issue to the matching persona |

All labels live under a "WoterClip" parent group in Linear, created by `/woterclip-init`.

## Schedule Cadences

| Workload | Cadence | Command |
|----------|---------|---------|
| Active sprint | Every 15-30 min | `/schedule 15m /heartbeat` |
| Steady state | Every 1-2 hours | `/schedule 1h /heartbeat` |
| Background | Every 4-6 hours | `/schedule 4h /heartbeat` |
| Manual only | No schedule | `/heartbeat` when needed |

## Migrating from Paperclip

Use `/persona-import` to convert Paperclip agent directories into WoterClip personas. It maps SOUL.md, TOOLS.md, HEARTBEAT.md role-specific sections, and AGENTS.md safety rules into the WoterClip format. Budget tracking, PARA memory, and approval workflows are not imported (replaced by Claude Code built-in features or intentionally omitted from v1).

## Design

See [`docs/specs/2026-03-25-woterclip-design.md`](docs/specs/2026-03-25-woterclip-design.md) for the full design spec and [`docs/specs/2026-03-25-woterclip-implementation-plan.md`](docs/specs/2026-03-25-woterclip-implementation-plan.md) for the build order.

## License

MIT
