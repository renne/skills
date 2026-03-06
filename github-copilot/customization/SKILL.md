---
name: customization
description: GitHub Copilot customization guide covering personal custom instructions, repository custom instructions (.github/copilot-instructions.md and .github/instructions/*.instructions.md), organization custom instructions, prompt engineering strategies (start general, give examples, break complex tasks, avoid ambiguity), and how to write effective custom instruction files. Use when configuring Copilot to follow project coding standards, adding persistent instructions, or improving the quality of Copilot responses.
---
# GitHub Copilot Customization

Source: https://docs.github.com/en/copilot/concepts/prompting/response-customization

## Overview

GitHub Copilot can be customized to follow your project's coding standards, preferred frameworks, and team conventions — without repeating context in every prompt.

There are two main ways to customize Copilot:
1. **Custom instructions** — persistent context added automatically to every prompt.
2. **Prompt engineering** — strategies for writing better one-off prompts.

## Custom Instructions

### Personal Instructions

- Apply to all Copilot Chat conversations **for you** across GitHub.com.
- Set via the Copilot Chat page on GitHub.com (popup).
- Examples:
  - `Always respond in Spanish.`
  - `Explain a single concept per line. Be clear and concise.`

### Repository Custom Instructions

Stored in the repository and apply to **all contributors** working in that repository's context.

#### Repository-wide instructions

File: `.github/copilot-instructions.md`

Applied to all requests in the context of the repository. Copilot code review reads only the first 4,000 characters.

Example:
```markdown
# Project Overview
This is a React/TypeScript web app for task management using Node.js, Express, and MongoDB.

## Folder Structure
- `/src` — Frontend React source
- `/server` — Node.js backend
- `/docs` — API specs and guides

## Libraries and Frameworks
- React 18, Tailwind CSS
- Node.js 20, Express
- MongoDB, Mongoose

## Coding Standards
- Use TypeScript strict mode
- Use semicolons; prefer single quotes for strings
- Use functional React components with hooks
- Use arrow functions for callbacks
- Write JSDoc comments for all exported functions
```

#### Path-specific instructions

Files: `.github/instructions/*.instructions.md`

Applied only to files matching specified paths — useful to avoid overloading repository-wide instructions with language-specific rules.

Example file `.github/instructions/python.instructions.md`:
```markdown
---
applyTo: "**/*.py"
---
- Follow PEP 8 formatting.
- Use type annotations for all function signatures.
- Prefer `pathlib.Path` over `os.path`.
```

#### Agent instructions

Files: `AGENTS.md`, `CLAUDE.md`, or `GEMINI.md` in the repository root.

Similar to repository-wide instructions; some Copilot features use these automatically.

### Organization Custom Instructions (Copilot Enterprise)

- Set by organization owners in GitHub org settings.
- Apply to all org members regardless of which Copilot plan they use.
- Ideal for org-wide preferences and security policies.
- Examples:
  - `Always respond in German.`
  - `For security questions, refer to the Security Knowledge Base or #security on Slack.`
  - `Do not generate code blocks in responses.`

### Precedence Order

When multiple instruction types apply, higher entries take precedence:

1. **Personal** instructions
2. **Path-specific** repository instructions (`.github/instructions/*.instructions.md`)
3. **Repository-wide** instructions (`.github/copilot-instructions.md`)
4. **Agent** instructions (`AGENTS.md`, etc.)
5. **Organization** instructions

## Writing Effective Custom Instructions

- Keep instructions **short and self-contained** — one statement per concept.
- Include a **project overview** (purpose, goals, tech stack).
- Specify the **folder structure** of the repository.
- List **coding standards** (naming, formatting, best practices).
- List **tools and frameworks** with version numbers where relevant.
- Avoid instructions that:
  - Reference external URLs or documents (e.g., "see styleguide.md in another repo").
  - Dictate exact response length or word choice.
  - Apply only to narrow edge cases.

## Prompt Engineering Strategies

### Start General, Then Get Specific

First describe the goal broadly, then add constraints:
```
Write a JavaScript function that checks if a number is prime.
The function should take an integer and return true/false.
It should throw an error if the input is not a positive integer.
```

### Give Examples

Provide example input/output or example implementations:
```
Write a Go function to find all dates in a string and return them as a slice.
Supported formats: 05/02/24, 05-02-2024, 5/2/24, etc.
Example: findDates("appointment on 11/14/2023 and 12-1-23") → ["11/14/2023", "12-1-23"]
```

### Break Complex Tasks into Smaller Tasks

Instead of one complex prompt, break it into sequential simpler prompts:
1. "Write a function to generate a 10×10 grid of random letters."
2. "Write a function to find words in that grid given a word list."
3. "Combine both into a word-search puzzle generator."

### Avoid Ambiguity

- Bad: "What does this do?"
- Good: "What does the `parseUserToken` function do?"

If using a less-common library, describe what it does. If you want a specific library, import it at the top of the file first.

### Indicate Relevant Code

- Open files you want Copilot to use as context; close unrelated files.
- In VS Code, use `@workspace` to reference the whole project or `#file` to reference a specific file.
- Highlight code before asking about it.

### Experiment and Iterate

- If suggestions aren't right, delete and rephrase.
- Reference the previous response: "Based on your previous answer, also handle the error case where…"
- Use threads for separate tasks; delete messages that are no longer relevant.

### Follow Good Coding Practices

Copilot gives better results when your existing code is clean:
- Consistent style and patterns
- Descriptive names for variables/functions
- Comments explaining intent
- Modular, well-scoped components
- Existing unit tests

## References

- [About customizing GitHub Copilot responses](https://docs.github.com/en/copilot/concepts/prompting/response-customization)
- [Prompt engineering for GitHub Copilot Chat](https://docs.github.com/en/copilot/concepts/prompting/prompt-engineering)
- [Adding repository custom instructions for GitHub Copilot](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)
- [Adding personal custom instructions for GitHub Copilot](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-personal-instructions)
- [Copilot customization cheat sheet](https://docs.github.com/en/copilot/reference/customization-cheat-sheet)
