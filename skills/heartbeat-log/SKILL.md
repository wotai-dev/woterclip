---
name: heartbeat-log
description: This skill should be used when the user asks to "show heartbeat log", "heartbeat history", "what has woterclip done", "show agent activity", "summarize heartbeats", or wants to analyze past heartbeat activity. Parses heartbeat-log.jsonl for summaries and analytics.
version: 0.1.0
---

# Heartbeat Log

Parse and summarize the WoterClip heartbeat log file (`.woterclip/heartbeat-log.jsonl`).

## Log Format

Each line is a JSON object in one of two kinds, distinguished by `type`. Full field
definitions live in `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md`.

**Issue line** — one per issue worked. A line with **no `type` key is an issue line**, which is
what makes every pre-contract line readable without migration:
```json
{"heartbeat": 7, "timestamp": "2026-03-25T10:15:00Z", "issue": "#79", "persona": "backend", "duration_sec": 720, "status": "in_progress", "actions": ["committed a1b2c3d", "created sub-issue #83"]}
```

**Beat line** — one per beat, carrying the beat's cost and why it stopped:
```json
{"type": "beat", "started_at": "2026-03-25T10:03:00Z", "ended_at": "2026-03-25T10:17:05Z", "beat_elapsed_sec": 845, "issues_worked": 2, "stop_reason": "time_ceiling"}
```

**Treat a missing field as unavailable, never as zero.** Lines written before this contract
carry no `duration_sec` value and have no accompanying beat line. Render those as `—`; counting
them as `0` would report a beat that ran for minutes as free, and would drag any average toward
zero.

## Procedure

### Step 1: Read Log

Read `.woterclip/heartbeat-log.jsonl`. If missing or empty, report "No heartbeat history found."

Parse each line as JSON. Handle malformed lines gracefully (skip with warning).

### Step 2: Summarize

**Recent activity** (default: last 10 heartbeats):

```
Heartbeat History
─────────────────
#7  10:15  backend   #79  In Progress  (12 min)
#6  09:45  backend   #81  Completed    (8 min)
#5  09:15  ceo       #80  Triaged      (4 min)
```

**Aggregate stats** (when asked for analytics or summary):

- Total heartbeats, and total beats (count of beat lines)
- Heartbeats per persona (breakdown)
- Average duration per persona — **over issue lines that carry `duration_sec` only**, stating how many lines were excluded as unavailable
- Total and average `beat_elapsed_sec` across beat lines
- Stop-reason breakdown (how many beats hit `time_ceiling` vs `issue_budget` vs `complete`)
- Completion rate (completed / total)
- Most active issues (by heartbeat count)
- Blocked issues and duration blocked

### Step 3: Filter Options

Support filtering when the user asks:
- **By persona**: "show backend heartbeats" → filter by `persona` field
- **By issue**: "show activity for #79" → filter by `issue` field
- **By date range**: "show today's heartbeats" → filter by `timestamp`
- **By status**: "show blocked heartbeats" → filter by `status`

## Notes

- The log file is append-only and informational — safe to truncate if it grows too large
- Heartbeat numbers are per-issue (derived from comments), not global sequence numbers
- Field definitions — including what `duration_sec` and `beat_elapsed_sec` each measure — are in `${CLAUDE_PLUGIN_ROOT}/references/beat-economics.md`
