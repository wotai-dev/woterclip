---
name: heartbeat
description: This skill should be used when the user asks to "run a heartbeat", "run the agent loop", "process Linear issues", "check for work", or runs the /heartbeat command. Executes the WoterClip heartbeat — picks up Linear issues, resolves personas, does work, and reports back.
version: 0.1.0
---

# WoterClip Heartbeat

Execute the WoterClip heartbeat cycle: pick up assigned Linear issues, resolve the right persona, do the work, and report back with structured comments.

**Arguments:**
- `--dry-run` — Show what would be picked up without doing work
- `--persona <name>` — Only pick issues matching a specific persona

**Reference files** (consult as needed during execution):
- `${CLAUDE_PLUGIN_ROOT}/references/comment-format.md` — Comment templates and rules
- `${CLAUDE_PLUGIN_ROOT}/references/label-conventions.md` — Label lifecycle and read-modify-write pattern
- `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md` — Linear states, sort order, inbox filtering

## Step 1: Load Config & Lock

1. Read `.woterclip/config.yaml`. If missing, stop and instruct the user to run `/woterclip-init`.
2. Check for lockfile at `.woterclip/.heartbeat-lock`.
   - If lockfile exists and is **less than** `stale_lock_hours` old → stop with message: "Previous heartbeat still active. Skipping."
   - If lockfile exists and is **older than** `stale_lock_hours` → delete it, log: "Cleaned stale lockfile."
   - If no lockfile → proceed.
3. Create lockfile with current ISO timestamp.
4. **On any exit path** (success, error, or early return), delete the lockfile.

Check quiet hours: if `quiet_hours.enabled` and current time is within the quiet window:
- `behavior: "skip"` → delete lockfile and exit with message: "Quiet hours active. Skipping."
- `behavior: "triage-only"` → proceed but only load Orchestrator persona (skip worker personas in step 3).

## Step 2: Check Inbox

1. Call `mcp__claude_ai_Linear__list_issues` with filter for assigned issues (`assignee: "me"`).
2. Filter client-side:
   - **Keep** only issues with status "In Progress" or "Todo"
   - **Skip** issues without a persona label (unless Orchestrator is default and issue has no label)
   - **Skip** `agent-blocked` issues unless new human comments exist since the last agent comment (check via `mcp__claude_ai_Linear__list_comments`)
3. Sort:
   - Primary: status — In Progress before Todo
   - Secondary: priority — Urgent > High > Medium > Low > None
4. Detect stale `agent-working` labels: if an issue has `agent-working` but no heartbeat comment within `stale_lock_hours`, clean the stale label (remove `agent-working`, post cleanup comment).

## Step 3: Pick Issue

1. If `--persona <name>` flag is set, filter to only issues matching that persona's label.
2. Pick the first issue from the sorted inbox.
3. If `--dry-run`, report what would be picked and exit:
   ```
   Dry run — would pick:
     WOT-XX [backend] "Issue title" (In Progress, High)
   Queue:
     WOT-YY [frontend] "Other issue" (Todo, Medium)
   ```
4. If no issues match → delete lockfile and exit: "No issues in queue. Heartbeat complete."

## Step 4: Resolve Persona

1. Read the issue's labels. Find the persona label by matching against the `personas` map in config.
2. No persona label found → load the persona with `is_default: true` (typically Orchestrator).
3. Load persona files from `.woterclip/<persona.path>/`:
   - `SOUL.md` → inject into context as identity instructions
   - `TOOLS.md` → inject into context as tool guidance
   - `config.yaml` → read runtime settings

Apply runtime config from persona's `config.yaml`:
- `model` — note the target model (informational; cannot switch mid-session)
- `thinking_effort` — apply if supported
- `max_turns` — respect as work budget
- `enable_chrome` — note for browser-dependent tasks

## Step 5: Validate Tools

Read `required_tools` from persona config. For each entry, verify the tool prefix is available:
- `mcp__claude_ai_Linear` should match any tool starting with `mcp__claude_ai_Linear__`
- If a required tool prefix has **no matching tools** available → stop work on this issue immediately
  - Post a blocked comment naming the missing tool
  - Apply `agent-blocked` label (read-modify-write)
  - Remove `agent-working` if present
  - Proceed to step 11 (next issue)

## Step 6: Lock Issue

1. Call `mcp__claude_ai_Linear__get_issue` to read the issue's current labels.
2. If `agent-working` is already present (from a previous heartbeat on same issue), proceed without re-saving.
3. Otherwise, append `agent-working` to the labels array and call `mcp__claude_ai_Linear__save_issue` with the full label set.

## Step 7: Understand Context

1. Read issue title, description, and all comments via `mcp__claude_ai_Linear__get_issue` and `mcp__claude_ai_Linear__list_comments`.
2. If the issue has a parent, read the parent issue for broader context.
3. Identify new comments since the last heartbeat (look for comments after the last WoterClip-formatted comment).
4. Parse heartbeat counter: find the last comment matching `Heartbeat #N` pattern. Next comment will be `#N+1`. If none found, start at `#1`.

## Step 8: Do Work

Follow the persona's SOUL.md instructions. This step varies by persona:

**Orchestrator persona:** Triage the issue – apply persona labels, create sub-issues, or escalate. Never write code.

**CEO persona:** Make strategic decisions – prioritization, scope, architecture, coordination. Never write code.

**Worker personas (backend, frontend, etc.):**
- Use repo tools (Read, Write, Edit, Bash, Grep, Glob) to implement changes
- For large scope: create Linear sub-issues via `mcp__claude_ai_Linear__save_issue` with `parentId` set to current issue, `team` from config, and appropriate persona labels
- For small scope: work directly, use internal tasks to track progress
- Commit changes with descriptive conventional commit messages
- Respect `max_turns` from persona config as a work budget

**If Linear MCP becomes unavailable mid-work:** Stop immediately. Leave `agent-working` label in place (will be cleaned as stale on next heartbeat). Delete lockfile and exit with error log.

## Step 9: Report

Post a structured comment on the Linear issue via `mcp__claude_ai_Linear__save_comment`.

Follow the comment format from `${CLAUDE_PLUGIN_ROOT}/references/comment-format.md`:
- Include `Heartbeat #N` counter (incremented from step 7)
- Include timestamp and duration
- Include persona name in footer
- List commits with SHAs, sub-issues created, and next steps
- For blocked status: name who needs to act (Board user from config `linear.user_name`)

Append heartbeat metadata to `.woterclip/heartbeat-log.jsonl`:
```json
{"heartbeat": N, "timestamp": "ISO", "issue": "WOT-XX", "persona": "name", "duration_sec": N, "status": "in_progress|completed|blocked", "actions": ["description"]}
```

## Step 10: Update State

Read the issue's current labels via `mcp__claude_ai_Linear__get_issue`, then update based on outcome:

| Outcome | Labels | Status |
|---------|--------|--------|
| **Completed** | Remove `agent-working` | Move to Done (or In Review if PR opened) |
| **Blocked** | Remove `agent-working`, add `agent-blocked` | Keep In Progress |
| **More work needed** | Keep `agent-working` | Keep In Progress |

For blocked issues: include the Board user's display name in the comment text (e.g., "**@Alex Kim** — please review").

## Step 11: Next Issue or Exit

1. If issues worked this heartbeat < `max_issues_per_heartbeat`, return to **Step 2** to pick the next issue.
2. Otherwise, delete lockfile and exit.
3. If 0 todo issues remain in queue, suggest pausing the schedule.
4. If 3+ issues are blocked, suggest Board attention rather than more heartbeats.
