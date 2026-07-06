---
name: heartbeat
description: This skill should be used when the user asks to "run a heartbeat", "run the agent loop", "process GitHub issues", "check for work", or runs the /heartbeat command. Executes the WoterClip heartbeat — picks up GitHub issues, resolves personas, does work, and reports back.
version: 0.1.0
---

# WoterClip Heartbeat

Execute the WoterClip heartbeat cycle: pick up assigned GitHub issues, resolve the right persona, do the work, and report back with structured comments. All GitHub operations go through the `gh` CLI — no MCP server required.

**Arguments:**
- `--dry-run` — Show what would be picked up without doing work
- `--persona <name>` — Only pick issues matching a specific persona

**Reference files** (consult as needed during execution):
- `${CLAUDE_PLUGIN_ROOT}/references/comment-format.md` — Comment templates and rules
- `${CLAUDE_PLUGIN_ROOT}/references/label-conventions.md` — Label lifecycle and atomic label operations
- `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md` — GitHub state model, sort order, inbox filtering
- `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md` — Canonical create/attach/verify procedure for sub-issues

All `gh issue` / `gh api` calls below target the repo from config `github.repo` (pass `--repo <owner/name>` explicitly — never rely on the working directory's default remote).

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

1. Fetch open issues assigned to the authenticated user:
   ```bash
   gh issue list --repo <owner/name> --assignee @me --state open \
     --json number,title,labels,createdAt --limit 100
   ```
2. Filter client-side (labels come from the JSON above):
   - **Skip** issues labeled `backlog` or `in-review`
   - **Skip** issues without a persona label (unless Orchestrator is default and issue has no label)
   - **Skip** `agent-blocked` issues unless new human comments exist since the last agent comment (check via `gh issue view N --json comments`)
3. Sort:
   - Primary: status label — `in-progress` before plain/`todo`
   - Secondary: priority label — `priority:high` > no priority label > `priority:low`
4. Detect stale `agent-working` labels: if an issue has `agent-working` but no heartbeat comment within `stale_lock_hours`, clean the stale label (`gh issue edit N --remove-label agent-working`, post cleanup comment).

## Step 3: Pick Issue

1. If `--persona <name>` flag is set, filter to only issues matching that persona's label.
2. Pick the first issue from the sorted inbox.
3. If `--dry-run`, report what would be picked and exit:
   ```
   Dry run — would pick:
     #12 [backend] "Issue title" (in-progress, priority:high)
   Queue:
     #15 [frontend] "Other issue" (todo)
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

Read `required_tools` from persona config. Verify each entry by its kind:
- `gh` → verify with `gh auth status` (exit 0) — this proves both the CLI and authentication
- `mcp__*` entries (e.g., `mcp__neon`) → these are MCP tool prefixes, not executables: verify by checking whether any tool starting with that prefix is available in the current session. Never run `command -v` on an `mcp__*` name.
- Other executables (e.g., `docker`) → verify with `command -v <tool>`

**If `gh` itself is unavailable or unauthenticated:** no GitHub mutation is possible — do not attempt the blocked-comment path (it needs gh). Log the failure locally to `.woterclip/heartbeat-log.jsonl`, delete the lockfile, and exit with an error message telling the user to run `gh auth login`. (Same rule as the mid-work failure in step 8.)

**If any other required tool is unavailable** → stop work on this issue immediately:
  - Post a blocked comment naming the missing tool
  - Apply `agent-blocked` label, remove `agent-working` if present:
    `gh issue edit N --add-label agent-blocked --remove-label agent-working`
  - Proceed to step 11 (next issue)

## Step 6: Lock Issue

1. Read the issue's current labels: `gh issue view N --json labels --jq '.labels[].name'`.
2. If `agent-working` is already present (from a previous heartbeat on same issue), proceed.
3. Otherwise: `gh issue edit N --add-label agent-working` (atomic — no need to rewrite the label set).

## Step 7: Understand Context

1. Read issue title, body, and all comments: `gh issue view N --json title,body,comments`.
2. If the issue is a sub-issue, read its parent for broader context: follow the `Parent: #N` reference in the issue body (guaranteed by the sub-issue creation procedure in `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md`), then `gh issue view <parent> --json title,body,comments`.
3. Identify new comments since the last heartbeat (look for comments after the last WoterClip-formatted comment).
4. Parse heartbeat counter: find the last comment matching `Heartbeat #N` pattern. Next comment will be `#N+1`. If none found, start at `#1`.

## Step 8: Do Work

Follow the persona's SOUL.md instructions. This step varies by persona:

**Orchestrator persona:** Triage the issue – apply persona labels, create sub-issues, or escalate. Never write code.

**CEO persona:** Make strategic decisions – prioritization, scope, architecture, coordination. Never write code.

**Worker personas (backend, frontend, etc.):**
- Use repo tools (Read, Write, Edit, Bash, Grep, Glob) to implement changes
- For large scope: decompose into GitHub sub-issues following `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md` (create with `--assignee @me` + persona label + `Parent: #N` body reference, attach by issue ID with `-F sub_issue_id=`, verify the attach, summarize on the parent)
- For small scope: work directly, use internal tasks to track progress
- Commit changes with descriptive conventional commit messages
- Respect `max_turns` from persona config as a work budget

**If `gh` auth expires or GitHub API errors persist mid-work:** Stop immediately. Leave `agent-working` label in place (will be cleaned as stale on next heartbeat). Delete lockfile and exit with error log.

## Step 9: Report

Post a structured comment on the GitHub issue: `gh issue comment N --repo <owner/name> --body "..."`.

Follow the comment format from `${CLAUDE_PLUGIN_ROOT}/references/comment-format.md`:
- Include `Heartbeat #N` counter (incremented from step 7)
- Include timestamp and duration
- Include persona name in footer
- List commits with SHAs, sub-issues created, and next steps
- For blocked status: @-mention who needs to act (Board user's login from config `github.board_user`)

Append heartbeat metadata to `.woterclip/heartbeat-log.jsonl`:
```json
{"heartbeat": N, "timestamp": "ISO", "issue": "#12", "persona": "name", "duration_sec": N, "status": "in_progress|completed|blocked", "actions": ["description"]}
```

## Step 10: Update State

Read the issue's current labels (`gh issue view N --json labels`), then update based on outcome:

| Outcome | Commands (in order) |
|---------|--------|
| **Completed** | `gh issue edit N --remove-label agent-working --remove-label in-progress` (labels first — `gh issue close` does not touch labels, and stale-label cleanup only scans open issues), then `gh issue close N --comment "..."` — or, if a PR was opened, swap to review instead: `gh issue edit N --remove-label agent-working --remove-label in-progress --add-label in-review` (issue stays open) |
| **Blocked** | One combined edit: `gh issue edit N --remove-label agent-working --add-label agent-blocked` (stays open) |
| **More work needed** | Keep `agent-working`, ensure `in-progress`: `gh issue edit N --add-label in-progress` (stays open) |

If any `--add-label` fails because the label doesn't exist on the repo, create it and retry (see `${CLAUDE_PLUGIN_ROOT}/references/label-conventions.md` § Label Operations) — do not skip the transition.

For blocked issues: @-mention the Board user's GitHub login in the comment text (e.g., "**@board-login** — please review"). Note: GitHub does not notify a user of their own comments — if `github.board_user` is the same account gh is authenticated as, the mention will NOT produce a notification; the Board must watch the repo or check `/woterclip-status` for blocked items.

## Step 11: Next Issue or Exit

1. If issues worked this heartbeat < `max_issues_per_heartbeat`, return to **Step 2** to pick the next issue.
2. Otherwise, delete lockfile and exit.
3. If 0 todo issues remain in queue, suggest pausing the schedule.
4. If 3+ issues are blocked, suggest Board attention rather than more heartbeats.
