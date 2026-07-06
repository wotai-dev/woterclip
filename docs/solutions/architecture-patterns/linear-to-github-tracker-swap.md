---
module: tracker-backend
tags: [github-issues, gh-cli, linear, migration, labels, sub-issues]
problem_type: platform-migration
---

# Swapping a tracker backend: Linear → GitHub Issues (lessons from #1)

WoterClip's tracker backend moved from Linear (MCP tools) to GitHub Issues (`gh` CLI) in
[#1](https://github.com/wotai-dev/woterclip/issues/1) /
[PR #2](https://github.com/wotai-dev/woterclip/pull/2). The swap looked mechanical —
every reviewer pass found it mostly was — but the bugs that survived into review all came
from the same root cause: **concepts that don't map 1:1 between trackers fail silently,
not loudly**. Documented here so the next backend/platform swap starts from these checks.

## The non-obvious gh/GitHub facts that caused real findings

- **`gh api -f` sends strings; `-F` sends typed values.** The sub-issues endpoint
  (`POST /repos/{o}/{r}/issues/{n}/sub_issues`) requires an integer `sub_issue_id` —
  `-f sub_issue_id=$id` 422s. Every `gh api` write with a non-string parameter needs `-F`.
- **Sub-issue attach is keyed by issue ID, not issue number.** `gh api .../issues/<number>
  --jq .id` first, then attach. Confusing the two fails on other people's repos where
  IDs and numbers diverge wildly.
- **GitHub does not auto-assign issue creators.** Any inbox model built on
  `--assignee @me` must pass `--assignee @me` at creation, or decomposed work silently
  vanishes from the agent's queue.
- **`gh issue close` does not touch labels**, and any "stale label cleanup" that scans
  open issues will never see a closed one — remove state labels *before* closing.
- **`gh issue edit --add-label` errors on labels that don't exist** (`'X' not found`) —
  it never creates them. Every label an agent writes must be created at init time (or
  guarded with a create-and-retry fallback).
- **GitHub never notifies a user of their own comments.** An @-mention escalation model
  breaks in the common single-account setup (agent authenticated as the human it mentions).
  Either recommend a bot account or document the gap — don't claim "notifies them".

## The concept-mapping traps

- **States**: Linear's six states collapse to open/closed + status *labels*
  (`backlog`/`todo`/`in-progress`/`in-review`). Split them by writer: agent-written labels
  are required infrastructure (create unconditionally); human-written ones are optional.
- **Priority**: no native field on GitHub — a `priority:high`/none/`priority:low` label
  scheme loses precision vs Linear's five levels; say so explicitly in the mapping doc.
- **Label groups**: don't exist on GitHub; flat names, so drop group config keys in the
  schema migration rather than emulating them.
- **Read-modify-write → atomic**: GitHub's `--add-label/--remove-label` are atomic per
  call — strictly safer than Linear's full-array save, but mutually-exclusive transitions
  still need to be *one combined* edit, or a mid-sequence failure violates the invariant.

## The migration lesson (biggest catch of the review)

A config-schema migration is not done when the *config file* migrates. The v1→v2
migration originally carried `personas/*` over "unchanged" — but v1 persona configs pin
`required_tools: [mcp__claude_ai_Linear]`, which the new tool validation treats as an
unavailable executable, **auto-blocking every issue after an apparently successful
migration**. Rule: when a migration changes what a config *value* means (here:
required_tools entries went from MCP prefixes to executables), sweep every scaffolded
file that carries such values, and add a post-migration check that greps for the old
vocabulary. Also: run the version check on *every* re-init path — a "merge (skip existing
files)" default that skips the migrated file bypasses the migration entirely.

## Process notes that paid off

- Centralizing the multi-step sub-issue procedure in one reference file
  (`references/sub-issues.md`) turned four drift-prone restatements into one — and let a
  single edit fix the `-f`/`-F` and `--assignee` bugs everywhere at once.
- The review's adversarial persona (prompted to attack the *procedure as literally
  executed*, step by step, under mid-procedure failures) found more real issues than any
  other lens — for instruction-prose repos, "what happens when step N's command fails"
  is the highest-yield question.
