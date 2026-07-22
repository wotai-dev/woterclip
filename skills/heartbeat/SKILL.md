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
- `${CLAUDE_PLUGIN_ROOT}/references/persona-dispatch.md` — Subagent dispatch, outcome contract, and fallback rules
- `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md` — Clock capture, time ceiling, stop reasons, log fields

All `gh issue` / `gh api` calls below target the repo from config `github.repo` (pass `--repo <owner/name>` explicitly — never rely on the working directory's default remote).

## Step 1: Load Config & Lock

1. Read `.woterclip/config.yaml`. If missing, stop and instruct the user to run `/woterclip-init`. Nothing is created yet, so this exit records nothing.
2. Check for lockfile at `.woterclip/.heartbeat-lock`.
   - Exists and **less than** `stale_lock_hours` old → stop: "Previous heartbeat still active. Skipping." **This beat never took the lock — do not delete it and do not record a beat line.**
   - Exists and **older than** `stale_lock_hours` → delete it, log: "Cleaned stale lockfile."
   - No lockfile → proceed.
3. Take the lock. This one command creates it **and prints it**, so the beat observes its own id and start epoch:
   ```bash
   printf '{"beat_id":"%s","started_at":"%s","started_epoch":%s}\n' \
     "$(date -u +%s)-$$" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$(date -u +%s)" \
     | tee .woterclip/.heartbeat-lock
   ```
   Carry the printed `beat_id` and `started_epoch` forward — they are the beat's identity and its clock.
4. **Ownership rule, applied at every exit from here on.** Re-read `.woterclip/.heartbeat-lock`. Delete it **only if it still carries this beat's `beat_id`**; if it is missing or carries another id, a later beat cleaned and re-took it — leave it alone. Deleting a lock this beat does not own hands two beats the same repo.
5. **Every exit from here on** records one beat line (step 9 format) naming that exit's stop reason, then applies the ownership rule. `--dry-run` records none — no beat's work was done. The exit-to-reason map is in `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md`. Beat elapsed is `$(date -u +%s)` minus `started_epoch`.

Check quiet hours: if `quiet_hours.enabled` and current time is within the quiet window:
- `behavior: "skip"` → record a `quiet_hours` beat line, apply the ownership rule, exit: "Quiet hours active. Skipping."
- `behavior: "triage-only"` → proceed but only load Orchestrator persona (skip worker personas in step 3).

## Step 2: Check Inbox

1. Fetch open issues assigned to the authenticated user:
   ```bash
   gh issue list --repo <owner/name> --assignee @me --state open \
     --json number,title,labels,createdAt --limit 100
   ```
2. Filter and sort client-side per `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md` § Filter Rules and § Sort Order (labels come from the JSON above).
3. Detect stale `agent-working` labels: if an issue has `agent-working` but no heartbeat comment within `stale_lock_hours`, clean the stale label (`gh issue edit N --remove-label agent-working`, post cleanup comment).

## Step 3: Pick Issue

1. If `--persona <name>` flag is set, filter to only issues matching that persona's label.
2. Pick the first issue from the sorted inbox.
3. If `--dry-run`, report what would be picked, then apply the ownership rule and exit. No beat occurred, so record **no** beat line:
   ```
   Dry run — would pick:
     #12 [backend] "Issue title" (in-progress, priority:high)
   Queue:
     #15 [frontend] "Other issue" (todo)
   ```
4. If no issues match → record a beat line (`queue_empty` when zero issues were worked this beat, otherwise `complete` with the running count), apply the ownership rule, exit: "No issues in queue. Heartbeat complete."

## Step 4: Resolve Persona

1. Read the issue's labels. Find the persona label by matching against the `personas` map in config.
2. No persona label found → load the persona with `is_default: true` (typically Orchestrator).
3. Load persona files from `.woterclip/<persona.path>/`:
   - `SOUL.md` → inject into context as identity instructions
   - `TOOLS.md` → inject into context as tool guidance
   - `config.yaml` → read runtime settings

The persona's `config.yaml` supplies `model`, `thinking_effort`, `max_turns`, and `enable_chrome`; how each feeds the step 8 dispatch is defined in `${CLAUDE_PLUGIN_ROOT}/references/persona-dispatch.md`.

## Step 5: Validate Tools

Read `required_tools` from persona config. Verify each entry by its kind:
- `gh` → verify with `gh auth status` (exit 0) — this proves both the CLI and authentication
- `mcp__*` entries (e.g., `mcp__neon`) → these are MCP tool prefixes, not executables: verify by checking whether any tool starting with that prefix is available in the current session. Never run `command -v` on an `mcp__*` name.
- Other executables (e.g., `docker`) → verify with `command -v <tool>`

**If `gh` itself is unavailable or unauthenticated:** no GitHub mutation is possible — do not attempt the blocked-comment path (it needs gh). Record a `blocked_exit` beat line, apply the ownership rule, and exit with an error message telling the user to run `gh auth login`. (Same rule as the mid-work failure in step 8.)

**If any other required tool is unavailable** → stop work on this issue immediately:
  - Post a blocked comment naming the missing tool
  - Apply `agent-blocked` label, remove `agent-working` if present:
    `gh issue edit N --add-label agent-blocked --remove-label agent-working`
  - Proceed to step 11 (next issue). **This is not a beat exit** — record no beat line and do not touch the lockfile; the beat's stop reason still comes from step 11.

## Step 6: Lock Issue

1. Read the issue's current labels: `gh issue view N --json labels --jq '.labels[].name'`.
2. If `agent-working` is already present (from a previous heartbeat on same issue), proceed.
3. Otherwise: `gh issue edit N --add-label agent-working` (atomic — no need to rewrite the label set).

## Step 7: Understand Context

1. Run `date -u +%s` and carry the printed value as this issue's start epoch — it covers the dispatch. Separate from the beat clock in the lockfile.
2. Read issue title, body, and all comments: `gh issue view N --json title,body,comments`.
3. If the issue is a sub-issue, read its parent for broader context: follow the `Parent: #N` reference in the issue body (guaranteed by the sub-issue creation procedure in `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md`), then `gh issue view <parent> --json title,body,comments`.
4. Identify new comments since the last heartbeat (look for comments after the last WoterClip-formatted comment).
5. Parse heartbeat counter: find the last comment matching `Heartbeat #N` pattern. Next comment will be `#N+1`. If none found, start at `#1`.

## Step 8: Do Work

Dispatch the work to a persona subagent per `${CLAUDE_PLUGIN_ROOT}/references/persona-dispatch.md` — prompt composition, the model parameter, fallback, error handling, and the outcome contract are all defined there. The branch shape:

1. Dispatch on the persona's `model`; the subagent returns a structured outcome that steps 9–10 consume.
2. Refused at invocation — no subagent primitive, or override rejected before execution began → work inline on the session model; record the fallback for the step 9 report (never block the issue for this reason alone).
3. Failure after execution may have begun, or unparseable outcome → treat as a **blocked** outcome (step 10 blocked path); when indeterminate, prefer blocked — re-running inline could duplicate work.

The work itself — wherever it runs — follows the persona's SOUL.md instructions and varies by persona:

**Orchestrator persona:** Triage the issue – apply persona labels, create sub-issues, or escalate. Never write code.

**CEO persona:** Make strategic decisions – prioritization, scope, architecture, coordination. Never write code.

**Worker personas (backend, frontend, etc.):**
- Use repo tools (Read, Write, Edit, Bash, Grep, Glob) to implement changes
- For large scope: decompose into GitHub sub-issues following `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md` (create with `--assignee @me` + persona label + `Parent: #N` body reference, attach by issue ID with `-F sub_issue_id=`, verify the attach, summarize in your returned outcome)
- For small scope: work directly, use internal tasks to track progress
- Commit changes with descriptive conventional commit messages
- Respect `max_turns` from persona config as a work budget

**If `gh` auth expires or GitHub API errors persist mid-work:** Stop immediately. Leave `agent-working` in place (cleaned as stale by a later beat). Record a `blocked_exit` beat line, apply the ownership rule, exit.

## Step 9: Report

Before posting, re-read the lockfile and confirm it still carries this beat's `beat_id`, and that the issue still carries `agent-working`. If a later beat superseded this one, record a `blocked_exit` beat line and exit **without posting the comment and without deleting the lock** — it belongs to the successor. If the report post itself fails with persistent gh errors, follow the step 8 mid-work rule.

Post a structured comment on the GitHub issue: `gh issue comment N --repo <owner/name> --body "..."`.

Follow the comment format from `${CLAUDE_PLUGIN_ROOT}/references/comment-format.md`:
- Include `Heartbeat #N` counter (incremented from step 7)
- Include timestamp and duration
- Include the `**Model:**` line naming the model that performed the work — the loop's dispatch parameter (session model on fallback, naming both configured and actual); the subagent's self-reported model is advisory only
- Include persona name in footer
- List commits with SHAs, sub-issues created, and next steps
- For blocked status: @-mention who needs to act (Board user's login from `github.board_user`) **and include the required `**Clears when:**` line** naming what will unblock the issue

Append an **issue line** to `.woterclip/heartbeat-log.jsonl` — one per issue worked. `duration_sec` is `date -u +%s` now, minus the start epoch printed at step 7 — seconds on this issue, not the beat's cost:
```json
{"heartbeat": N, "timestamp": "ISO", "issue": "#12", "persona": "name", "duration_sec": N, "status": "in_progress|completed|blocked|triaged|decomposed", "actions": ["description"]}
```

At **any** beat exit — step 3, 5, 8, 9, or 11 — append exactly one **beat line**. This is where beat cost lives. Field and stop-reason definitions are in `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md`:
```json
{"type": "beat", "started_at": "ISO", "ended_at": "ISO", "beat_elapsed_sec": N, "issues_worked": N, "stop_reason": "time_ceiling"}
```

## Step 10: Update State

Read the issue's current labels (`gh issue view N --json labels`), then update based on outcome:

Transitions are defined in `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md` § WoterClip Outcomes. Two ordering rules the loop must honor:

- **Completed** — remove `agent-working` and `in-progress` *before* `gh issue close` (close does not touch labels, and stale-label cleanup only scans open issues). If a PR opened instead, swap to `in-review` and leave the issue open.
- **Blocked** — one combined edit: `gh issue edit N --remove-label agent-working --add-label agent-blocked`.

If any `--add-label` fails because the label doesn't exist on the repo, create it and retry (see `${CLAUDE_PLUGIN_ROOT}/references/label-conventions.md` § Label Operations) — do not skip the transition.

For blocked issues: @-mention the Board user's GitHub login in the comment text (e.g., "**@board-login** — please review"). Note: GitHub does not notify a user of their own comments — if `github.board_user` is the same account gh is authenticated as, the mention will NOT produce a notification; the Board must watch the repo or check `/woterclip-status` for blocked items.

## Step 11: Next Issue or Exit

"Eligible issues remain" means the sorted inbox from step 2 minus issues worked this beat — do not re-query to answer these tests. The ceiling (1200s default) and stop reasons are defined in `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md`.

1. Elapsed ≥ ceiling with issues remaining → stop intake, reason `time_ceiling`.
2. Else issues worked ≥ `max_issues_per_heartbeat` with issues remaining → stop intake, reason `issue_budget`.
3. Else issues remain → return to **Step 2** for the next issue.
4. Else the queue is exhausted → reason `complete`.

Before exiting, add to the report: suggest pausing the schedule if 0 todo issues remain, and suggest Board attention rather than more heartbeats if 3+ issues are blocked. Then exit per the step 1 invariant.
