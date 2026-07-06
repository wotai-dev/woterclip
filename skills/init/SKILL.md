---
name: woterclip-init
description: This skill should be used when the user asks to "initialize woterclip", "set up woterclip", "woterclip init", "configure woterclip for this repo", or runs the /woterclip-init command. Scaffolds a repo with WoterClip config, persona directories, and GitHub labels.
version: 0.2.0
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
   - One label per persona that has a non-null label (e.g., `backend` `1D76DB`, `frontend` `5319E7`)
3. Skip creation for any label that already exists (matching on name).

Optionally offer to create the status labels (`backlog`, `todo`, `in-progress`, `in-review`) and priority labels (`priority:high`, `priority:low`) — see `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md`. These are recommended but not required: the heartbeat treats unlabeled open issues as eligible.

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

- **Yes** → Suggest: `/schedule 30m /heartbeat` and explain cadence options
- **Not now** → Explain they can run `/heartbeat` manually or set up `/schedule` later

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
| `.woterclip/` already exists | Ask user: overwrite, merge, or cancel. Default to merge (skip existing files). |
| Template file missing from plugin | Log warning, create a minimal placeholder, continue. |

## Re-initialization & Migration

If `.woterclip/config.yaml` already exists:

1. Read the existing config `version`.
2. **If `version: 1` (Linear-era config), run the v1→v2 migration:**
   - `linear.user_name` → **ask the user for their GitHub login** — a Linear display name is not a GitHub login; never guess. Pre-fill from `gh api user --jq .login`.
   - `linear.team` → discarded; `github.repo` comes from `gh repo view` (confirm with the user).
   - `linear.project` → discarded (no equivalent; GitHub milestones are out of scope).
   - `labels.group` → dropped (GitHub has no label groups; labels are flat).
   - `labels.working` / `labels.blocked` and the entire `personas` + `heartbeat` sections carry over unchanged.
   - Write `version: 2`. Back up the old config to `config.yaml.bak` first.
   - Create the GitHub labels (step 3) — Linear labels are not migrated automatically.
3. Otherwise ask the user: **overwrite** (fresh start), **merge** (add missing personas only), or **cancel**.
4. For merge: only create persona directories and labels that don't exist yet.
5. For overwrite: back up existing config to `config.yaml.bak` before writing.
