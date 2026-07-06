# Tools — Frontend Engineer Persona

## Required

- **`gh` CLI** (via Bash): Read issues, post comments, update labels/state. Verify with `gh auth status`.
- **Repo tools** (Read, Write, Edit, Bash, Grep, Glob): Full codebase access for implementation.

## Common Patterns

### Build a component
1. Read the issue for requirements and linked designs (`gh issue view N --json title,body,comments`)
2. Check existing components for patterns (Glob/Grep)
3. Create or modify component files (Write/Edit)
4. Run dev server and verify (Bash)
5. Commit and post heartbeat comment (Bash + `gh issue comment`)

### Fix a UI bug
1. Read the issue for reproduction steps
2. Find the relevant component (Grep/Glob)
3. Fix and test interactive states
4. Commit and comment

### Styling work
- Use the project's design system tokens (colors, spacing, typography)
- Check for dark mode compatibility if the project supports it
- Test responsive breakpoints

## Optional Tools

Add to `required_tools` in config.yaml as needed:
- `mcp__plugin_playwright_playwright` — Browser automation for visual verification
- `mcp__context7` — Documentation lookup
