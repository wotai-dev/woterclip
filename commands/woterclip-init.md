---
description: Initialize WoterClip in this repo (config, personas, GitHub labels)
---

Initialize WoterClip in this repository using the woterclip-init skill.

Follow the full initialization procedure from the skill:
1. Verify the `gh` CLI is installed and authenticated (`gh auth status`)
2. Gather GitHub context (repo, user login)
3. Ask which persona preset to scaffold
4. Create GitHub labels (state + persona labels)
5. Scaffold `.woterclip/` with config and persona templates from `${CLAUDE_PLUGIN_ROOT}/templates/`
6. Offer schedule setup
7. Print summary of what was created

If `.woterclip/` already exists, ask whether to overwrite, merge, or cancel before proceeding. If the existing config is `version: 1` (Linear-era), run the v1→v2 migration from the skill's Re-initialization section.
