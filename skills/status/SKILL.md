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

Report whether a recurring heartbeat is active. Check if `/schedule` is running `/heartbeat` by noting this is informational — the user knows their schedule state.

### Step 3: Last Heartbeat

Read the last line of `.woterclip/heartbeat-log.jsonl` (if it exists). Report:
- Heartbeat number, timestamp, and how long ago it ran
- Which persona and issue were involved
- Outcome (completed, in progress, blocked)

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
- `+` Newly created sub-issues

**Queue** (next heartbeat would pick these up):
- Open issues with persona labels, not labeled `backlog` or `in-review`, sorted `in-progress` first then by `priority:*` label
- Show: issue number, persona label, status label, priority label, title

**Blocked** (needs Board attention):
- Issues with `agent-blocked` label
- Show: issue number, Board user mention, blocker summary from last agent comment

### Step 5: Format Output

```
WoterClip Status
────────────────
Last beat:    Heartbeat #N — X min ago

Since last heartbeat:
  ✓ #12  [persona]   Completed    "Title"
  → #13  [persona]   In Progress  "Title"
  ✗ #14  [persona]   Blocked      "Title"

Queue (next heartbeat):
  #15  [persona]  Status  Priority  "Title"

Blocked (needs Board):
  #14  @board-login — blocker summary
```

## History Mode

When `--history` is passed, read `.woterclip/heartbeat-log.jsonl` and display the last 10 entries:

```
Heartbeat History (last 10)
───────────────────────────
#N  HH:MM  persona  #12  Status      (duration)
#N  HH:MM  persona  #15  Status      (duration)
```

If the log file doesn't exist or is empty, report "No heartbeat history found."
