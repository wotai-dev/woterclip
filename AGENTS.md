<!--
  AGENTS.md for woterclip — bootstrapped 2026-07-06 from
  ~/.config/compound-engineering/AGENTS.md.template, calibrated against
  compound-engineering plugin version 3.18.0 (2026-07-06).

  This file is tracked in git (imported by CLAUDE.md via @AGENTS.md) — originally
  gitignored per the template default, un-ignored in #1 so the import resolves for
  every contributor.
  Repo-specific decisions baked in: GitHub Issues tracker (wotai-dev/woterclip,
  single-project — no Projects board), worktree branching via /ce-worktree, no UI
  (browser-testing block replaced by the plugin validation checklist), no test
  frameworks (this repo is entirely markdown/YAML — "tests" are the validation
  checklist + a local plugin-load smoke check).
-->

# This is a Claude Code plugin, not an app

There is no runtime code — every "feature" is markdown/YAML read by Claude Code's plugin
loader. Plugin conventions (manifest schema, SKILL.md frontmatter, `${CLAUDE_PLUGIN_ROOT}`
resolution, hook wiring) drift between Claude Code releases — verify against the current
plugin-dev docs, not training data. Note the two-level structure: this repo ships the
plugin; `/woterclip-init` scaffolds a per-repo `.woterclip/` in *target* repos. Edits here
propagate to every repo that installs the plugin.

## Development process

This repo uses the **compound-engineering** plugin (`/ce-*` skills) as the spine of every change, with **GitHub Issues** for issue tracking (single-project repo — plain Issues, no Projects board). Default end-to-end pipeline:

```
[GitHub Issue #N created]
  → /ce-worktree (branch: <type>/<issue-number>-<slug>)
  → /ce-brainstorm → /ce-plan   (authored on the branch — every brainstorm/plan/doc-review artifact ships in THIS unit's PR, never a standalone docs PR; see the callout under the diagram)
  → [/ce-doc-review on the plan doc — conditional, run for safety-critical or multi-unit plans]
  → /ce-work (commits reference #N)
  → [/ce-debug if bugs surface — conditional, loop until green]
  → /ce-simplify-code (clean up before review)
  → validation pass (YAML parse + frontmatter + reference resolution + local plugin load — see Validation checklist)
  → /ce-code-review
  → /ce-commit-push-pr (PR body includes "Closes #N" as the closing link)
  → /ce-resolve-pr-feedback — MANDATORY, covers human AND Copilot/automated reviewers: after the PR opens, wait for Copilot's review to land, then read + fix-or-reply + resolve EVERY Copilot thread here. Resolving Copilot threads is part of this step, not a separate later gate — the commit→push→PR cycle is not complete until they are all resolved
  → [merge PR — only after every Copilot/automated-reviewer thread is resolved] → #N auto-closes on merge
  → /ce-compound (writes to docs/solutions/)
```

Not every step fires every time — the tier rules below decide what runs. See `## Issue tracking` and `## Branching, commits, and PRs` below for the bookend mechanics.

**One PR in flight at a time — merge before you move on.** Don't pile up a stack of open PRs. Take each unit all the way through the pipeline above (brainstorm → … → merge → compound) and *merge it* — every human + Copilot/automated-reviewer thread resolved — before starting the next branch. Parallel branches/worktrees are for isolation, not for hoarding un-merged work: finish → review → merge → *then* begin the next unit. This keeps the merge queue clean, avoids cross-PR conflicts on shared files (this doc, `templates/config.yaml`, `hooks/hooks.json`), and keeps each change reviewable on its own.

**Brainstorm + plan docs ride inside the feature PR — never a standalone docs PR.** Branch creation runs *first* (right after the issue), so every `/ce-brainstorm`, `/ce-plan`, and `/ce-doc-review` artifact (conventionally `docs/brainstorms/<…>.md`, `docs/plans/<…>.md`) is authored *on the feature branch* — it ships in this unit's PR automatically and closes out under the same `Closes #N`, conventionally as the first commit (`docs(scope): brainstorm + plan for #N`). They are the *same unit of work* as the code: the written record of why this PR exists. **Don't author or commit them on the default branch** — a doc stranded there can only reach the remote through its *own* separate PR, the second-PR churn this ordering exists to kill. (If a brainstorm ever happens before you branch — the one genuinely-exploratory step that might re-scope or produce nothing — leave those notes uncommitted and move them onto the feature branch before committing.)

**Reflective skills (off-pipeline, fire occasionally — not per change):**

- `/ce-strategy` — create or update a `STRATEGY.md` at the repo root naming the target problem, approach, users, key metrics, and tracks of work. Run when starting a new product, shifting direction, or when nothing is happening for ambiguous reasons.
- `/ce-product-pulse` — generate a time-windowed pulse report on what users experienced and how the product performed (usage, quality, errors, signals worth investigating). Run weekly/monthly, or when "we should look at how things are going" lands. For this repo the signal sources are GitHub issues/PRs from installers and heartbeat-log reports from repos running the plugin.
- `/ce-ideate` — generate and critically evaluate grounded ideas about a topic. Run when exploring "what should we improve / try next" before committing to a brainstorm — produces several options with trade-offs rather than refining a single idea.
- `/ce-pov` — decisive, project-grounded verdict on an external input: whether to adopt, switch to, or revisit a technology, library, pattern, or platform; whether an outside change (a CVE, a deprecation, an ecosystem shift) actually affects this repo; or a mid-session second opinion. Always returns a graded verdict judged against *this* project, never a neutral explainer — and `/ce-brainstorm` hands verdict-shaped questions here rather than scoping them itself.
- `/ce-optimize` — metric-driven iterative optimization loops. Define a measurable goal, build measurement scaffolding, then run parallel experiments that try many approaches, measure each, and converge. Run when an existing surface (e.g., heartbeat pick-up accuracy, persona-routing quality) has a measurable target you need to drive.

> **Plugin version:** calibrated against `compound-engineering` v3.18.0 (2026-07). If sub-skill behavior diverges from this doc, the plugin wins — re-read the relevant skill at `~/.claude/plugins/cache/compound-engineering-plugin/compound-engineering/<ver>/skills/<skill>/SKILL.md` and update this doc.

> **Branching:** this repo uses git worktrees via `/ce-worktree`, which keeps the main checkout untouched while feature work happens in a sibling directory. If you don't have the plugin, the manual fallback (`git checkout -b`) is below.

### Without the compound-engineering plugin

The `/ce-*` slash commands above belong to the [compound-engineering plugin](https://github.com/EveryInc/compound-engineering-plugin) — recommended for everyone working on this repo, but **not required**. The plugin codifies the pipeline; the *process itself* is achievable with plain git, a code editor, and the GitHub CLI (`gh`) (the GitHub MCP is an optional alternative).

**To install the plugin** (once per machine):

1. Open Claude Code in any repo and run `/plugin` → search for `compound-engineering` → install.
2. Run `/ce-setup` from the repo root to auto-check tools (`gh`, `jq`, etc.) and bootstrap `.compound-engineering/config.local.yaml`.
3. After install, every `/ce-*` reference in this doc is a single command.

**If you don't have the plugin**, here's the manual fallback for each step in the pipeline:

| Plugin command | Manual equivalent |
|---|---|
| `/ce-brainstorm` | Discuss in a GitHub issue comment / a design doc before writing anything. The goal is alignment on user-visible behavior + scope before the first commit. |
| `/ce-plan` | Write an implementation plan to `docs/plans/<YYYY-MM-DD>-<issue-number>-<slug>.md`. The existing specs (`docs/specs/2026-03-25-woterclip-implementation-plan.md`) are a good shape reference. |
| `/ce-doc-review` | CE-native skill. Without it: have a teammate read your plan doc, or write a quick adversarial-review prompt and run it against the doc via the Task tool. Goal: surface contradictions, missing acceptance criteria, scope creep, role-specific gaps. |
| `/ce-worktree` | `git worktree add ../woterclip-<slug> -b <type>/<issue-number>-<slug>` (or plain `git checkout -b` if you don't want the isolation). |
| `/ce-work` | Just write the markdown/YAML. Follow tier rules: validation-first for safety-critical, validation-alongside for standard, no required checks for routine prose fixes. |
| `/ce-code-review` | CE-native skill (parallel reviewer personas + merge/dedup pipeline). Without it: open a draft PR and request review from a teammate, or — if the `pr-review-toolkit` plugin is installed — run its `code-reviewer` agent via the Task tool. |
| `/ce-simplify-code` | CE-native skill. Without it: read the diff yourself with simplification in mind (for this repo: skill wordiness, imperative-form drift, reference files that should absorb SKILL.md detail). |
| `/ce-debug` | CE-native skill. Without it: follow the 4-phase manual process — reproduce → isolate → root-cause → fix-with-validation. For this repo "reproduce" usually means loading the plugin locally and re-running the failing skill/command. |
| `/ce-commit-push-pr` | `git add -A && git commit -m "..." && git push -u origin <branch> && gh pr create --title ... --body ...`. The "Conventional commits" section below has the message format. PR body writing is built in (see `## Branching, commits, and PRs` → PR body for the structure it follows). |
| `/ce-resolve-pr-feedback` | Reply to human PR comments AND resolve every Copilot/automated-reviewer thread (read each, fix or reply, then resolve via the GraphQL `resolveReviewThread` mutation); push fixes as additional commits. The commit→push→PR cycle is not done until all Copilot threads are resolved — never merge with any thread still open. |
| `/ce-compound` | After non-trivial work, write a learnings doc to `docs/solutions/<category>/<slug>.md` with frontmatter (`module`, `tags`, `problem_type`). If the repo keeps an optional `CONCEPTS.md` glossary (see Compound knowledge store below), also add or refine any domain terms the work surfaced. |
| `/ce-compound-refresh` | Edit the relevant `docs/solutions/` entry in place when the underlying skill/template/convention changes. |
| `/ce-strategy` | Maintain a `STRATEGY.md` at the repo root by hand — target problem, approach, users, key metrics, current tracks of work. Update when direction shifts. |
| `/ce-product-pulse` | Review installer-filed GitHub issues, PR feedback, and heartbeat-log reports by hand and write the digest to `docs/pulse/<YYYY-MM-DD>.md`. |
| `/ce-ideate` | Brainstorm in a doc — list candidate ideas, then critically evaluate each against constraints. Divergent generation followed by convergent evaluation; goal is options-with-tradeoffs, not a single refined direction. |
| `/ce-pov` | Write a short decision memo: name the external thing being judged, lay out how this repo's stack and constraints bear on it, end with an explicit adopt / hold / reject verdict plus its strongest counter-argument. The CE skill grounds the verdict in the repo automatically. |
| `/ce-optimize` | Define the target metric, build measurement scaffolding (eval set, scorecard), then iterate experiments by hand and track results in a doc. The CE skill systematizes this — manually it's an experiment log + a clear win/loss criterion. |

The GitHub Issues bookends (issue lifecycle, status transitions) and the Conventional Commits / branch-naming / PR-body rules apply equally with or without the plugin — those are repo conventions, not plugin features.

### Tier rules (blast-radius)

| Tier | Examples in this repo | Process |
|---|---|---|
| **Routine** | Prose/typo fixes in SOUL.md or README, en-dash style fixes, command description tweaks, adding an example to a reference doc | `/ce-work` directly with a bare prompt; `/ce-code-review` optional |
| **Standard** | New persona template following the 3-file pattern, new command or skill mirroring an existing one, new `references/*.md` wired into a skill, docs/spec updates | `/ce-plan` → `/ce-work` → `/ce-code-review` → `/ce-commit-push-pr` |
| **Safety-critical** | `skills/heartbeat/SKILL.md` (the core loop), `templates/config.yaml` schema, the init skill's migration logic, `hooks/hooks.json`, anything touching the label state machine (`agent-working`/`agent-blocked`) or lockfile create/delete paths | `/ce-brainstorm` → `/ce-plan` (every unit gets `Execution note: validate-first`) → `/ce-work` → `/ce-code-review` → `/ce-commit-push-pr` → `/ce-compound` |

Why these surfaces are safety-critical: this repo's bugs don't corrupt data *here* — they corrupt **other repos' orchestration state**. A bad heartbeat edit can misapply GitHub labels (breaking inbox filtering and the state machine for every scaffolded repo), strand a lockfile (the agent silently stops picking up work forever), or violate the `agent-working`/`agent-blocked` mutual exclusion the whole state machine assumes. A `templates/config.yaml` schema change without a `version` bump + matching init-skill migration breaks every already-scaffolded repo on its next init. And this is a public plugin — a merged bug ships to every installer.

`/ce-code-review` is mandatory before merge for this tier.

### When to brainstorm

`/ce-brainstorm` fires when ANY of these are true:

- New persona *type* or a change to the persona hierarchy (Board → CEO → Orchestrator → Workers)
- Any change to the heartbeat flow (pick-up, locking, reporting, exit paths) or the label state machine
- Any `templates/config.yaml` schema change (requires a `version` bump + init-skill migration logic — see CLAUDE.md editing guidelines)
- Cross-cutting work touching 4+ files non-trivially or spanning multiple components (e.g., a change that touches the heartbeat skill + persona templates + init migration together)
- The user story can't be stated in one sentence

Skip brainstorm for: prose fixes, README edits, single-file reference updates, scaffolding a new persona that mirrors an existing template.

### When to validate-first

`/ce-plan` writes `Execution note: validate-first` on every unit in the safety-critical tier. `/ce-work` honors the note: state the observable check first (what a correct heartbeat run / scaffold looks like), then make the edit, then run the check — in separate steps.

For Standard and Routine tiers, run the validation checklist below before review when the change touches YAML, frontmatter, or file references; prose-only edits don't need it.

### Validation (this repo's test suite)

There is no build system, no test framework, and no dependencies — "development" is editing markdown and YAML. The validation checklist (from CLAUDE.md) is the closest thing to a test suite; run whatever applies to the diff:

| Check | How |
|---|---|
| YAML parses | `python3 -c "import yaml; yaml.safe_load(open('file.yaml'))"` for every touched `.yaml` |
| Skill frontmatter | every `skills/*/SKILL.md` has `name` + `description` fields |
| Command/agent frontmatter | every `commands/*.md` and `agents/*.md` has a `description` field |
| References resolve | every `${CLAUDE_PLUGIN_ROOT}/...` path named in a skill points at a file that exists |
| Local plugin load (smoke) | `claude --plugin-dir /Users/alexkim/Documents/Github-Mac-2026/woterclip` — plugin loads, touched commands/skills appear |
| End-to-end (safety-critical only) | run `/woterclip-init` or `/heartbeat` against a scratch repo and confirm the touched flow behaves |

### Compound learnings

Run `/ce-compound` after any non-trivial fix, decision, or pattern discovery. It writes to `docs/solutions/` (see `## Compound knowledge store` below).

`/ce-compound-refresh` runs when existing entries go stale or get superseded. Trigger it when:

- A new learning **contradicts or supersedes** an older entry in the same area (e.g., the Orchestrator/CEO persona split made any pre-split "CEO does triage" note misleading).
- Cited file paths or names **moved or were renamed** (e.g., the runtime scaffold moved from `.claude/woterclip/` to `.woterclip/`).
- Tooling or platform references **changed** (Claude Code plugin-loader behavior shifts, `gh` CLI behavior changes, marketplace.json schema changes).
- An entry references **deprecated patterns** that have been replaced repo-wide.
- Periodic **monthly sweep** when reviewing `docs/solutions/` — quick scan for entries older than 60 days, prune or refresh anything that no longer reflects the repo.

Do not run broad sweeps mid-feature or for every routine commit — refreshes should be intentional, scoped to entries you've already identified as stale.

## Compound knowledge store

`docs/solutions/` — documented solutions to past problems (bugs, architecture patterns, design patterns, tooling decisions, conventions, workflow practices, institutional knowledge). Organized into category subdirectories (e.g., `conventions/`, `architecture-patterns/`) with YAML frontmatter (`module`, `tags`, `problem_type`). (Directory doesn't exist yet — the first `/ce-compound` run creates it.)

**Optional — `CONCEPTS.md` (repo-root domain glossary).** This repo has strong domain vocabulary worth pinning (persona, SOUL/TOOLS/config triple, heartbeat, Board, Orchestrator, label state machine, lockfile). If adopted, a `CONCEPTS.md` at the root defines the terms that mean something precise here — one-sentence definitions that `docs/solutions/`, `AGENTS.md`, and `CLAUDE.md` can cite without redefining. `/ce-compound` accretes to it automatically as a side effect of documenting a learning **if the file already exists**; a repo-wide first draft is a `/ce-compound-refresh` bootstrap. Entries must stand alone (no file paths or drift-prone values — state the behavior, not the value).

Adjacent existing knowledge stores in this repo (search these too before re-solving):

- `docs/specs/2026-03-25-woterclip-design.md` — the design spec (persona system, heartbeat loop, label conventions)
- `docs/specs/2026-03-25-woterclip-implementation-plan.md` — the phased implementation plan the scaffold was built from

Relevant when implementing or debugging in documented areas — search before re-solving.

## Issue tracking

All work on this repo is tracked as **GitHub Issues** on `wotai-dev/woterclip`. Issues are referenced by their repo-scoped number — `#N` (e.g. `#42`) — with no team prefix. Cross-repo references use `wotai-dev/<repo>#N`.

(The plugin's *runtime* tracker is also GitHub Issues — WoterClip orchestrates issues in whatever repo it's scaffolded into, via the `gh` CLI. This section is about tracking development of the plugin itself.)

Single-project repo → **no GitHub Projects board.** Plain GitHub Issues (open/closed + labels) are enough: the issue's open→closed state *is* the lifecycle, and `Closes #N` in a merged PR closes it.

Drive issues with the **`gh` CLI**: `gh issue create|edit|close|view`. (The GitHub MCP is an optional alternative.)

### One issue per unit of work

Every PR maps to ≥1 issue. Exceptions allowed only for trivial chores (typo sweeps, .gitignore tweaks) where the commit message itself documents the change.

For multi-PR work (e.g., a config-schema change that needs the `templates/config.yaml` bump + init-skill migration in one PR and a persona-template update in another), create a parent issue and one **GitHub sub-issue** per PR (Issues UI → "Create sub-issue", or `gh api` against the `/repos/{owner}/{repo}/issues/{n}/sub_issues` REST endpoint — the sub-issue must already exist). The parent shows a sub-issue progress bar and stays open until you close it: closing all sub-issues does **not** auto-close the parent, so close it by hand once the last child merges.

### Issue lifecycle

No board — the lifecycle is the issue's own state plus optional open-state labels:

- **Open** — optionally positioned with `backlog` / `todo` / `in-progress` / `in-review` labels
- **Closed as completed** — merged / shipped (auto via `Closes #N` on merge)
- **Closed as not planned** — won't do
- **Closed as duplicate** — superseded by another issue ("Close as duplicate" links the canonical one)

Orthogonal flag: apply a **`blocked`** label (plus a comment naming the blocker) to any open issue waiting on an external dependency.

`Closes #N` (or `Fixes #N` / `Resolves #N`) in the **PR body** is required — it links the PR to the issue **and auto-closes the issue when the PR merges**. Three preconditions for the auto-close to fire: (1) the PR targets the repo's **default branch** (keywords on PRs into any other branch are silently ignored — no link, no close); (2) repo Settings → Issues hasn't disabled auto-close; (3) the keyword is in the PR **body** or a **commit message** — never the PR **title**.

After merge, whoever runs `/ce-commit-push-pr` should **verify** the close landed (`gh issue view <N>` shows it closed) rather than assume it — a mistyped keyword or a non-default base branch breaks the chain silently.

### Labels (recommended starter set)

Create as needed. Labels drive filtering and dashboards; they're not enforced.

| Group | Labels | Drives |
|---|---|---|
| **Tier** | `tier:routine`, `tier:standard`, `tier:safety-critical` | Dev process (see `## Development process` → tier rules) |
| **Area** | `area:heartbeat`, `area:personas`, `area:init`, `area:skills`, `area:commands`, `area:hooks`, `area:templates`, `area:docs` | Component filtering (mirrors the plugin component map in CLAUDE.md) |
| **Status** (plain-Issue repo) | `backlog`, `todo`, `in-progress`, `in-review` | Visible open-state position. Done / Cancelled / Duplicate aren't labels; they're the issue's close-reason (completed / not planned / duplicate). |
| **Flags / triage** | `blocked`, `needs-info`, `priority:high`, `priority:low`, plus GitHub's built-in `good first issue` + `help wanted` | Orthogonal signals that coexist with any status. Start here and **add more as patterns emerge** — labels are cheap, unenforced, and easy to retire. |
| **Type** | `type:bug`, `type:feature`, `type:improvement` | Issue type. GitHub Issues has no native type field — use labels. GitHub also seeds default labels (`bug`, `enhancement`, `duplicate`, `wontfix`, …) you can keep or replace. |

### Issue title and body conventions

- **Title:** imperative, scope-prefixed when natural — e.g. `heartbeat: delete lockfile on tool-validation exit path`, `personas: add Frontend worker template`, `init: migrate config schema v1 → v2`
- **Body must include:**
  - Acceptance criteria (what "done" looks like)
  - Link to the relevant plan doc (e.g., `docs/plans/...md` or a `docs/solutions/...md` reference if one already covers the area)
  - Tier label rationale (one sentence: "safety-critical because touches `skills/heartbeat/SKILL.md` lockfile handling")

## Branching, commits, and PRs

### Branch naming

`<type>/<issue-number>-<short-slug>` (e.g. `feat/42-frontend-persona`)

- `<type>` ∈ `feat` | `fix` | `chore` | `docs` | `refactor` | `test`
- `<issue-number>` is the GitHub issue number — the **bare** number (e.g. `42`), no `#` and no team prefix. (`#` is awkward in branch names; reserve `#N` for references in commit/PR bodies, where GitHub auto-links it.)
- `<short-slug>` is 2-5 hyphenated words derived from the issue title

Examples:

- `feat/12-frontend-persona`
- `fix/23-lockfile-cleanup`
- `feat/42-persona-import-cmd`
- `docs/31-heartbeat-spec-update`

`/ce-worktree` creates branches following this convention when given an issue number.

### Commit messages (conventional commits)

Format: `<type>(scope): subject`

- `<type>` matches the branch type (`feat`, `fix`, `chore`, `docs`, `refactor`, `test`)
- `scope` is the affected component — common scopes in this repo: `heartbeat`, `personas`, `init`, `skills`, `commands`, `hooks`, `templates`, `references`, `docs`, `plugin` (manifest/marketplace)
- Subject is imperative, lowercase, no period, ≤72 chars

Body references the issue: `Refs #N` for regular references (multiple commits per branch is normal). Use `Closes #N` only on the commit (or PR body) that genuinely closes the issue — on GitHub the keyword *actually closes* the issue once the commit/PR lands on the default branch, so reserve it for the one that finishes the work.

`/ce-commit` and `/ce-commit-push-pr` follow this convention and append the standard `Co-Authored-By: Claude ...` attribution.

Examples:

```
feat(personas): add frontend worker persona template

Adds the SOUL/TOOLS/config triple under templates/personas/frontend/,
following the backend template's structure and placeholder conventions.

Refs #12
```

```
fix(heartbeat): delete lockfile on tool-validation exit path

The tool-validation failure branch returned without removing
.woterclip/.heartbeat-lock, permanently blocking subsequent heartbeats.

Closes #23
```

### PR title

`<type>(scope): <descriptive subject>` — same shape as a commit subject. Keep the issue number **out** of the title; it lives in the PR body as `Closes #N`. (Closing keywords aren't parsed in titles anyway, and GitHub appends the PR's own `(#M)` number to the squash-merge commit, so a number in the title would just double up.)

Examples:

- `feat(personas): frontend worker persona template`
- `fix(heartbeat): lockfile cleanup on every exit path`
- `feat(commands): persona-import from marketplace repos`

### PR body (must include)

- **`Closes #N`** as a top-level line — links the PR to the issue **and auto-closes it when the PR merges to the default branch**. Use `Refs #N` for partial work or supporting PRs that should link without closing.
- **Plan reference** — link to the relevant `docs/plans/...md` or `docs/solutions/...md` if applicable. The brainstorm + plan docs are committed **in this same PR** (see `## Development process` → the "Brainstorm + plan docs ride inside the feature PR" callout), not in a separate docs-only PR — so this normally links to a file the PR *adds*, not a pre-existing one on the default branch
- **Validation** — bulleted checklist of what was verified before merge (which validation-checklist rows ran, whether the plugin was loaded locally, whether an end-to-end scratch-repo run happened for safety-critical changes)
- **Config schema changes** — if the PR touches `templates/config.yaml`'s schema, confirm the `version` field was bumped and the init skill's migration logic updated (per CLAUDE.md editing guidelines)
- **Out-of-scope callouts** — anything explicitly deferred to a follow-up issue, with the `#M` reference

`/ce-commit-push-pr` drafts the body following this structure — PR-description writing is built into that skill (its `references/pr-description-writing.md` is loaded internally during the push step), so no separate command is needed.

### Push

- First push: `git push -u origin <branch>` (sets upstream)
- Subsequent pushes: `git push`
- Never force-push to `main`. Force-push to feature branches is acceptable when rebasing pre-review (after the first reviewer has commented, prefer additive commits to preserve review thread anchors).

### Merge

- **Merge commit by default** — preserves the per-branch commit history on `main` so the development journey (review iterations, intermediate decisions, rationale-in-progress) stays inspectable via `git log --first-parent main` for the merge spine and full `git log` for the in-branch detail
- **Exception:** trivial PRs (single-commit chores, typo sweeps) where the in-branch history adds no signal — use squash merge to keep main from accumulating one-commit noise per chore
- Because every branch commit lands on `main`, the **Conventional Commits rules above are mandatory, not aspirational** — bad commit messages on a feature branch become permanent main-history noise. Reword sloppy WIP commits via interactive rebase before opening the PR (or before the final push if the PR is already open and pre-review)
- The PR body (and thus the merge commit) should carry the `Closes #N` reference. On merge to the default branch GitHub closes the issue automatically — **verify** it landed (`gh issue view <N>`) rather than assuming. Not-planned / duplicate closes are manual (see `## Issue tracking` § Issue lifecycle)
- **Resolve automated reviewer threads as part of `/ce-resolve-pr-feedback`, before merging (not a separate optional pass that gets skipped)** — GitHub Copilot fires on every PR and frequently produces substantive technical findings (e.g., YAML keys that don't match what the skills actually read, dead references, frontmatter drift). Treat Copilot review threads as a required reviewer pass at the same weight as human review, and handle them in the same `/ce-resolve-pr-feedback` step: after opening the PR, wait for Copilot's review to land, then for each comment read it, fix or reply, and mark the thread resolved via the GraphQL `resolveReviewThread` mutation (`gh api graphql -f query='mutation { resolveReviewThread(input:{threadId:"..."}) { thread { isResolved } } }'`). To find open threads, list them with `gh pr view <n> --json reviewThreads` or the GraphQL `reviewThreads` query and confirm each `isResolved: true` before merge. Do not merge with unresolved Copilot threads — fixing the diff in a follow-up PR after merging is the slower, noisier path
- Safety-critical PRs (heartbeat skill, config schema + init migration, hooks, label state machine, lockfile paths) require a green `/ce-code-review` before merge

### After merge

- Delete the feature branch (GitHub setting: delete head branches automatically; locally `git worktree remove <path>` then `git branch -d <branch>`)
- Periodically prune locals whose remotes are gone: `git fetch -p && git branch -vv | awk '/: gone]/ {print $1}' | xargs -r git branch -d`
- If the work was non-trivial, run `/ce-compound` to capture the learning before context fades — writes to `docs/solutions/`

## Releases & release notes

Cut **versioned GitHub Releases with auto-generated notes**. The Releases page is the source of truth — **no `CHANGELOG.md`** to hand-maintain — and the compound-engineering plugin has **no release-notes skill** (it shipped a release-notes command once — removed in the v3.14.1 consolidation), so releasing is native GitHub, not a `/ce-*` step.

**A `vX.Y.Z` git tag *is* the release.** A release bump touches **two** manifests, which must stay on one synchronized version: `.claude-plugin/plugin.json` (`version`) and `.claude-plugin/marketplace.json` (the plugin entry's `version`). Currently `0.1.0`.

Add **`.github/release.yml`** so GitHub's "Generate release notes" groups merged PRs by label — the categories map to the repo's Type labels:

```yaml
changelog:
  exclude:
    labels: [ignore-for-release]
  categories:
    - title: 🚀 Features
      labels: [type:feature]
    - title: 🐛 Fixes
      labels: [type:bug]
    - title: 🔧 Improvements
      labels: [type:improvement]
    - title: 📚 Docs
      labels: [area:docs]
    - title: Other Changes
      labels: ["*"]
```

**Cutting a release:**

1. On a branch, bump both manifest versions and open a `chore(release): vX.Y.Z` PR; merge it through the normal pipeline.
2. Tag the merge commit on the default branch and push the tag (tags bypass branch protection — push them directly, never via a PR):
   ```bash
   git checkout main && git pull
   git tag -a vX.Y.Z -m "vX.Y.Z" && git push origin vX.Y.Z
   ```
3. On the Release, click **"Generate release notes"** — GitHub categorizes every PR merged since the previous tag per `.github/release.yml`.

**What makes the notes good is upstream of the release:** Conventional-Commit PR titles become the line items and `Closes #N` links each to its issue — but the **categories come from PR labels**, so **label every PR with a `type:`** (and `area:docs` for docs) before merge, or it lands in "Other Changes". Add an `ignore-for-release` label to omit a PR. Tags must be `vX.Y.Z`.
