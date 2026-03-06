# AGENTS.md

This repository is a collection of [Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) for GitHub Copilot and other compatible AI agents. All skill files must follow the rules below.

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

## Adding a New Skill

1. Choose the appropriate category directory or create a new one.
2. Create a subdirectory named after the skill (lowercase, hyphens for spaces, max 64 characters).
3. Create `SKILL.md` inside that directory with the required frontmatter.
4. Write clear, specific instructions in the body.
5. Add a `## References` section with source links.
6. Update `README.md` to list the new skill.

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
