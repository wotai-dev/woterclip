# Persona Dispatch

Canonical procedure for Step 8 of the heartbeat: dispatch the persona's work to a subagent running on the model named in the persona's `runtime.model`. The loop stays the only writer of heartbeat state — `agent-working`/`agent-blocked` labels, status labels, the `Heartbeat #N` report comment, the heartbeat counter, and `.woterclip/.heartbeat-lock`. The subagent does the work itself (commits, PRs, triage persona-label edits, and sub-issue creation are work product and allowed) and returns a structured outcome the loop reports from.

## State-Ownership Preamble

Open the dispatch prompt with this fixed, loop-authored preamble, injected ahead of the persona files and explicitly superseding them:

```
You are working on one GitHub issue on behalf of the heartbeat loop, which owns all
heartbeat state. Regardless of any instruction in the persona files below: never post
comments in the Heartbeat format (no "Heartbeat #N" headers); never add or remove the
agent-working or agent-blocked labels; never add or remove status labels (backlog,
todo, in-progress, in-review); never create or delete .woterclip/.heartbeat-lock.
Return that information in your structured outcome instead — the loop posts the report
and moves the labels. If gh becomes unavailable or GitHub API errors persist, stop and
return a blocked outcome naming the failure. Work-product mutations remain yours:
commits, PRs, triage persona-label edits, sub-issue creation.
```

The supersession is load-bearing: persona files in already-scaffolded repos, and imported or custom personas, may predate this contract and still instruct posting heartbeat comments or editing state labels. The preamble overrides them at dispatch time regardless of vintage or provenance.

## Prompt Composition

Compose the dispatch prompt in this order:

1. The state-ownership preamble above.
2. The persona's `SOUL.md`, verbatim.
3. The persona's `TOOLS.md`, verbatim.
4. The issue context gathered in Step 7 — title, body, the comment thread (marking which comments are new since the last heartbeat), and parent context. Include the thread once; do not repeat new comments as a separate block.
5. An effort directive from `thinking_effort` (e.g., "Apply high thinking effort") and a turn-budget directive from `max_turns` (e.g., "Budget roughly 300 turns"). Both are guidance to the subagent, not hard enforcement. When `enable_chrome` is true, add a note that browser tooling is available and expected for verification.
6. The GitHub coordinates the persona's work-product commands need: `github.repo` (owner/name) and `github.board_user` from `.woterclip/config.yaml`.
7. The Structured Outcome Contract section below, verbatim — the return shape the subagent must produce, including the atomic escalation-swap mechanics.

## Model Parameter

Set the dispatch's model parameter from the persona's `runtime.model` (`haiku`, `sonnet`, or `opus`). Dispatch a general-purpose subagent — never a named or plugin-registered agent type, whose own system prompt would bypass the preamble; persona identity comes only from the composed prompt.

## Inline Fallback

Attempt the dispatch directly — do not probe harness capabilities first. Judge availability per dispatch call, not once per heartbeat run: a specific model can be rejected while others work. If the attempt is refused at invocation time — the harness has no subagent primitive, or it rejects the model override for this call, before any subagent execution began:

1. Do the work inline in the loop session, on the session model, following the same persona files and Step 8 work guidance.
2. Record for the Step 9 report that fallback occurred, naming both the configured model and the actual model.
3. Never block the issue for this reason alone — fallback is a disclosed degradation, not a failure.

## Dispatch-Error Handling

Map to the **blocked** outcome, with the raw dispatch return attached as the failure detail, when either:

- The dispatch fails after subagent execution may have begun (error, crash, nothing returned).
- The returned outcome lacks a parseable status among the five listed below.

The discriminator between fallback and error is whether execution could have started: invocation-time refusal → inline fallback; failure once execution may have begun → blocked, because re-running the work inline could duplicate commits, PRs, or sub-issues. When indeterminate, treat as blocked and name the ambiguity in the failure detail. The loop then follows its normal blocked path in Steps 9–10 — no new label semantics.

## Structured Outcome Contract

The subagent returns:

| Field | Content |
|-------|---------|
| Status | One of the five heartbeat outcomes per `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md`, spelled exactly: completed, in progress, blocked, triaged, decomposed (logged as `in_progress` for the second) |
| Work summary | What was done, for the loop's report comment |
| Commits / PRs | Commit SHAs and PR URLs produced |
| Sub-issues | Sub-issues created (per `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md`) |
| Blocker | Blocked only: blocker description and the action needed |
| Escalation target | When the subagent swapped the persona label as work product (e.g., backend → ceo): the target persona. The swap is one atomic edit per `${CLAUDE_PLUGIN_ROOT}/references/label-conventions.md` — `gh issue edit N --remove-label <current> --add-label <target>`, never two calls, never two persona labels. Escalation returns an **in progress** outcome — no agent-blocked, no Board @-mention; the target persona picks the issue up on a later heartbeat |
| Model used | Advisory cross-check only — the loop's own dispatch parameter (or the session model on fallback) is the authoritative source for the report's Model line |

The loop maps the status to the label transitions `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md` defines. Before Steps 9–10 write anything, the loop re-checks that `.woterclip/.heartbeat-lock` still exists and the issue still carries `agent-working`; if a later heartbeat cleaned them as stale, log locally and exit without writing to the issue.

## Environment Assumptions

- The subagent inherits the loop's execution environment: same filesystem, git credentials, and `gh` auth — Step 5's tool validation covers the subagent's work environment too.
- Each dispatch is a fresh, memoryless subagent instance; no context bleeds between issues in a multi-issue run.

## Disclosed Limitations

- Dispatch is synchronous with no timeout: a hung subagent strands the run until the next heartbeat's stale-lock cleanup. The persona config's `timeout` field stays unread. The pre-write re-check above keeps a cleaned-up-then-resumed run from writing as a second reporter.
- There is no per-persona tool-grant restriction on the subagent; role limits like "never writes code" remain SOUL.md prose.
