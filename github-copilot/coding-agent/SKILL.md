---
name: coding-agent
description: GitHub Copilot coding agent guide covering autonomous PR creation from issues, making changes to existing PRs, tracking agent sessions, reviewing Copilot PRs, custom agents, hooks, agent skills, MCP server integration, and built-in security protections. Use when assigning tasks to Copilot to create or modify pull requests, configuring the coding agent, or customizing agent behavior with custom instructions, MCP, or hooks.
---
# GitHub Copilot Coding Agent

Source: https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent

## Overview

Copilot coding agent is an autonomous AI agent that works independently in the background to complete development tasks, just like a human developer. It has access to a sandboxed GitHub Actions-powered development environment where it can:

- Explore your code
- Make changes and commit them
- Execute automated tests and linters
- Open a pull request for your review

## What Copilot Coding Agent Can Do

- Fix bugs
- Implement incremental new features
- Improve test coverage
- Update documentation
- Address technical debt
- Fix security alerts (assignable from security campaigns)

## Availability

Copilot coding agent is available with:
- **Copilot Pro** and **Copilot Pro+**
- **Copilot Business** (requires admin to enable the policy)
- **Copilot Enterprise** (requires admin to enable the policy)

## How to Assign Tasks

### Create a new pull request

1. **From a GitHub Issue**: Open an issue and assign it to **Copilot** (select Copilot as the assignee).
2. **From Copilot Chat**: In GitHub.com or VS Code, type a task description and ask Copilot to open a PR.
3. **From the Agents panel**: Available on every GitHub page (top-right area); start a new session.
4. **From the GitHub CLI**: Use `gh copilot` commands to trigger Copilot.
5. **From external tools via MCP**: Agentic coding tools and IDEs with MCP support.

### Make changes to an existing pull request

Mention `@copilot` in a PR comment to ask Copilot to make changes to an existing PR created by a human.

## Tracking Agent Sessions

Track Copilot's progress via:
- **Agents panel** (available on every GitHub page)
- **Visual Studio Code** and **JetBrains IDEs** agents panel
- **Session logs**: View every step Copilot took, including commands run and changes made
- **GitHub CLI** (`gh copilot sessions`)
- **Raycast** integration

## Reviewing Copilot Pull Requests

After Copilot finishes, it requests a review from you. To review:

1. Open the pull request.
2. Review the changes, session logs, and any test results.
3. Leave comments mentioning `@copilot` to ask for further changes.
4. Approve and merge when satisfied.

> **Note**: The developer who asked Copilot to create a PR cannot approve that same PR (ensures independent review in repositories requiring approvals).

## Customizing Copilot Coding Agent

### Custom Instructions

Store natural language instructions in your repository to guide the agent's behavior:

- **Repository-wide**: `.github/copilot-instructions.md`
- **Path-specific**: `.github/instructions/*.instructions.md`
- **Agent instructions**: `AGENTS.md` in the repository root

Example `.github/copilot-instructions.md`:
```markdown
## Project Overview
This is a React/TypeScript app using Tailwind CSS for styling and Jest for tests.

## Coding Standards
- Use TypeScript strict mode
- Prefer functional components with hooks
- Write unit tests for all new functions
- Run `npm run lint` before committing
```

### MCP Servers

Connect Model Context Protocol (MCP) servers to give Copilot access to external tools and data:
- Databases, APIs, or internal services
- Custom tooling for your workflow

Configure MCP servers in the repository's `.github/copilot-mcp.json` or via GitHub organization/enterprise settings.

### Custom Agents

Create specialized versions of Copilot for different tasks:
- **Frontend agent**: Expert in React components and styling guidelines.
- **Documentation agent**: Focused on writing and updating technical docs.
- **Testing agent**: Specialized in generating comprehensive unit tests.

Custom agents are defined with tailored system prompts and can be selected when starting a coding task.

### Hooks

Hooks let you execute custom shell commands at key points during agent execution:

| Hook | When it runs |
|------|-------------|
| `pre-task` | Before Copilot starts the task |
| `post-edit` | After Copilot makes file edits |
| `pre-push` | Before Copilot pushes commits |
| `post-task` | After Copilot completes the task |

Use hooks for validation, security scanning, build verification, or custom logging.

### Agent Skills

Skills provide Copilot with specialized instructions, scripts, and resources to perform specific tasks. Define skills as `SKILL.md` files following the [VS Code Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) specification.

## Built-in Security Protections

Copilot coding agent includes multiple security layers:

- **CodeQL scanning**: Analyzes generated code for security issues.
- **Dependency advisory check**: New dependencies are checked against the GitHub Advisory Database (CVSS High/Critical and malware advisories).
- **Secret scanning**: Detects hardcoded secrets (API keys, tokens) in generated code.
- **Sandboxed environment**: Copilot works in a restricted environment with firewall-controlled internet access and read-only repository access.
- **Branch restrictions**: Can only push to branches starting with `copilot/`.
- **Write-access gating**: Only responds to users with write access to the repository.
- **Outside collaborator treatment**: Draft PRs require approval before Actions workflows run; Copilot cannot approve or merge its own PRs.
- **Co-authorship tracking**: Commits are co-authored by the requesting developer for compliance.

## Costs

Copilot coding agent uses:
- **GitHub Actions minutes**: For running the development environment.
- **Copilot premium requests**: Each task session consumes premium requests.

Within your plan's monthly allowance, basic tasks typically incur no additional cost.

## References

- [About GitHub Copilot coding agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)
- [Asking GitHub Copilot to create a pull request](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-a-pr)
- [Asking GitHub Copilot to make changes to an existing pull request](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/make-changes-to-an-existing-pr)
- [Tracking GitHub Copilot's sessions](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/track-copilot-sessions)
- [Reviewing a pull request created by GitHub Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/review-copilot-prs)
- [About custom agents](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-custom-agents)
- [About hooks](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-hooks)
- [MCP and Copilot coding agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/mcp-and-coding-agent)
