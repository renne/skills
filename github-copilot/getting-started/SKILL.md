---
name: getting-started
description: GitHub Copilot overview covering what it is, all core features (inline suggestions, Chat, coding agent, CLI, code review, pull request summaries, Copilot Edits, Spaces), available plans (Free, Pro, Pro+, Business, Enterprise), and how to get access. Use when introducing GitHub Copilot, comparing plans, or deciding which Copilot features to use.
---
# GitHub Copilot — Getting Started

Source: https://docs.github.com/en/copilot

## What is GitHub Copilot?

GitHub Copilot is an AI coding assistant that helps you write code faster and with less effort. It provides autocomplete-style suggestions, a chat interface for coding questions, and autonomous agents that can create pull requests on your behalf.

You can use Copilot in:

- Your IDE (VS Code, Visual Studio, JetBrains IDEs, Xcode, Vim/Neovim, Eclipse, Azure Data Studio)
- The GitHub website (github.com)
- GitHub Mobile
- Windows Terminal (Terminal Chat)
- The command line (GitHub Copilot CLI)

## Core Features

### Inline Suggestions

Copilot offers autocomplete-style code suggestions as you type in your IDE. Describe what you want in a comment and Copilot will suggest the implementation. Supports 30+ programming languages including Python, JavaScript, TypeScript, Ruby, Go, C#, C++, and more.

**Next edit suggestions** (public preview in VS Code, Visual Studio, Xcode, Eclipse): Predicts the location and content of your next edit based on current edits.

### Copilot Chat

A conversational AI interface for coding questions, code explanations, test generation, and bug fixes. Available on GitHub.com, in IDEs (VS Code, Visual Studio, JetBrains, Eclipse, Xcode), GitHub Mobile, and Windows Terminal.

### Copilot Coding Agent

An autonomous AI agent that works independently in the background to complete development tasks. You can assign GitHub issues to Copilot, which then creates a pull request for your review. Works inside a GitHub Actions-powered sandboxed environment.

### Copilot CLI

Use Copilot from the command line for Q&A, local file changes, and GitHub.com interactions (e.g., listing PRs, creating issues). Supports autopilot mode and the `/fleet` command for parallel task execution.

### Copilot Code Review

AI-generated pull request review suggestions to improve code quality.

### Copilot Pull Request Summaries

AI-generated descriptions of PR changes, affected files, and reviewer focus areas.

### Copilot Edits (VS Code, Visual Studio, JetBrains)

Make changes across multiple files from a single chat prompt. Two modes:
- **Edit mode** — granular control; you choose which files to change.
- **Agent mode** — autonomous; Copilot determines files, runs terminal commands, and iterates.

### Copilot Spaces

Organize code, docs, and specs into Spaces that ground Copilot's responses in specific context for a task.

### Copilot Memory (public preview)

Copilot can store and reuse useful details it has learned about a repository to improve future coding agent and code review output.

### GitHub Spark (public preview)

Build and deploy full-stack apps using natural-language prompts, integrated with GitHub.

## Plans

| Plan | For | Key features |
|------|-----|--------------|
| **Copilot Free** | Individual developers | Core inline suggestions, limited Chat (50 chat messages/month, 2,000 completions/month), access to Claude Sonnet and GPT-4o |
| **Copilot Pro** | Individual developers | Unlimited suggestions & chat, coding agent (public preview), code review, PR summaries, multiple AI models |
| **Copilot Pro+** | Power users | All Pro features plus higher premium request limits, access to all frontier models (o3, Claude Opus, Gemini Ultra) |
| **Copilot Business** | Organizations | All Pro features, policy & access management, usage data, audit logs, content exclusion, SSO |
| **Copilot Enterprise** | Enterprises | All Business features, organization custom instructions, Copilot Spaces, fine-tuned models |

## Getting Access

- **Individuals**: Sign up at github.com/features/copilot. Free tier available; paid plans via subscription.
- **Students, teachers, open source maintainers**: May qualify for free Copilot Pro access.
- **Organizations**: Owners purchase Copilot Business or Enterprise from organization settings.
- **Enterprises**: Enterprise owners manage licenses and enable Copilot for organizations.

## Next Steps

1. Install the GitHub Copilot extension in your IDE.
2. Open a file and start typing — Copilot will suggest completions.
3. Open Copilot Chat (`Ctrl+Alt+I` / `Cmd+I` in VS Code) to ask coding questions.
4. Assign a GitHub issue to Copilot to create a pull request autonomously.

## References

- [What is GitHub Copilot?](https://docs.github.com/en/copilot/get-started/what-is-github-copilot)
- [GitHub Copilot features](https://docs.github.com/en/copilot/get-started/features)
- [Plans for GitHub Copilot](https://docs.github.com/en/copilot/get-started/plans)
- [Quickstart for GitHub Copilot](https://docs.github.com/en/copilot/get-started/quickstart)
- [Best practices for using GitHub Copilot](https://docs.github.com/en/copilot/get-started/best-practices)
