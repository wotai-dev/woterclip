# Comment Format

All heartbeat comments follow a structured template posted via `gh issue comment N --body`.

## Standard Template

```markdown
## Heartbeat #N — YYYY-MM-DD HH:MM UTC (duration)

**Status:** In Progress | Completed | Blocked | Triaged | Decomposed
**Model:** opus

<!-- Fallback form, used when dispatch was unavailable or the model override was
     rejected and the loop did the work inline on the session model:
     **Model:** sonnet (fallback — configured: opus) -->

### What was done
- [`a1b2c3d`](link) feat(api): commit message
- Description of non-commit work

### Created sub-issues
- #XX — Description (persona)

### What's next
- Next steps for this issue

### Blockers
None

---
*WoterClip · persona-name · #XX · from [Heartbeat #N-1](link)*
```

## Blocked Template

```markdown
## Heartbeat #N — YYYY-MM-DD HH:MM UTC (duration)

**Status:** Blocked
**Model:** opus

### Blocker
Clear description of what is blocking progress.

### Action needed
@board-login — specific ask for what they need to do.

**Clears when:** the observable condition that unblocks this issue.

### What was done before blocking
- Work completed before hitting the blocker

---
*WoterClip · persona-name · #XX*
```

## Rules

- Always include heartbeat counter (`#N`) and timestamp with duration
- Always include persona name and issue reference in footer
- Reference previous heartbeat comment link for carry-forward context
- Blocked comments must @-mention who needs to act (Board user's GitHub login from config
  `github.board_user`). Caveat: GitHub does not notify a user of their own comments — when
  the Board user is the same account gh is authenticated as, the mention is visible but
  produces no notification (the Board should watch the repo or use `/woterclip-status`)
- Blocked comments must carry a `**Clears when:**` line inside `### Action needed`, naming the
  observable condition that unblocks the issue — not a restatement of the blocker and not only
  who is being asked. "Clears when the Board grants repo access to the deploy key" is usable;
  "Clears when the Board responds" is not. `/woterclip-status` renders this line as each blocked
  item's exit condition, so a missing or vague one leaves the Board reading a queue that says
  what is stuck without saying what would unstick it
- Completion comments must list shipped commits/PRs with links
- Use `⚠️` flag for uncertain work that needs manual verification
- Fast-path triage comments: `**Triage:** → backend` for obvious routing
- Issue references are bare `#N` — GitHub auto-links them within the same repo
- The `**Model:**` line is informational for readers, not part of heartbeat-counter
  derivation

## Heartbeat Counter

The counter is **derived from GitHub comments**, not stored locally:

1. Fetch comments: `gh issue view N --json comments --jq '.comments[].body'`
2. Parse the last WoterClip comment for `Heartbeat #N`
3. Increment N for the new comment
4. If no previous comment exists, start at `#1`
5. If comments are deleted, counter resets — this is informational, not functional

## Footer Format

The footer line connects the comment to its context:

- `WoterClip` — identifies this as an agent comment
- `persona-name` — which persona produced this work
- `#XX` — the issue reference (auto-linked by GitHub)
- `from [Heartbeat #N-1](link)` — link to previous heartbeat comment (omit on first
  heartbeat; comment permalinks come from `gh issue view N --json comments --jq
  '.comments[].url'`)
