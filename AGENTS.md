# AGENTS.md

This file defines behavioral rules for AI agents working in this environment. All reusable knowledge and learned patterns are stored in the [CQ knowledge base](https://cq-mcp.bartschnet.de).

## Repository Contents

This repository (`~/.copilot/skills/`) contains:

- **`AGENTS.md`** — agent behavioral rules (this file)
- **`copilot-instructions.md`** — session initialization and core directives
- **`hardware/`** — hardware reference files (manuals, BIOS ROMs, diagrams)

All skills, procedures, and learned patterns belong in **CQ** — not in this repository.

## Security Rules

**This is a public GitHub repository. Never commit passwords, tokens, API keys, or other secrets into any file.** If an example requires credentials, use placeholder values like `<your-token>`, `<api-key>`, or `$TOKEN` (shell variable).

This rule applies to:
- API tokens (e.g. NetBird `nbp_...`, GitHub PATs, cloud provider keys)
- Passwords and passphrases
- Private keys or certificates
- Any value that would grant access if disclosed

If a skill currently contains a real credential, remove it immediately, rotate the credential, and replace with a placeholder.


