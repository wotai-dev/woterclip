# Beat Economics

Defines what a beat costs, what bounds it, and what it records. A **beat** is one heartbeat
execution cycle — the loop from picking up an issue through to exit, which may work several
issues.

Consumed by `skills/heartbeat/SKILL.md` (captures the clock, enforces the ceiling, writes the
log), `skills/status/SKILL.md`, and `skills/heartbeat-log/SKILL.md` (both read the log).

## Clock Capture

The beat captures a start timestamp as its **first action in Step 1**, before reading config:

```bash
BEAT_START=$(date -u +%s)
```

Elapsed seconds at any later point is `$(date -u +%s) - BEAT_START`. Anchoring at beat start
rather than at lock acquisition keeps the measured window independent of step ordering.

This capture is the only source of elapsed time. Before it existed, `duration_sec` appeared in
the log schema but no step produced it — every value was absent, and any average over it was an
average of nothing.

## Time Ceiling

| Property | Value |
|----------|-------|
| Default | `1200` seconds (20 minutes) |
| Where enforced | Step 11, alongside the existing `max_issues_per_heartbeat` check |
| Configurable | No — changing it is a plugin edit to this file and Step 11, not a config key |

**The default is an initial estimate, not a measured value.** It was chosen before any real beat
durations existed, because `duration_sec` was never produced. Revise it once
`.woterclip/heartbeat-log.jsonl` holds real `beat_elapsed_sec` values across a range of repos.

**The ceiling bounds intake, not spend.** It is evaluated only between issues. A beat that
crosses the ceiling mid-issue finishes that issue and declines the next, so worst-case beat cost
is the ceiling plus one full persona dispatch. Interrupting a dispatch would strand state the
loop owns and the subagent has not yet returned through its structured outcome.

## Stop Reasons

A closed set. Every beat records exactly one.

| Value | Meaning |
|-------|---------|
| `complete` | Worked every eligible issue; nothing left in the queue |
| `queue_empty` | No eligible issues at pick-up time; no work performed |
| `issue_budget` | Reached `max_issues_per_heartbeat` with issues still eligible |
| `time_ceiling` | Reached the time ceiling with issues still eligible |
| `blocked_exit` | Stopped early — a required tool was unavailable, or `gh` failed mid-work |
| `quiet_hours` | Quiet hours active with `behavior: "skip"` |
| `lock_conflict` | Another beat held a live lockfile |

When both `issue_budget` and `time_ceiling` are reached at the same decision point, record
`time_ceiling` — it is the newer and tighter bound, and recording it makes ceiling tuning visible.

## Scheduling Preconditions

Recorded here because they decide which primitive the docs recommend.

| Property | Cron schedule | Self-paced loop |
|----------|---------------|-----------------|
| Survives a closed session | Yes | No — in-session only |
| Recovers from a beat that died mid-work | Yes — the next tick fires regardless | **Unverified** |
| Cadence adapts to queue depth | No | Yes |
| Maximum wake delay | The configured interval | Model-chosen, unbounded |

**Cron is the recommended unattended default**, on the session-independence row alone: a
self-paced loop cannot span a closed session, so it cannot run unattended for days no matter how
it handles errors.

**The error-recovery row is unverified.** Confirming it needs a live scaffolded repo running a
self-paced loop with a beat forced to fail, which has not been performed. Until it is, do not
recommend self-pacing for unattended operation — a self-paced loop schedules its own successor,
so a beat that dies may take the loop with it, and nothing would report that it had stopped.

**Treat a quiet repo as stopped after ~2 hours** of no beat with a non-empty queue, under either
primitive. This is a reporting threshold for `/woterclip-status`, not an enforced timeout.

## Log Fields

`.woterclip/heartbeat-log.jsonl` carries two line kinds, distinguished by `type`.

**Issue lines** — one per issue worked, unchanged except that `duration_sec` is now produced:

```json
{"heartbeat": 7, "timestamp": "ISO", "issue": "#79", "persona": "backend", "duration_sec": 720, "status": "in_progress", "actions": ["committed a1b2c3d"]}
```

`duration_sec` is seconds spent on **that issue**. It is not the beat's cost.

**Beat lines** — exactly one per beat, appended last:

```json
{"type": "beat", "started_at": "ISO", "ended_at": "ISO", "beat_elapsed_sec": 845, "issues_worked": 2, "stop_reason": "time_ceiling"}
```

`beat_elapsed_sec` is the beat's cost. Exactly one beat line per beat means beat cost is never
ambiguous, even when several issues were worked.

### Reading Rules

- **A line with no `type` key is an issue line.** Every line written before this contract existed
  lacks `type`, so this rule makes old history readable without migration.
- **Treat a missing field as unavailable, never as zero.** A pre-contract line has no
  `duration_sec` value and no beat line accompanies it; render that as unavailable. Showing `0`
  would misreport a beat that ran for minutes as free.
- **A beat with no beat line is a pre-contract beat.** Report its cost as unavailable rather than
  summing its issue lines — issue durations exclude the loop's own overhead.
- Beats that exit with `queue_empty`, `quiet_hours`, or `lock_conflict` perform no issue work and
  therefore write a beat line with `issues_worked: 0` and no issue lines.
