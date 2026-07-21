# WoterClip

GitHub Issues-backed agent orchestration for Claude Code. A single Claude instance wears different "hats" (personas) based on GitHub issue labels – an Orchestrator routes work, a CEO makes strategic calls, and worker personas execute.

**Get the persona starter pack →** https://alexk1919.gumroad.com/l/woterclip-persona-pack

## How It Works

```
GitHub Issues → /heartbeat → Persona Matching → Work → Report Back
```

1. **Issues** live on GitHub with persona labels (`backend`, `frontend`, etc.)
2. **Heartbeat** picks the highest-priority issue and resolves which persona handles it
3. **Personas** (CEO, Backend, Frontend, ...) define identity, tools, and runtime config
4. **Reports** are structured comments on the GitHub issue with progress, commits, and blockers
5. **Schedule** runs heartbeats automatically via Claude Code's `/schedule`

The human is the **Board** – the ultimate escalation target when the agent is blocked.

## Install

In Claude Code, run `/plugin` → **Add Marketplace** → enter `wotai-dev/woterclip`, then install the plugin.

Or for local development:

```bash
git clone https://github.com/wotai-dev/woterclip.git
claude --plugin-dir /path/to/woterclip
```

### Prerequisites

- [Claude Code](https://claude.ai/code) installed
- [GitHub CLI](https://cli.github.com) installed and authenticated:
  ```bash
  brew install gh   # or your platform's equivalent
  gh auth login
  ```
- A GitHub repository with Issues enabled
- Issues assigned to your GitHub account (the heartbeat queries `--assignee @me`)

No MCP server is required – all GitHub operations go through the `gh` CLI, so scheduled heartbeats work headlessly.

Note on notifications: the agent posts comments as whatever account `gh` is authenticated as, and GitHub never notifies a user of their own comments. If you authenticate `gh` with your personal account (the common setup), blocked-escalation @-mentions won't notify you – watch the repo or check `/woterclip-status`. A separate bot/machine account for `gh auth` gives you real mention notifications.

## Quick Start

```bash
# 1. Initialize WoterClip in your repo (creates config + personas + GitHub labels)
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

1. **Load config** – read `.woterclip/config.yaml`, check lockfile
2. **Check inbox** – query GitHub for assigned open issues, filter and sort
3. **Pick issue** – `in-progress` labeled first, then the rest, by priority label
4. **Resolve persona** – match issue label → persona directory, load SOUL.md + TOOLS.md
5. **Validate tools** – check required tools are available (`gh auth status`)
6. **Lock issue** – add `agent-working` label
7. **Understand context** – read issue, comments, parent, heartbeat counter
8. **Do work** – dispatch a persona subagent on the persona's configured model (inline fallback, disclosed in the report, when dispatch is unavailable)
9. **Report** – post structured comment with progress, commits, sub-issues
10. **Update state** – manage labels based on the returned outcome (completed / in progress / blocked / triaged / decomposed)
11. **Next or exit** – pick another issue or clean up and stop

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
| Orchestrator | Route issues, decompose work | Haiku | 50 | *(default – no label)* |
| CEO | Strategy, prioritization, architecture | Sonnet | 100 | `ceo` |
| Backend | API, database, server-side | Opus | 300 | `backend` |
| Frontend | UI, components, styling | Sonnet | 200 | `frontend` |

Create custom personas with `/persona-create` or copy directories between repos.

### Per-Repo Structure

After `/woterclip-init`, your repo gets:

```
.woterclip/
├── config.yaml              # GitHub settings, heartbeat behavior, persona routing
├── heartbeat-log.jsonl      # Append-only heartbeat history (created at runtime)
└── personas/
    ├── orchestrator/
    │   ├── SOUL.md
    │   ├── TOOLS.md
    │   └── config.yaml
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

WoterClip uses GitHub issue labels for state management:

| Label | Purpose |
|-------|---------|
| `agent-working` | Agent is actively working this issue |
| `agent-blocked` | Agent is blocked, needs Board attention |
| `backend`, `frontend`, etc. | Routes issue to the matching persona |
| `backlog`, `todo`, `in-progress`, `in-review` | Optional status labels (GitHub issues are only open/closed – these carry the working-state distinctions) |
| `priority:high`, `priority:low` | Optional priority labels (GitHub has no native priority field) |

GitHub has no label groups, so labels are flat – created by `/woterclip-init`.

## Schedule Cadences

| Workload | Cadence | Command |
|----------|---------|---------|
| Active sprint | Every 15-30 min | `/schedule 15m /heartbeat` |
| Steady state | Every 1-2 hours | `/schedule 1h /heartbeat` |
| Background | Every 4-6 hours | `/schedule 4h /heartbeat` |
| Manual only | No schedule | `/heartbeat` when needed |

## Migrating from Linear (v1 configs)

WoterClip was originally Linear-backed. If a repo has a `version: 1` config (`linear:` section), re-run `/woterclip-init` – it detects the old schema and migrates: your GitHub login replaces the Linear display name (you'll be asked – they're not the same thing), the repo replaces the team, and labels are recreated on GitHub. Linear issue data is not migrated.

## Migrating from Paperclip

Use `/persona-import` to convert Paperclip agent directories into WoterClip personas. It maps SOUL.md, TOOLS.md, HEARTBEAT.md role-specific sections, and AGENTS.md safety rules into the WoterClip format. Budget tracking, PARA memory, and approval workflows are not imported (replaced by Claude Code built-in features or intentionally omitted from v1).

## Background

WoterClip is inspired by [Paperclip](https://github.com/paperclipai/paperclip), an agent orchestration platform that uses a central API for task management, agent checkout, and chain-of-command routing. WoterClip takes the same core ideas – persona-based identity, structured heartbeats, hierarchical escalation – and rebuilds them as a Claude Code plugin backed by GitHub Issues instead of a custom API. The result is simpler (no server, no database, no separate processes) while keeping the parts that worked well: SOUL.md for agent identity, structured comments for audit trails, and a CEO/worker hierarchy for task decomposition.

## Design

See [`docs/specs/2026-03-25-woterclip-design.md`](docs/specs/2026-03-25-woterclip-design.md) for the full design spec and [`docs/specs/2026-03-25-woterclip-implementation-plan.md`](docs/specs/2026-03-25-woterclip-implementation-plan.md) for the build order. (The specs describe the original Linear-backed design – the tracker swap to GitHub Issues happened in [#1](https://github.com/wotai-dev/woterclip/issues/1).)

## License

[MIT](LICENSE)
