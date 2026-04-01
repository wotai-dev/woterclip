---
description: Run a WoterClip heartbeat cycle (pick up issues, do work, report)
argument-hint: "[--dry-run] [--persona NAME]"
---

Run the WoterClip heartbeat using the heartbeat skill.

Arguments passed: $ARGUMENTS

Parse the arguments:
- If `--dry-run` is present, pass it through to the heartbeat procedure (step 3 reports what would be picked without doing work)
- If `--persona <name>` is present, filter to only issues matching that persona's label

Execute the full 11-step heartbeat procedure. On any error or exit, ensure the lockfile at `.woterclip/.heartbeat-lock` is deleted.
