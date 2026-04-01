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

Call `mcp__claude_ai_Linear__list_issues` with `assignee: "me"`. Filter and categorize:

**Since last heartbeat** (issues that changed since the last logged heartbeat timestamp):
- `✓` Completed issues
- `→` In Progress issues
- `✗` Blocked issues
- `+` Newly created sub-issues

**Queue** (next heartbeat would pick these up):
- Issues with persona labels, status Todo or In Progress, sorted by priority
- Show: issue ID, persona label, status, priority, title

**Blocked** (needs Board attention):
- Issues with `agent-blocked` label
- Show: issue ID, Board user mention, blocker summary from last agent comment

### Step 5: Format Output

```
WoterClip Status
────────────────
Last beat:    Heartbeat #N — X min ago

Since last heartbeat:
  ✓ WOT-XX  [persona]   Completed    "Title"
  → WOT-XX  [persona]   In Progress  "Title"
  ✗ WOT-XX  [persona]   Blocked      "Title"

Queue (next heartbeat):
  WOT-XX  [persona]  Status  Priority  "Title"

Blocked (needs Board):
  WOT-XX  @User — blocker summary
```

## History Mode

When `--history` is passed, read `.woterclip/heartbeat-log.jsonl` and display the last 10 entries:

```
Heartbeat History (last 10)
───────────────────────────
#N  HH:MM  persona  WOT-XX  Status      (duration)
#N  HH:MM  persona  WOT-XX  Status      (duration)
```

If the log file doesn't exist or is empty, report "No heartbeat history found."
