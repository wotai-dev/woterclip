# Sub-Issues

Canonical procedure for decomposing an issue into GitHub sub-issues. Every skill, agent, and persona that creates sub-issues follows this reference — do not restate the commands elsewhere.

All commands target the repo from config `github.repo` (pass `--repo <owner/name>` explicitly).

## Create and Attach

For each sub-issue:

1. **Create it** — assigned to the agent's own account so future heartbeats can see it (the inbox query is `--assignee @me`; GitHub does not auto-assign the creator), labeled for its persona, with the parent referenced in the body:

   ```bash
   gh issue create --repo <owner/name> \
     --title "Clear, actionable title" \
     --body "Scope and context. Parent: #<parent-number>" \
     --label <persona-label> \
     --assignee @me
   ```

   The `Parent: #N` body reference is required — it is the fallback parent link if the attach step fails, and the primary way step 7 of the heartbeat resolves parent context.

2. **Resolve the sub-issue's ID** (the attach endpoint takes the issue **ID**, not its number):

   ```bash
   sub_id=$(gh api repos/<owner>/<name>/issues/<sub-number> --jq .id)
   ```

3. **Attach it to the parent** — `-F` (typed field), not `-f`: the API requires an integer `sub_issue_id` and rejects the string a raw `-f` field would send:

   ```bash
   gh api repos/<owner>/<name>/issues/<parent-number>/sub_issues -F sub_issue_id=$sub_id
   ```

4. **Verify the attach succeeded** (non-zero exit or error body = failure). On failure, do not retry blindly — post a comment on the parent naming the sub-issues that were created but not attached (`Created but unattached: #X, #Y`) so the Board can attach them manually. The body's `Parent: #N` reference keeps the relationship discoverable either way.

5. **Report the decomposition in your returned outcome** — every created sub-issue (`#N — description (persona)`); the heartbeat loop's report comment lists them. Only when operating outside a dispatched heartbeat (e.g., the standalone orchestrator agent), post the same summary as a parent comment instead — never in the Heartbeat format.

## Priority

Sub-issues inherit the parent's `priority:*` label when present; blocking sub-issues get `priority:high`.

## Parent Completion Check

Before closing a parent because "all sub-issues are done":

```bash
gh api repos/<owner>/<name>/issues/<parent-number>/sub_issues --jq '.[].state'
```

- **Never close the parent when this list is empty** — an empty list means no children are attached (possibly because attach failed), not that all children are done.
- Cross-check the list against the parent's decomposition summary comment; if any sub-issue named there is missing from the attached list, treat the decomposition as incomplete and escalate instead of closing.
- Close only when the attached list is non-empty and every state is `closed`.

## Parent Lookup (from a sub-issue)

To find a sub-issue's parent, read the `Parent: #N` reference in its body (guaranteed by step 1 above). The REST issue payload may also carry parent/sub-issue metadata (`sub_issues_summary`), but the body reference is the reliable, version-independent mechanism — prefer it.
