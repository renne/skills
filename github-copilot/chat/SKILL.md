---
name: chat
description: GitHub Copilot Chat guide covering how to ask coding questions, explain code, generate tests, fix bugs, and use slash commands and chat participants in VS Code, JetBrains, Visual Studio, GitHub.com, GitHub Mobile, and Windows Terminal. Also covers extending Chat with MCP servers and changing AI models. Use when asking Copilot a coding question, explaining or debugging code, or setting up Copilot Chat in an IDE or browser.
---
# GitHub Copilot Chat

Source: https://docs.github.com/en/copilot/concepts/chat

## Overview

GitHub Copilot Chat is a conversational AI interface that lets you interact with Copilot to get coding assistance, explanations, and suggestions. It is available in:

- **IDEs**: VS Code, Visual Studio, JetBrains IDEs, Eclipse, Xcode
- **GitHub website** (github.com)
- **GitHub Mobile**
- **Windows Terminal**
- **GitHub Copilot CLI**

## Common Use Cases

- **Code suggestions**: Ask Copilot to implement a function or algorithm.
- **Explain code**: Ask "What does this function do?" or "Explain this regex."
- **Generate unit tests**: Ask Copilot to write tests for a function.
- **Fix bugs**: Paste an error message or broken code and ask for a fix.
- **Refactor**: Ask Copilot to simplify, optimize, or restructure code.
- **Documentation**: Ask Copilot to generate docstrings or comments.
- **PR summaries**: Ask Copilot to summarize pull request changes.

## Slash Commands (IDE)

In VS Code and JetBrains, type `/` in the chat input to see available slash commands:

| Command | Description |
|---------|-------------|
| `/explain` | Explain the selected code or current file |
| `/fix` | Propose a fix for problems in the selected code |
| `/tests` | Generate unit tests for the selected code |
| `/doc` | Generate documentation for the selected code |
| `/new` | Scaffold a new project or file |
| `/newNotebook` | Create a new Jupyter notebook (VS Code) |
| `/terminal` | Explain the last terminal command or error |

## Chat Participants / Context Variables (VS Code)

Prefix your message with a participant or variable to scope Copilot's context:

| Participant/Variable | Description |
|----------------------|-------------|
| `@workspace` | Ask about the entire project/workspace |
| `@vscode` | Ask about VS Code settings and commands |
| `@terminal` | Ask about the terminal or last command |
| `#file` | Reference a specific file |
| `#selection` | Reference the current selection |
| `#codebase` | Ask about the whole codebase (index-based) |

## Using Copilot Chat in GitHub (Website)

On github.com, open the Copilot Chat icon in the sidebar. You can:

- Ask general coding questions.
- Reference repositories, files, issues, PRs, or commits with `#` (e.g., `#repo`, `#file`, `#issue`).
- Use the GitHub MCP server (pre-configured) to perform actions like creating branches or merging PRs.

## Using Copilot Chat in GitHub Mobile

Available in the GitHub Mobile app. Ask coding questions or get help with repository-specific code.

## Using Copilot Chat in Windows Terminal

Windows Terminal Canary integrates Copilot Chat via the Terminal Chat interface. Ask for shell command suggestions or explanations inline.

## AI Models for Copilot Chat

You can switch the AI model used by Copilot Chat. Options vary by plan but may include:

- GPT-4o / GPT-4.1
- Claude Sonnet / Claude Opus (Anthropic)
- Gemini (Google)
- o3 (OpenAI) — on Pro+ and above

Different models may perform better for different tasks (coding, reasoning, long-context).

## Extending Copilot Chat with MCP

The Model Context Protocol (MCP) is an open standard that lets you connect Copilot Chat to external data sources and tools.

- **VS Code & JetBrains**: Configure MCP servers in settings to provide additional context (databases, APIs, custom tools).
- **GitHub.com**: The GitHub MCP server is pre-configured, enabling Copilot to create branches, merge PRs, and more.

See [Extending GitHub Copilot Chat with MCP](https://docs.github.com/en/copilot/how-tos/context/model-context-protocol/extending-copilot-chat-with-mcp).

## Customizing Chat Responses

Instead of repeating context in every message, you can create **custom instructions** that are automatically added to every prompt:

- **Personal instructions**: Your individual preferences (e.g., "Always respond in Portuguese").
- **Repository instructions**: Stored in `.github/copilot-instructions.md`; applies to all contributors.
- **Organization instructions**: Set by org owners; applies to all org members (Copilot Enterprise).

## Tips for Better Chat Results

- Be specific: "What does the `parseDate` function do?" not "What does this do?"
- Provide context: reference the relevant file or selection before asking.
- Iterate: if you don't get what you want, refine your prompt and ask again.
- Use threads for separate tasks; delete old messages that are no longer relevant.

## Limitations

- Copilot Chat may not always produce correct or optimal solutions.
- Generated code may contain bugs or security vulnerabilities — always review before using in production.
- Responses are non-deterministic; the same prompt can produce different results each time.

## References

- [About GitHub Copilot Chat](https://docs.github.com/en/copilot/concepts/chat)
- [Asking GitHub Copilot questions in your IDE](https://docs.github.com/en/copilot/how-tos/chat-with-copilot/chat-in-ide)
- [Asking GitHub Copilot questions in GitHub](https://docs.github.com/en/copilot/how-tos/chat-with-copilot/chat-in-github)
- [Getting started with prompts for GitHub Copilot Chat](https://docs.github.com/en/copilot/how-tos/chat-with-copilot/get-started-with-chat)
- [GitHub Copilot Chat Cookbook](https://docs.github.com/en/copilot/tutorials/copilot-chat-cookbook)
