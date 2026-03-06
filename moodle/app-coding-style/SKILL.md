---
name: app-coding-style
description: Moodle App (Angular/TypeScript) coding style guide. Use when writing or reviewing TypeScript/Angular code for the Moodle mobile app, including ESLint compliance, async/await patterns, guard clauses, Angular component conventions, and Ionic UI usage.
---
# Moodle App Coding Style (TypeScript / Angular)

Source: https://moodledev.io/general/development/policies/codingstyle-moodleapp

## Overview

The Moodle App is built using Angular and the Ionic Framework. The coding style for the app is enforced primarily through ESLint. Warnings are treated with the same severity as errors, and all new code must conform to the rules unless there is a clear, documented exception.

## Core Principles

- Prioritize simplicity and readability.
- Integrate a linter (ESLint) in your development environment.
- New code must always conform to ESLint rules.
- Prefer refactoring over disabling linting rules.

## TypeScript Specifics

### Async/Await

Prefer `async/await` over Promises with `.then()`/`.catch()`/`.finally()`. Do **not** mix both styles within a single function.

```typescript
// Good
async function greet() {
    const response = await fetch('/profile.json');
    const data = await response.json();
    alert(`Hello, ${data.name}!`);
}

// Bad — mixing async/await with .then()
async function greet() {
    const response = await fetch('/profile.json');
    return response.json().then(data => {
        alert(`Hello, ${data.name}!`);
    });
}
```

### Guard Clauses (If Guards)

Use top-level guard clauses to reduce nested indentation and handle edge cases up front. This improves readability by avoiding deeply nested `if` blocks.

```typescript
// Good — guard clause at the top
getPrivateInfo() {
    if (!this.isLoggedIn()) {
        throw new Error('Please, log in!');
    }
    return this.privateInfo;
}

// Bad — deeply nested
getPrivateInfo() {
    if (this.isLoggedIn()) {
        return this.privateInfo;
    } else {
        throw new Error('Please, log in!');
    }
}
```

### Disabling ESLint Rules

- Prefer refactoring code to comply with linting rules rather than disabling them.
- Inline `// eslint-disable` comments are permitted but strongly discouraged.
- If a rule must be disabled, always include a comment explaining why.

## Angular Conventions

- Follow Angular best practices for component, service, and module organization.
- Use Angular's dependency injection system for service management.
- Keep components focused on presentation logic; move business logic to services.
- Use `OnPush` change detection strategy where possible for performance.

## Ionic Framework

- Use Ionic components for mobile UI (e.g., `<ion-content>`, `<ion-list>`, `<ion-item>`).
- Do not use Bootstrap or other web-only UI frameworks.
- Follow Ionic's theming and styling conventions.

## General TypeScript Style

- Use explicit type annotations for public APIs and function signatures.
- Avoid using `any` type; prefer specific types or generics.
- Use `const` by default, `let` only when reassignment is needed; avoid `var`.
- Use template literals instead of string concatenation.
- Use destructuring for objects and arrays where it improves readability.

## Tools

- **ESLint**: The primary linting tool. Warnings are treated as errors.
- **VSCode**: The recommended editor. The repository includes VSCode-specific settings.
- ESLint plugins also warn about deprecated or removed APIs.

## References

- [Moodle App Coding Style Guide](https://moodledev.io/general/development/policies/codingstyle-moodleapp)
- [Moodle App Development Guide](https://moodledev.io/general/app/development/development-guide)
- [General Moodle Coding Style (PHP)](https://moodledev.io/general/development/policies/codingstyle)
