---
name: woterclip-init
description: This skill should be used when the user asks to "initialize woterclip", "set up woterclip", "woterclip init", "configure woterclip for this repo", or runs the /woterclip-init command. Scaffolds a repo with WoterClip config, persona directories, and GitHub labels.
version: 0.1.0
---

# WoterClip Initialization

Initialize WoterClip in the current repository. This creates the `.woterclip/` directory with config, persona templates, and corresponding GitHub labels.

## Prerequisites

Before starting, verify the `gh` CLI is ready:

1. Run `gh auth status` — must exit 0 (installed **and** authenticated)
2. Run `gh repo view --json nameWithOwner` — must resolve (the current directory is a repo with a GitHub remote)
3. If either fails, stop and instruct the user:
   - Install: `brew install gh` (or the platform equivalent)
   - Authenticate: `gh auth login`
   - Run from a checkout whose `origin` points at GitHub
   - Re-run `/woterclip-init`

## Initialization Procedure

### Step 1: Gather GitHub Context

1. Run `gh repo view --json nameWithOwner --jq .nameWithOwner` to resolve the target repo
2. Run `gh api user --jq .login` to identify the authenticated user
3. Present findings and ask the user to confirm:
   - The **repo** whose issues drive the heartbeat (default: the current checkout's repo)
   - The **Board user's GitHub login** for @-mentions in comments (pre-filled from `gh api user`)

When the confirmed Board user equals the authenticated login (the common single-account setup), warn: GitHub does not notify a user of their own comments, so blocked-escalation @-mentions will not produce notifications — the Board should watch the repo or rely on `/woterclip-status`. A separate bot/machine account for `gh auth` avoids this.

### Step 2: Choose Persona Preset

Ask the user which persona set to scaffold:

| Preset | Personas Created |
|--------|-----------------|
| **engineering** (default) | Orchestrator, CEO, Backend, Frontend |
| **full** | Orchestrator, CEO, Backend, Frontend, Infra, QA |
| **minimal** | Orchestrator, CEO only |
| **custom** | Orchestrator, CEO + user-specified personas |

For "custom", ask the user to name each persona and its GitHub label.

### Step 3: Create GitHub Labels

GitHub labels are flat (no groups). Create the WoterClip labels on the target repo:

1. Check what already exists: `gh label list --repo <owner/name> --limit 200`
2. Create each missing label with `gh label create <name> --repo <owner/name> --color <hex> --description "<desc>"`:
   - `agent-working` (color `0E8A16`) — state label for active work
   - `agent-blocked` (color `B60205`) — state label for blocked issues
   - `in-progress` and `in-review` — status labels the heartbeat writes on state transitions
   - `priority:high` and `priority:low` — priority labels the Orchestrator writes when bumping sub-issues
   - One label per persona that has a non-null label (e.g., `backend` `1D76DB`, `frontend` `5319E7`)
3. Skip creation for any label that already exists (matching on name).

All labels above are created unconditionally — the heartbeat and Orchestrator write them (`gh issue edit --add-label` fails when a label doesn't exist on the repo). Only the human-managed `backlog` and `todo` status labels are optional — offer them, but the heartbeat treats unlabeled open issues as eligible without them (see `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md`).

### Step 4: Scaffold Config & Personas

1. Create the directory structure:
   ```
   .woterclip/
   ├── config.yaml
   └── personas/
       ├── orchestrator/
       │   ├── SOUL.md
       │   ├── TOOLS.md
       │   └── config.yaml
       ├── ceo/
       │   ├── SOUL.md
       │   ├── TOOLS.md
       │   └── config.yaml
       ├── backend/          (if selected)
       │   ├── SOUL.md
       │   ├── TOOLS.md
       │   └── config.yaml
       └── frontend/         (if selected)
           ├── SOUL.md
           ├── TOOLS.md
           └── config.yaml
   ```

2. Copy templates from the plugin's `templates/` directory:
   - Read each template file from `${CLAUDE_PLUGIN_ROOT}/templates/`
   - Replace `{{USER_NAME}}` with the Board user's GitHub login
   - Replace `{{REPO}}` with the target repo (`owner/name`)
   - Write to `.woterclip/`

3. Update `config.yaml` personas section to match the selected preset — remove entries for personas that weren't scaffolded.

### Step 5: Offer Schedule Setup

Ask the user if they want to set up a recurring heartbeat:

- **Yes, unattended** → Suggest `/schedule 30m /heartbeat`. This is the default recommendation: it is cron-scheduled, so it survives a closed session and the next tick recovers from a beat that died mid-work.
- **Yes, while I'm working** → Suggest `/loop /heartbeat` with no interval. The loop self-paces from what each beat finds, so a quiet queue waits longer — but it is in-session only, so it stops when the session closes.
- **Not now** → Explain they can run `/heartbeat` manually or set up a schedule later

Note that a repo showing no beat for ~2 hours with a non-empty queue has stopped rather than gone idle.

### Step 6: Print Summary

Display what was created:

```
WoterClip initialized!

GitHub labels created on owner/name:
  ✓ agent-working
  ✓ agent-blocked
  ✓ backend
  ✓ frontend

Config: .woterclip/config.yaml
Personas:
  ✓ orchestrator → default (no label)
  ✓ ceo          → "ceo" label
  ✓ backend  → "backend" label
  ✓ frontend → "frontend" label

Next steps:
  1. Review .woterclip/config.yaml
  2. Customize persona SOUL.md files for your project
  3. Run /heartbeat or /schedule 30m /heartbeat
```

## Error Handling

| Error | Response |
|-------|----------|
| `gh` not installed or not authenticated | Stop. Print install/auth instructions (`brew install gh`, `gh auth login`). |
| No GitHub remote on the current repo | Stop. Ask user to run from a GitHub-hosted checkout or pass the repo explicitly. |
| Label creation fails | Log the error, continue with remaining labels, report at end. |
| `.woterclip/` already exists | Check the config `version` FIRST (see Re-initialization & Migration below — a `version: 1` config must migrate before any overwrite/merge/cancel choice). Then ask: overwrite, merge, or cancel. Default to merge (skip existing files). |
| Template file missing from plugin | Log warning, create a minimal placeholder, continue. |

## Re-initialization & Migration

If `.woterclip/config.yaml` already exists, **read the config `version` first — before offering any overwrite/merge/cancel choice** (a merge that "skips existing files" must never skip the migration):

1. Read the existing config `version`.
2. **If `version: 1` (Linear-era config), run the v1→v2 migration:**
   - `linear.user_name` → **ask the user for their GitHub login** — a Linear display name is not a GitHub login; never guess. Pre-fill from `gh api user --jq .login`.
   - `linear.team` → discarded; `github.repo` comes from `gh repo view` (confirm with the user).
   - `linear.project` → discarded (no equivalent; GitHub milestones are out of scope).
   - `labels.group` → dropped (GitHub has no label groups; labels are flat).
   - `labels.working` / `labels.blocked` and the `heartbeat` section carry over unchanged.
   - **Migrate every scaffolded persona config**: in each `.woterclip/personas/*/config.yaml`, replace `mcp__claude_ai_Linear` in `required_tools` with `gh` (heartbeat step 5 now validates `gh` via `gh auth status`; a leftover Linear entry would auto-block every issue for that persona). Flag Linear-era `TOOLS.md` files for regeneration from the plugin templates.
   - Write `version: 2`. Back up the old config to `config.yaml.bak` first.
   - Create the GitHub labels (step 3) — Linear labels are not migrated automatically.
   - **Post-migration check**: re-parse the migrated config and confirm no `required_tools` entry anywhere under `.woterclip/personas/` still references `mcp__claude_ai_Linear`.
3. Otherwise ask the user: **overwrite** (fresh start), **merge** (add missing personas only), or **cancel**.
4. For merge: only create persona directories and labels that don't exist yet.
5. For overwrite: back up existing config to `config.yaml.bak` before writing.
