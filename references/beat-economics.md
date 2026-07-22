# Beat Economics

Defines what a beat costs, what bounds it, and what it records. A **beat** is one heartbeat
execution cycle — the loop from picking up an issue through to exit, which may work several
issues.

Consumed by `skills/heartbeat/SKILL.md` (captures the clocks, enforces the ceiling, writes the
log), `skills/status/SKILL.md` and `skills/heartbeat-log/SKILL.md` (both read the log), and — for
the Scheduling Preconditions section only — `skills/init/SKILL.md` and `README.md`.

## The Lockfile: Identity and Clock

`.woterclip/.heartbeat-lock` is JSON, written once at step 3 by a command that also prints it:

```json
{"beat_id": "1784500980-4821", "started_at": "2026-07-22T10:03:00Z", "started_epoch": 1784500980}
```

It carries both the beat's **identity** and its **clock**, which makes one read answer two
questions: *do I still own this lock?* (does `beat_id` still match) and *how long have I run?*
(`$(date -u +%s)` minus `started_epoch`).

**Ownership rule.** Delete the lockfile only when it still carries this beat's `beat_id`. Missing
or different means a later beat cleaned and re-took it; leave it alone. A beat that never acquired the lock —
the `lock_conflict` and config-missing exits, both of which precede step 3 — deletes nothing.
`--dry-run` exits at step 3 *after* the lock was taken, so it does own its lock and must
release it; it simply records no beat line.

**Never hold a clock in a shell variable.** A skill is a procedure executed across separate tool
calls, and shell state does not survive between them: `BEAT_START=$(date -u +%s)` assigns a value
the beat can never read back, and comparing an empty value against the ceiling trips it on the
first check of every beat. Every clock value must be **printed** (`tee` for the lockfile, bare
`date -u +%s` for the per-issue clock) so it lands in output the beat can carry forward.

Two scopes, two clocks: the lockfile's `started_epoch` measures the whole beat and feeds the
ceiling and `beat_elapsed_sec`; a value printed at step 7 measures one issue and feeds
`duration_sec`. A beat working several issues cannot attribute its own elapsed time to any one of
them, which is why they are separate.

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

## Exit Paths and Stop Reasons

Every exit **that began a beat** records a beat line, then applies the ownership rule. This table
is the map — each exit in `skills/heartbeat/SKILL.md`, what it records, and whether it holds the
lock. Three exits began no beat and record nothing; two hold no lock.

| Exit | Where | Stop reason | `issues_worked` | Owns lock? |
|------|-------|-------------|-----------------|-----------|
| Config missing | Step 1 | *none — records nothing* | — | no |
| Live lockfile held by another beat | Step 1 | *none — records nothing* | — | **no** |
| Quiet hours, `behavior: "skip"` | Step 1 | `quiet_hours` | 0 | yes |
| `--dry-run` | Step 3 | *none — records nothing* | — | yes |
| No eligible issues, none worked | Step 3 | `queue_empty` | 0 | yes |
| No eligible issues, some worked | Step 3 | `complete` | count | yes |
| `gh` unavailable or unauthenticated | Step 5 | `blocked_exit` | count so far | yes |
| `gh` fails mid-work | Step 8 | `blocked_exit` | count so far | yes |
| Superseded by a later beat before reporting | Step 9 | `blocked_exit` | count so far | **no** |
| Ceiling reached, work remaining | Step 11 | `time_ceiling` | count | yes |
| Issue budget reached, work remaining | Step 11 | `issue_budget` | count | yes |
| Queue exhausted | Step 11 | `complete` | count | yes |

**A missing non-`gh` required tool is not a beat exit.** Step 5 blocks that issue and continues to
step 11; the beat's stop reason comes from step 11 as usual. Only `gh` itself being unavailable
ends the beat, because no GitHub mutation is possible without it.

**Three exits record nothing:** config-missing and `lock_conflict` (no beat began), and
`--dry-run` (no work was done). Two exits own no lock: `lock_conflict` never took one, and the
step 9 stand-down had its lock re-taken by a successor.

`queue_empty` is the most common beat in a quiet repo and the cheapest to record. It is also the
only evidence that a scheduled loop is alive and finding nothing, rather than stopped.

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
- **Treat a missing field as unavailable, never as zero. Render it as `—`.** Showing `0` would
  misreport a beat that ran for minutes as free, and would drag any average toward zero. Every
  reader uses this same token — do not substitute `n/a`, `null`, or a blank.
- **Group backwards: a beat line closes the group above it.** Beat lines are appended last, so a
  group is the run of issue lines up to and including the next beat line.
- **A trailing run of issue lines with no following beat line is either a pre-contract group or a
  beat that died mid-work.** Discriminate on `duration_sec`: present means the lines were written
  after this contract, so the beat **died** — render its cost as `—` and surface it as an
  incident, since a beat that never reached an exit is the failure a schedule is meant to
  recover from. Absent means pre-contract — render `—` without alarm. Never sum issue durations
  to synthesize a beat cost; they exclude the loop's own overhead.
- **If a group holds more issue lines than its beat line's `issues_worked`,** the excess belongs
  to an earlier died beat. Split it off and render it as died.
- Beats that exit with `queue_empty` or `quiet_hours` perform no issue work and therefore write a
  beat line with `issues_worked: 0` and no issue lines. `lock_conflict` and `--dry-run` write no
  beat line at all — no beat began, and no work was done.
