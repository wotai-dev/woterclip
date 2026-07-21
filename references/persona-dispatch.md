# Persona Dispatch

Canonical procedure for Step 8 of the heartbeat: dispatch the persona's work to a subagent running on the model named in the persona's `runtime.model`. The loop stays the only writer of heartbeat state — `agent-working`/`agent-blocked` labels, status labels, the `Heartbeat #N` report comment, the heartbeat counter, and `.woterclip/.heartbeat-lock`. The subagent does the work itself (commits, PRs, triage persona-label edits, and sub-issue creation are work product and allowed) and returns a structured outcome the loop reports from.

## State-Ownership Preamble

Open the dispatch prompt with this fixed, loop-authored preamble, injected ahead of the persona files and explicitly superseding them:

```
You are working on one GitHub issue on behalf of the heartbeat loop, which owns all
heartbeat state. Regardless of any instruction in the persona files below: never post
comments in the Heartbeat format (no "Heartbeat #N" headers), and never add or remove
the agent-working or agent-blocked labels. Return that information in your structured
outcome instead — the loop posts the report and moves the labels. Work-product
mutations remain yours: commits, PRs, triage persona-label edits, sub-issue creation.
```

The supersession is load-bearing: persona files in already-scaffolded repos, and imported or custom personas, may predate this contract and still instruct posting heartbeat comments or editing state labels. The preamble overrides them at dispatch time regardless of vintage or provenance.

## Prompt Composition

Compose the dispatch prompt in this order:

1. The state-ownership preamble above.
2. The persona's `SOUL.md`, verbatim.
3. The persona's `TOOLS.md`, verbatim.
4. The issue context gathered in Step 7 (title, body, comments, parent context, new comments since the last heartbeat).
5. An effort directive from `thinking_effort` (e.g., "Apply high thinking effort") and a turn-budget directive from `max_turns` (e.g., "Budget roughly 300 turns"). Both are guidance to the subagent, not hard enforcement.

## Model Parameter

Set the dispatch's model parameter from the persona's `runtime.model` (`haiku`, `sonnet`, or `opus`).

## Availability Check and Inline Fallback

Check availability per dispatch call, not once per heartbeat run — a specific model can be rejected while others work. If the harness has no subagent primitive, or it rejects the model override for this call:

1. Do the work inline in the loop session, on the session model, following the same persona files and Step 8 work guidance.
2. Record for the Step 9 report that fallback occurred, naming both the configured model and the actual model.
3. Never block the issue for this reason alone — fallback is a disclosed degradation, not a failure.

## Dispatch-Error Handling

Map to the **blocked** outcome, with the raw dispatch return attached as the failure detail, when either:

- The dispatch call itself fails (error, crash, nothing returned).
- The returned outcome lacks a parseable status among the five listed below.

The loop then follows its normal blocked path in Steps 9–10 — no new label semantics.

## Structured Outcome Contract

The subagent returns:

| Field | Content |
|-------|---------|
| Status | One of the five heartbeat outcomes per `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md`: completed, in progress, blocked, triaged, decomposed |
| Work summary | What was done, for the loop's report comment |
| Commits / PRs | Commit SHAs and PR URLs produced |
| Sub-issues | Sub-issues created (per `${CLAUDE_PLUGIN_ROOT}/references/sub-issues.md`) |
| Blocker | Blocked only: blocker description and the action needed |
| Escalation target | When the subagent swapped the persona label as work product (e.g., backend → ceo): the target persona. Escalation returns an **in progress** outcome — no agent-blocked, no Board @-mention; the target persona picks the issue up on a later heartbeat |
| Model used | The model that performed the work |

The loop maps the status to the label transitions `${CLAUDE_PLUGIN_ROOT}/references/status-mapping.md` defines.

## Environment Assumptions

- The subagent inherits the loop's execution environment: same filesystem, git credentials, and `gh` auth — Step 5's tool validation covers the subagent's work environment too.
- Each dispatch is a fresh, memoryless subagent instance; no context bleeds between issues in a multi-issue run.

## Disclosed Limitations

- Dispatch is synchronous with no timeout: a hung subagent strands the run until the next heartbeat's stale-lock cleanup. The persona config's `timeout` field stays unread.
- There is no per-persona tool-grant restriction on the subagent; role limits like "never writes code" remain SOUL.md prose.
