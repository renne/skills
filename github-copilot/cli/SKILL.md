---
name: cli
description: GitHub Copilot CLI terminal assistant guide covering installation, authentication, interaction modes (interactive, plan, autopilot), keyboard shortcuts, slash commands, LSP integration, MCP servers, agent skills, custom instructions, and session management. Use when setting up the Copilot CLI, understanding available commands and shortcuts, configuring MCP servers or LSP, managing skills, or customizing agent behavior in the terminal.
---
# GitHub Copilot CLI

Source: https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli

## Overview

GitHub Copilot CLI brings AI-powered coding assistance directly to the command line. It uses the same agentic harness as the GitHub Copilot coding agent and is deeply integrated with GitHub workflows.

Key capabilities:
- Explore, edit, build, and debug code via natural language
- Access GitHub repos, issues, and pull requests
- Extend via MCP servers (ships with GitHub MCP server by default)
- Load agent **Skills** from `~/.copilot/skills/` to augment knowledge
- Plan mode for structured multi-step implementations
- LSP integration for code intelligence

Default model: **Claude Sonnet 4.5** (changeable with `/model`).

---

## Installation

### Linux / macOS (curl)
```bash
curl -fsSL https://gh.io/copilot-install | bash
# Install as root (to /usr/local/bin):
curl -fsSL https://gh.io/copilot-install | sudo bash
# Install specific version:
curl -fsSL https://gh.io/copilot-install | VERSION="v0.0.369" bash
```

### Homebrew (macOS / Linux)
```bash
brew install copilot-cli               # stable
brew install copilot-cli@prerelease    # prerelease
```

### WinGet (Windows)
```bash
winget install GitHub.Copilot
winget install GitHub.Copilot.Prerelease
```

### npm (all platforms)
```bash
npm install -g @github/copilot          # stable
npm install -g @github/copilot@prerelease
```

### Launch
```bash
copilot               # start interactive session
copilot --banner      # force animated banner
copilot --experimental  # enable experimental features
```

---

## Authentication

### GitHub Account (recommended)
On first launch, use the `/login` slash command and follow the on-screen prompts.

### Personal Access Token (PAT)
1. Create a fine-grained PAT at https://github.com/settings/personal-access-tokens/new
2. Add the **"Copilot Requests"** permission
3. Export the token: `export GH_TOKEN=<token>` (or `GITHUB_TOKEN`)

---

## Interaction Modes

Press **Shift+Tab** to cycle through modes:

| Mode | Description |
|------|-------------|
| **Interactive** (default) | Normal conversational mode |
| **Plan** | Agent creates a structured plan before implementing; saves to session folder |
| **Autopilot** (experimental) | Agent continues working until task is complete without confirmation |

To activate plan mode externally, prefix messages with `[[PLAN]]`.

---

## Keyboard Shortcuts

### Navigation & Control
| Shortcut | Action |
|----------|--------|
| `Shift+Tab` | Cycle modes: interactive → plan → (autopilot) |
| `Ctrl+S` | Run command while preserving input |
| `Ctrl+T` | Toggle model reasoning display |
| `Ctrl+O` | Expand recent timeline (when no input) |
| `Ctrl+E` | Expand all timeline (when no input) |
| `↑` / `↓` | Navigate command history |
| `!` | Execute command in local shell (bypass Copilot) |
| `Esc` | Cancel current operation |
| `Ctrl+C` | Cancel / clear input / exit |
| `Ctrl+D` | Shutdown |
| `Ctrl+L` | Clear screen |

### Editing
| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Ctrl+H` | Delete previous character |
| `Ctrl+W` | Delete previous word |
| `Ctrl+U` | Delete from cursor to start |
| `Ctrl+K` | Delete from cursor to end |
| `Meta+←/→` | Move cursor by word |
| `Ctrl+G` | Edit prompt in external editor |

### Mentioning files
Prefix with `@` to include file contents in context: `@ src/main.py`

---

## Slash Commands

### Agent Environment
| Command | Description |
|---------|-------------|
| `/init` | Initialize Copilot instructions for this repository |
| `/agent` | Browse and select available agents |
| `/skills` | Manage skills for enhanced capabilities |
| `/mcp` | Manage MCP server configuration |
| `/plugin` | Manage plugins and plugin marketplaces |

### Models & Subagents
| Command | Description |
|---------|-------------|
| `/model` | Select AI model |
| `/fleet` | Enable fleet mode for parallel subagent execution |
| `/tasks` | View and manage background tasks |

### Code
| Command | Description |
|---------|-------------|
| `/ide` | Connect to IDE workspace |
| `/diff` | Review changes in current directory |
| `/review` | Run code review agent |
| `/lsp` | Manage language server configuration |
| `/terminal-setup` | Configure terminal for multiline input (Shift+Enter) |

### Permissions
| Command | Description |
|---------|-------------|
| `/allow-all` | Enable all permissions |
| `/add-dir` | Add directory to allowed file access list |
| `/list-dirs` | Show allowed directories |
| `/cwd` | Change or show working directory |
| `/reset-allowed-tools` | Reset allowed tools list |

### Session
| Command | Description |
|---------|-------------|
| `/resume` | Switch to different session |
| `/rename` | Rename current session |
| `/context` | Show context window token usage |
| `/usage` | Display session usage metrics |
| `/session` | Show session info and workspace summary |
| `/compact` | Summarize history to reduce context |
| `/share` | Share session to markdown or GitHub Gist |
| `/copy` | Copy last response to clipboard |

### Planning
| Command | Description |
|---------|-------------|
| `/plan` | Create an implementation plan before coding |
| `/research` | Deep research via GitHub search and web sources |

### Help & Config
| Command | Description |
|---------|-------------|
| `/help` | Show help |
| `/model` | Select AI model |
| `/theme` | Configure terminal theme |
| `/update` | Update CLI to latest version |
| `/experimental` | Show/toggle experimental features |
| `/instructions` | View/toggle custom instruction files |
| `/feedback` | Submit feedback survey |
| `/changelog` | Show version changelog |
| `/login` / `/logout` | Authentication |
| `/clear` | Clear conversation history |
| `/exit` / `/quit` | Exit CLI |

---

## Custom Instructions

Copilot loads custom instructions from (in order):
- `CLAUDE.md`
- `GEMINI.md`
- `AGENTS.md` (git root and cwd)
- `.github/instructions/**/*.instructions.md` (git root and cwd)
- `.github/copilot-instructions.md`
- `$HOME/.copilot/copilot-instructions.md`
- Directories from `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` env var

---

## Agent Skills

Skills augment the agent's domain knowledge. They live in `~/.copilot/skills/` following the [VS Code Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) standard.

### Structure
```
~/.copilot/skills/
└── <category>/
    └── <skill-name>/
        └── SKILL.md
```

### SKILL.md format
```markdown
---
name: skill-name
description: What it does and when to use it (1–1024 chars).
---
# Skill Title
Detailed instructions...
## References
- https://...
```

### Managing skills
- Use `/skills` to list and enable/disable skills interactively
- Keep a skills repo up to date: `git -C ~/.copilot/skills pull --recurse-submodules`

### Best practice (from custom instructions)
At the start of each task, the agent should:
1. Pull latest skills: `git -C ~/.copilot/skills pull --recurse-submodules && git -C ~/.copilot/skills submodule update --init --recursive --remote`
2. Search `~/.copilot/skills/` for relevant `SKILL.md` files
3. Read matching skills before proceeding

---

## MCP Servers

Copilot CLI ships with the **GitHub MCP server** by default. Custom MCP servers can be added via the Docker MCP Gateway or local STDIO.

Manage with `/mcp`. Configuration is typically in `~/.docker/mcp/config.yaml`.

---

## LSP Integration

Configure language servers for enhanced code intelligence (go-to-definition, hover, diagnostics).

**User-level** (`~/.copilot/lsp-config.json`) or **repo-level** (`.github/lsp.json`):

```json
{
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "fileExtensions": {
        ".ts": "typescript",
        ".tsx": "typescript"
      }
    }
  }
}
```

Use `/lsp` to check configured servers.

---

## Session State

Each session has a dedicated folder (shown in session info):
- **`plan.md`** – implementation plans created in plan mode
- **`files/`** – persistent session artifacts

The agent also has access to a per-session SQLite database with pre-created `todos` and `todo_deps` tables for task tracking.

---

## Experimental Features

Enable via `--experimental` flag or `/experimental` command.

Current experimental features:
- **Autopilot mode** – agent continues autonomously until task is complete (Shift+Tab to activate)

---

## References

- https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli
- https://docs.github.com/en/copilot/concepts/agents/about-copilot-cli
- https://github.com/github/copilot-cli
