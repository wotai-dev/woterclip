# Tools — Backend Engineer Persona

## Required

- **`gh` CLI** (via Bash): Read issues, post comments, update labels/state. Verify with `gh auth status`. Target the repo from config `github.repo` — pass `--repo <owner/name>` explicitly on every issue command.
- **Repo tools** (Read, Write, Edit, Bash, Grep, Glob): Full codebase access for implementation.

## Common Patterns

### Implement a feature
1. Read the issue and existing code (`gh issue view N --repo <owner/name> --json title,body,comments`)
2. Create or modify files (Edit/Write)
3. Run tests (Bash)
4. Commit changes (Bash — git)
5. Summarize the work for your returned outcome — commit SHAs, PR URLs, what's next; the heartbeat loop posts the report comment

### Fix a bug
1. Read the issue for reproduction steps
2. Find relevant code (Grep/Glob)
3. Fix and add regression test
4. Commit, then summarize the fix for your returned outcome — commit SHA, what's next

### Database work
- Use Neon MCP (`mcp__neon__*`) if available for database operations
- Always create reversible migrations
- Test migration up and down paths

## Optional Tools

Add to `required_tools` in config.yaml as needed:
- `mcp__neon` — Neon Postgres (database queries, migrations)
- `mcp__context7` — Documentation lookup
