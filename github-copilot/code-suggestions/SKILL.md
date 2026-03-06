---
name: code-suggestions
description: GitHub Copilot inline code suggestions guide covering ghost text autocomplete, next edit suggestions, accepting or rejecting suggestions, supported programming languages (Python, JavaScript, TypeScript, Ruby, Go, C#, C++, and 30+ more), and how to get better suggestions. Use when working with Copilot inline completions in an IDE, configuring suggestion behavior, or troubleshooting why suggestions are not appearing.
---
# GitHub Copilot Code Suggestions

Source: https://docs.github.com/en/copilot/concepts/completions/code-suggestions

## Overview

GitHub Copilot provides AI-powered inline code suggestions as you type in your IDE. It works like an intelligent autocomplete, offering single-line, multi-line, and whole-function completions based on your code context.

Supported IDEs: **VS Code, Visual Studio, JetBrains IDEs, Xcode, Vim/Neovim, Eclipse, Azure Data Studio**.

## Types of Suggestions

### Ghost Text Suggestions

- Appear inline as you type, in a lighter "ghost text" color.
- Can span a single symbol, a line, or multiple lines.
- Triggered automatically as you type, or manually with `Alt+\` (VS Code/JetBrains) / `Option+\` (macOS).
- You can also write a **natural language comment** describing what you want, and Copilot will suggest the implementation below it.

### Next Edit Suggestions (Public Preview)

Available in VS Code, Visual Studio, Xcode, Eclipse.

- Based on recent edits, Copilot predicts **where you are likely to edit next** and suggests what to type there.
- Useful for cascading changes: rename a variable in one place, Copilot suggests renaming it everywhere.
- Enable in settings: `GitHub Copilot: Enable Next Edit Suggestions`.

## Accepting, Cycling, and Dismissing Suggestions

| Action | VS Code / Linux | macOS |
|--------|----------------|-------|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss suggestion | `Esc` | `Esc` |
| Accept word by word | `Ctrl+Right` | `Cmd+Right` |
| See next suggestion | `Alt+]` | `Option+]` |
| See previous suggestion | `Alt+[` | `Option+[` |
| Open suggestion panel | `Ctrl+Enter` | `Ctrl+Enter` |

In JetBrains IDEs, the default accept key is `Tab`; it can be changed in Keymap settings.

## Supported Programming Languages

Copilot works with all programming languages but is especially strong for:

**Tier 1**: Python, JavaScript, TypeScript, Ruby, Go, C#, C++

**Also well supported**: C, Clojure, CSS, Dart, Dockerfile, Elixir, Go, Haskell, HTML, Java, Julia, Jupyter Notebook, Kotlin, Lua, MATLAB, Objective-C, Perl, PHP, PowerShell, R, Rust, Scala, Shell, Swift, TeX, Vue

Copilot also assists with SQL queries, API/framework code, and infrastructure-as-code (e.g., Terraform, Kubernetes YAML).

## Getting Better Suggestions

- **Open relevant files**: Copilot uses open files as context. Close irrelevant files.
- **Write descriptive comments**: Describe what a function should do before implementing it.
- **Use descriptive names**: Clear variable and function names help Copilot understand intent.
- **Include type hints** (Python/TypeScript): Copilot uses type information to generate more accurate code.
- **Write code top-down**: Define the structure first; Copilot fills in implementations.
- **Show examples**: Add example usages or unit tests before the function — Copilot will match the pattern.

## Code Referencing

GitHub Copilot checks each suggestion for matches with publicly available code. If a match is found:

- The suggestion may be **discarded** (if "Suggestions matching public code" is set to "Block").
- Or the suggestion is shown with a **code reference** (license and source) that you can inspect.

Configure this in your GitHub Copilot settings under "Suggestions matching public code."

## AI Model for Inline Suggestions

The default model for inline suggestions is **GPT-4.1 Copilot**, trained on high-quality public GitHub repositories covering 30+ languages.

In VS Code, Visual Studio, and JetBrains IDEs (with appropriate plan), you can switch the inline suggestion model via a model picker. Changing the model does **not** affect Copilot Chat or next edit suggestions.

## Enabling/Disabling Suggestions

- **Globally**: Toggle Copilot in the IDE status bar (click the Copilot icon).
- **Per language**: In VS Code, set `"github.copilot.enable": { "python": false }` in `settings.json`.
- **Content exclusion**: Organization/enterprise admins can exclude specific files or paths from Copilot using `.github/copilot-instructions.md` or admin policies.

## References

- [GitHub Copilot code suggestions in your IDE](https://docs.github.com/en/copilot/concepts/completions/code-suggestions)
- [Getting code suggestions in your IDE with GitHub Copilot](https://docs.github.com/en/copilot/how-tos/get-code-suggestions/get-ide-code-suggestions)
- [GitHub Copilot code referencing](https://docs.github.com/en/copilot/concepts/completions/code-referencing)
- [Finding public code that matches GitHub Copilot suggestions](https://docs.github.com/en/copilot/how-tos/get-code-suggestions/find-matching-code)
