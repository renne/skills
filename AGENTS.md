# AGENTS.md

This repository is a collection of [Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) for GitHub Copilot and other compatible AI agents. All skill files must follow the rules below.

## Repository Scope

**This is a public repository for generalized, reusable skills only.**

### ✅ Belongs here (`skills`)

- Generalized procedures for working with technologies (e.g., how to use the NetBird API, how to configure Traefik, Docker Compose patterns).
- Technology best practices that apply regardless of the specific infrastructure.
- Tool usage guides and reference patterns.
- Skills that would be useful in *any* deployment of a given technology.

### ❌ Does NOT belong here → goes in `networks` (private)

- Specific host names, IP addresses, subnet assignments, or VLAN IDs.
- Network topology maps or routing decisions tied to real infrastructure.
- Migration plans, upgrade roadmaps, or go/no-go checklists for this specific environment.
- Per-host service inventory (e.g., "Nextcloud runs on 10.0.0.29").
- Any detail that reveals the structure of personal or organizational infrastructure.

**Rule of thumb:** If the content would still be correct and useful after replacing all hostnames and IPs with placeholders, it belongs in `skills`. If it only makes sense in the context of the real infrastructure, it belongs in `networks`.

## Directory Structure

Each skill lives in its own dedicated directory containing a `SKILL.md` file:

```
<category>/
└── <skill-name>/
    └── SKILL.md
```

- `<category>` is an organizational grouping (e.g., `moodle/`, `nextcloud/`, `docker-compose/`).
- `<skill-name>` is the skill directory; its name must exactly match the `name` field in `SKILL.md`.
- Additional resource files (scripts, examples, reference docs) may be placed alongside `SKILL.md` inside the skill directory.

## SKILL.md Format

Every `SKILL.md` must begin with YAML frontmatter followed by Markdown content:

```markdown
---
name: skill-name
description: What the skill does and when to use it.
---
# Skill Title

Detailed instructions...
```

### Required Frontmatter Fields

| Field | Rules |
|-------|-------|
| `name` | Lowercase letters, numbers, and hyphens only. Must match the parent directory name exactly. Maximum 64 characters. |
| `description` | 1–1024 characters. Must describe **both** what the skill does and **when to use it**. Use action-oriented language to help the AI decide when to load the skill. |

### Optional Frontmatter Fields

| Field | Description |
|-------|-------------|
| `argument-hint` | Hint shown in the chat input when the skill is invoked as a slash command. |
| `user-invocable` | Set to `false` to hide the skill from the `/` menu (default: `true`). |
| `disable-model-invocation` | Set to `true` to require manual invocation only (default: `false`). |

## Skill Body

The Markdown body must contain clear instructions that describe:

- What the skill helps accomplish.
- When to use the skill.
- Step-by-step procedures or guidelines to follow.
- Code examples where applicable.
- A `## References` section linking to the authoritative source(s).

## Document Quirks, Roadblocks, and API Structures

Whenever you encounter unexpected behavior, a non-obvious limitation, or a working pattern through trial and error, **document it immediately** in the relevant `SKILL.md`. Do not wait until the end of the session.

Specifically, always document:

- **API quirks**: required fields that are not obvious from the docs, fields that silently ignored, error messages that are misleading, or undocumented defaults (e.g., "PUT /api/groups requires `name` even when only changing peers or else returns 422").
- **Roadblocks and failure modes**: things that look like they should work but don't, and the actual root cause (e.g., "Netbird Networks subnet resources do NOT install OS-level routes on client peers — use classic routes instead").
- **Working patterns**: the exact sequence of steps or API calls that solved a problem, so the same effort is not repeated.
- **Non-obvious dependencies**: services or configs that must be in a certain state before a step will succeed.

Place these under a `## Known Quirks and Limitations` or `## Troubleshooting` heading in the relevant `SKILL.md`. Use `⚠️` to mark critical pitfalls that would cause hard-to-diagnose failures.

The goal is to avoid duplicating debugging effort across sessions. If you wasted time on something, capture it so future sessions don't repeat the same journey.

## Adding a New Skill

1. Choose the appropriate category directory or create a new one.
2. Create a subdirectory named after the skill (lowercase, hyphens for spaces, max 64 characters).
3. Create `SKILL.md` inside that directory with the required frontmatter.
4. Write clear, specific instructions in the body.
5. Add a `## References` section with source links.
6. Update `README.md` to list the new skill.

## Security Rules

**This is a public GitHub repository. Never commit passwords, tokens, API keys, or other secrets into any skill file.** If an example requires credentials, use placeholder values like `<your-token>`, `<api-key>`, or `$TOKEN` (shell variable).

This rule applies to:
- API tokens (e.g. NetBird `nbp_...`, GitHub PATs, cloud provider keys)
- Passwords and passphrases
- Private keys or certificates
- Any value that would grant access if disclosed

If a skill currently contains a real credential, remove it immediately, rotate the credential, and replace with a placeholder.

## Checklist for Every Skill File

- [ ] File is named exactly `SKILL.md`.
- [ ] File is inside a dedicated skill directory (e.g., `moodle/php-coding-style/`).
- [ ] `name` field is present and matches the parent directory name.
- [ ] `description` field is present, 1–1024 characters, and explains both capability and trigger conditions.
- [ ] Body contains clear instructions or guidelines.
- [ ] A `## References` section lists the authoritative source(s).
- [ ] `README.md` has been updated to list the skill.

## References

- [Use Agent Skills in VS Code](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [Agent Skills Specification](https://agentskills.io/specification)

## Session Learning

At the end of every session, update the skills library with anything new you learned:

1. If you discovered a new pattern, tool, or procedure not covered by an existing skill, create a new `SKILL.md` following the format above.
2. If you improved upon or corrected an existing skill's instructions, edit that `SKILL.md` directly.
3. Run `git -C ~/.copilot/skills add -A && git -C ~/.copilot/skills commit -m "chore: update skills from session learnings"` to persist the changes. Always pull and rebase before push.
4. If the repository has a remote, push with `git -C ~/.copilot/skills push`.

Apply this rule at the end of any session in which you used or could have used a skill.

**Before context compaction:** If the context is about to be compacted, immediately follow steps 1–3 above to persist all accumulated learnings before they are lost.
