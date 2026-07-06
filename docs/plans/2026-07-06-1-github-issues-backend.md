# Plan: swap Linear for GitHub Issues (#1)

**Date:** 2026-07-06 · **Tier:** safety-critical · **Issue:** [#1](https://github.com/wotai-dev/woterclip/issues/1)
**Brainstorm:** [docs/brainstorms/2026-07-06-github-issues-backend.md](../brainstorms/2026-07-06-github-issues-backend.md)

Every unit is safety-critical tier → `Execution note: validate-first` (state the observable
check, make the edit, run the check).

## Unit 1 — Config schema v2

`Execution note: validate-first` — check: `templates/config.yaml` parses and contains
`version: 2`, a `github:` section, no `linear:` section, no `labels.group`.

- `templates/config.yaml`: `linear:` → `github:` (`board_user`, `repo`), drop `project`,
  drop `labels.group`, `version: 1` → `2`, update comments.
- `{{TEAM}}` → `{{REPO}}` placeholder.

## Unit 2 — References rewrite

`Execution note: validate-first` — check: no `Linear`/`mcp__claude_ai_Linear` in
`references/*.md`; every `gh` invocation names a real subcommand/flag.

- `references/status-mapping.md`: Linear states table → GitHub open/closed + status-label
  model (backlog/todo/in-progress/in-review labels; closed completed/not-planned). Inbox
  query: `gh issue list --assignee @me --state open`.
- `references/label-conventions.md`: flat labels (no group), atomic ops via
  `gh issue edit --add-label/--remove-label` replacing read-modify-write, persona-label
  table unchanged in spirit.
- `references/comment-format.md`: `save_comment` → `gh issue comment`, `WOT-XX` → `#N`,
  Board mention = real `@login`, counter still parsed from comments
  (`gh issue view N --json comments`).

## Unit 3 — Heartbeat skill

`Execution note: validate-first` — check: frontmatter intact (`name`, `description`);
no Linear references; each step names concrete `gh` commands; every exit path still
deletes the lockfile.

- Steps 2/6/7/9/10: Linear MCP calls → `gh issue list/view/edit/comment/close`.
- Step 5 tool validation: MCP prefix check → `gh auth status` (and per-persona
  `required_tools` as executables).
- Step 8: sub-issue creation via `gh issue create` + `gh api` sub_issues attach.
- Mid-work failure note: "Linear MCP becomes unavailable" → "gh auth expires / API errors".
- `heartbeat-log.jsonl` example: `"issue": "WOT-XX"` → `"issue": "#12"`.
- Description/trigger phrases: "process Linear issues" → "process GitHub issues".

## Unit 4 — Init skill

`Execution note: validate-first` — check: prerequisites reference only `gh`; migration
section covers v1→v2; label-creation step has no group/parentId concept.

- Prerequisites: Linear MCP check → `gh auth status` + `gh repo view` (in a repo with a
  GitHub remote).
- Step 1: teams/users → `gh api user --jq .login` + `gh repo view --json nameWithOwner`.
- Step 3: `create_issue_label` + parent group → `gh label create` (flat, with colors,
  `--force`-less; skip existing).
- Step 4: `{{TEAM}}` → `{{REPO}}` replacement.
- Re-initialization: add explicit v1→v2 migration (board_user asked fresh — never guess a
  GitHub login from a Linear display name; team discarded; group dropped; version bumped).

## Unit 5 — Orchestrator agent

`Execution note: validate-first` — check: frontmatter `tools:` lists no MCP tools;
procedure uses `gh` throughout.

- `agents/orchestrator.md`: tools → `Bash`, `Read`, `Grep`; every MCP call in the triage
  procedure → `gh` equivalent; sub-issue creation via REST attach; config keys renamed.

## Unit 6 — Persona templates

`Execution note: validate-first` — check: all four persona `config.yaml` files parse;
`required_tools` lists `gh` not MCP prefixes; TOOLS.md files name `gh` commands.

- `templates/personas/{orchestrator,ceo,backend,frontend}/TOOLS.md` + `config.yaml`.
- `templates/personas/ceo/SOUL.md` Linear mentions.

## Unit 7 — Remaining skills + commands

`Execution note: validate-first` — check: repo-wide grep for `linear`/`Linear` hits only
`docs/specs/*`, the brainstorm/plan docs, and deliberate "formerly Linear" migration notes.

- `skills/status/SKILL.md`, `skills/persona-create/SKILL.md`,
  `skills/persona-import/SKILL.md`, `skills/persona-list/SKILL.md`,
  `commands/woterclip-init.md` (+ any other command frontmatter naming Linear).

## Unit 8 — Manifests + top-level docs

`Execution note: validate-first` — check: both manifest JSONs parse (`python3 -m json.tool`);
README quickstart works against GitHub; CLAUDE.md/AGENTS.md describe the GitHub backend.

- `plugin.json` / `marketplace.json`: description "Linear-backed" → "GitHub Issues-backed",
  keyword `linear` → `github-issues`. No version bump (release PR concern).
- `README.md`: prerequisites (gh CLI, not Linear MCP), label setup, quickstart.
- `CLAUDE.md`: "Linear-backed" description, core-loop diagram "(Linear)" → "(GitHub)",
  tracker line, label/heartbeat conventions, `{{TEAM}}`→`{{REPO}}` template note.
- `AGENTS.md`: runtime-tracker note (no longer "Linear in target repos") + safety-critical
  wording. Also commits the previously-untracked AGENTS.md + .gitignore change.

## Unit 9 — Validation pass

Full checklist from AGENTS.md: YAML parse on all `.yaml`, frontmatter on all skills/
commands/agents, `${CLAUDE_PLUGIN_ROOT}` references resolve, local plugin-load smoke check,
final repo-wide Linear grep (expect, per Unit 7: docs/specs, the brainstorm/plan docs, and
deliberate "formerly Linear" migration notes only).
