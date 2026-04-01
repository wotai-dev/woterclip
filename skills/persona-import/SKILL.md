---
name: persona-import
description: This skill should be used when the user asks to "import a paperclip agent", "convert paperclip to woterclip", "migrate from paperclip", "import persona from paperclip", or wants to convert existing Paperclip agent directories into WoterClip persona format.
version: 0.1.0
---

# Import Persona from Paperclip

Convert a Paperclip agent directory into a WoterClip persona. Maps Paperclip's agent structure (SOUL.md, AGENTS.md, HEARTBEAT.md, TOOLS.md) to WoterClip's persona structure (SOUL.md, TOOLS.md, config.yaml).

## Prerequisites

- `.woterclip/config.yaml` must exist (run `/woterclip-init` first)
- User must provide the path to the Paperclip agent directory

## File Mapping

| Paperclip File | WoterClip Target | What to Import |
|----------------|-----------------|----------------|
| `SOUL.md` | `SOUL.md` | Copy as-is — identity, posture, voice |
| `HEARTBEAT.md` | Append to `SOUL.md` | Only the role-specific responsibilities section (bottom). Skip the generic heartbeat procedure (top) — replaced by the plugin's heartbeat skill |
| `AGENTS.md` | `config.yaml` + `SOUL.md` | Safety rules → append to SOUL.md Boundaries section. Workspace paths → drop. Memory refs → drop |
| `TOOLS.md` | `TOOLS.md` | Copy, replacing Paperclip API references with Linear MCP equivalents |
| `life/`, `memory/` | Skip | Claude Code has built-in memory |

## Import Procedure

### Step 1: Read Source

Ask the user for the Paperclip agent directory path. Read all files present:
- `SOUL.md` (required)
- `AGENTS.md` (optional)
- `HEARTBEAT.md` (optional)
- `TOOLS.md` (optional)

If `SOUL.md` doesn't exist, stop — this is not a valid Paperclip agent directory.

### Step 2: Ask for Persona Details

Ask the user:
1. **Persona name** — e.g., "Backend Engineer" (pre-fill from SOUL.md if obvious)
2. **Linear label** — e.g., `backend` (must be unique)
3. **Role type** — `engineer`, `orchestrator`, `analyst`, or custom
4. **Model** — `opus`, `sonnet`, or `haiku` (suggest based on role)
5. **Escalates to** — default: `ceo`

### Step 3: Transform SOUL.md

1. Start with the Paperclip `SOUL.md` content
2. From `HEARTBEAT.md`: extract the role-specific section (usually the bottom half, after the generic heartbeat steps). Append as a "## Working Style" or "## Role Responsibilities" section
3. From `AGENTS.md`: extract safety rules and boundaries. Append to the "## Boundaries" section
4. Add a WoterClip-specific "## Quality Checklist" section if not present
5. Drop any Paperclip-specific references (workspace paths, PARA memory, budget tracking)

### Step 4: Transform TOOLS.md

1. Start with the Paperclip `TOOLS.md` content
2. Replace Paperclip API references:
   - `paperclip` skill → `mcp__claude_ai_Linear__*` (Linear MCP tools)
   - `para-memory-files` → drop (replaced by Claude Code built-in memory)
   - Paperclip API calls (`POST /checkout`, `PATCH /api/issues`) → Linear MCP equivalents
3. Add a "## Required" section listing `mcp__claude_ai_Linear` and any other tool prefixes
4. If AGENTS.md references playbooks, note them for manual migration

### Step 5: Generate config.yaml

Create the persona config:

```yaml
name: <Name>
role: <role>
label: <label>
escalates_to: <escalates_to>

required_tools:
  - mcp__claude_ai_Linear
  # additional tools based on TOOLS.md

runtime:
  model: <model>
  thinking_effort: <high for opus, medium otherwise>
  max_turns: <300 for opus, 200 for sonnet, 100 for haiku>
  enable_chrome: false
  timeout: 0
  extra_args: []
```

### Step 6: Write Files

1. Create `.woterclip/personas/<label>/`
2. Write `SOUL.md`, `TOOLS.md`, `config.yaml`
3. Update `.woterclip/config.yaml` to add the persona to the `personas` map
4. Create the Linear label if it doesn't exist

### Step 7: Summary

```
Imported: <Name> (from <source-path>)
  Label:    <label>
  Model:    <model>

Files created:
  ✓ .woterclip/personas/<label>/SOUL.md
  ✓ .woterclip/personas/<label>/TOOLS.md
  ✓ .woterclip/personas/<label>/config.yaml

What was imported:
  ✓ SOUL.md — identity and posture
  ✓ HEARTBEAT.md — role responsibilities (appended to SOUL.md)
  ✓ AGENTS.md — safety rules (appended to Boundaries)
  ✓ TOOLS.md — tool list (Paperclip refs replaced)

What was NOT imported:
  ✗ Budget tracking (no WoterClip equivalent)
  ✗ PARA memory (replaced by Claude Code built-in memory)
  ✗ Approval workflows (not in v1)

Review SOUL.md and customize for this project's specific needs.
```
