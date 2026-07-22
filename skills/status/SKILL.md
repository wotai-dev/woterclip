---
name: woterclip-status
description: This skill should be used when the user asks to "check woterclip status", "show agent status", "what is woterclip doing", "show heartbeat status", "what's in the queue", or runs the /woterclip-status command. Shows current WoterClip state, issue queue, and blocked items.
version: 0.1.0
---

# WoterClip Status

Display the current state of WoterClip in this repository: schedule info, last heartbeat, issue activity, queue, and blocked items.

**Arguments:**
- `--history` — Show recent heartbeat history from the log file

## Status Procedure

### Step 1: Load Config

Read `.woterclip/config.yaml`. If missing, report that WoterClip is not initialized and suggest `/woterclip-init`.

### Step 2: Check Schedule

Determine schedule state at read time by asking the harness what recurring work it has registered for this repo (its schedule/cron listing, or the equivalent surface for the loop primitive in use).

Report one of three states, and never infer one from another:
- **Scheduled** — a recurring heartbeat is registered. Report its cadence.
- **Not scheduled** — the harness returned a listing and no heartbeat is in it.
- **Unknown** — the harness exposes no way to enumerate schedules, or the query failed.

Report `Unknown` rather than `Not scheduled` whenever the listing is unavailable. A false "not scheduled" tells the Board their automation is off when it may be running.

### Step 3: Last Beat

Read `.woterclip/heartbeat-log.jsonl` (if it exists). Line kinds and fields are defined in `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md`.

From the last **beat line**, report:
- When the beat ran and how long ago
- `beat_elapsed_sec` as the beat's cost, and `issues_worked`
- `stop_reason` — surface `time_ceiling` and `issue_budget` prominently, since both mean work was left on the queue

From the last **issue line**, report:
- Heartbeat number, persona, and issue
- Outcome (completed, in progress, blocked, triaged, decomposed)

**Absent fields render as `—`, never as `0`.** A line with no `type` key is an issue line from before beat lines existed; a beat with no beat line has an unavailable cost, not a zero one. Do not sum issue-line durations to synthesize a missing beat cost — issue durations exclude the loop's own overhead.

If no log file exists, report "No heartbeat history found."

### Step 4: Current Issues

Run two queries against the repo from config `github.repo`:
- Open issues (queue + blocked): `gh issue list --repo <owner/name> --assignee @me --state open --json number,title,labels`
- Recently completed (closed since the last logged heartbeat timestamp): `gh issue list --repo <owner/name> --assignee @me --state closed --search "closed:>=<last-heartbeat-ISO-date>" --json number,title,labels,closedAt`

Filter and categorize:

**Since last heartbeat** (issues that changed since the last logged heartbeat timestamp):
- `✓` Completed issues
- `→` In Progress issues
- `✗` Blocked issues
- `↪` Triaged issues (Orchestrator routed to a persona; unclaimed until that persona's heartbeat)
- `+` Newly created sub-issues (decomposed parents show these as children)

**Queue** (next heartbeat would pick these up):
- Open issues with persona labels, not labeled `backlog` or `in-review`, sorted `in-progress` first then by `priority:*` label
- Show: issue number, persona label, status label, priority label, title

**Blocked** (needs Board attention):
- Issues with `agent-blocked` label
- Show: issue number, Board user mention, blocker summary from last agent comment, and the **exit condition** — the `**Clears when:**` line from that comment's `### Action needed` section (see `${CLAUDE_PLUGIN_ROOT}/references/comment-format.md`)
- If the comment carries no `**Clears when:**` line, show `Clears when: not stated` rather than repeating the blocker summary. Silently reusing the blocker text would make an unanswered question look answered

### Step 5: Format Output

```
WoterClip Status
────────────────
Schedule:     Scheduled (every 30m) | Not scheduled | Unknown
Last beat:    Heartbeat #N — X min ago — 14m 5s, 2 issues, stopped: time_ceiling

Since last heartbeat:
  ✓ #12  [persona]   Completed    "Title"
  → #13  [persona]   In Progress  "Title"
  ✗ #14  [persona]   Blocked      "Title"
  ↪ #16  [ceo]       Triaged      "Title"
  + #17  [persona]   Sub-issue    "Title" (parent #12)

Queue (next heartbeat):
  #15  [persona]  Status  Priority  "Title"

Blocked (needs Board):
  #14  @board-login — blocker summary
       Clears when: observable condition that unblocks it
```

## History Mode

When `--history` is passed, read `.woterclip/heartbeat-log.jsonl` and display the last 10 beats, each with its issue lines. Render an absent field as `—`:

```
Heartbeat History (last 10 beats)
─────────────────────────────────
14m 5s   2 issues  stopped: time_ceiling
  #N  HH:MM  persona  #12  Status  (12m 1s)
  #N  HH:MM  persona  #15  Status  (2m 4s)
8m 12s   1 issue   stopped: complete
  #N  HH:MM  persona  #81  Status  (8m 0s)
—        1 issue   stopped: —            ← pre-contract beat, cost unavailable
  #N  HH:MM  persona  #79  Status  (—)
```

If the log file doesn't exist or is empty, report "No heartbeat history found."
