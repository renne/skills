# Nextcloud Coding Style & General Guidelines

Source: https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/index.html

## Overview

Nextcloud's coding standards cover PHP, JavaScript/TypeScript, and general collaborative development practices. The goal is to ensure consistency, readability, and maintainability across the entire Nextcloud ecosystem.

## General Development Guidelines

### Collaborative Development

- **Discuss features first**: Before starting significant work, discuss features on the Nextcloud forums or GitHub issues.
- **Incremental changes**: Keep changes small and focused. Large, complex pull requests are harder to review and merge.
- **Consensus-driven decisions**: Technical decisions are reached collaboratively, with maintainers having significant input.
- **Use GitHub**: Manage code with GitHub — branch features, and submit pull requests for review.

### Branch Strategy

- Develop features on separate branches.
- Only merge to the main branch when features are fully ready and tested.
- Ensure all pull requests pass linting and automated tests before merge.

## Code Review & CI

- Every pull request undergoes rigorous code review.
- Automated CI checks include: static analysis, linting, and tests.
- All CI checks must pass before a PR is considered for merge.

## PHP Coding Standards

### Formatting

- Use **PHP CS Fixer** with the shared Nextcloud coding standards config.
- Begin PHP files with `<?php`; omit the closing `?>` tag.
- Files must end with a single blank line.
- Use tabs for indentation (Nextcloud-specific convention).
- Always use curly braces `{}` for control structures, even for single-line bodies.
- Use blank lines after the namespace declaration and opening the PHP tag.

### Naming

| Element | Convention | Example |
|---------|------------|---------|
| Classes / Interfaces | UpperCamelCase | `MyController` |
| Methods / Functions | lowerCamelCase | `getUserName()` |
| Variables | lowerCamelCase | `$userId` |
| Constants | UPPER_CASE | `MAX_SIZE_LIMIT` |

- Do **not** prefix private members with underscores.

### Operators

- Use **strict comparison** (`===`, `!==`) instead of loose comparison (`==`, `!=`).
- No spaces after function names and inside parentheses.
- Single space around binary operators.
- Avoid Yoda conditions (`if (null === $var)`).

### Control Structures

- Always use curly braces for control structures.
- Split long `if`/`else` conditions across multiple lines for readability.
- In `switch` statements, always use `break`.

```php
// Good
if ($userId === $currentUser->getId()) {
    $this->doSomething();
}

// Bad — missing braces
if ($userId === $currentUser->getId())
    $this->doSomething();
```

## JavaScript/TypeScript Coding Standards

### Formatting

- Use **ESLint** with Nextcloud's shared ESLint configuration for JavaScript and TypeScript.
- Source code goes in `src/`, compiled assets in `js/` and `css/`.

### Framework

- Use **Vue.js** for UI app development.
- TypeScript is recommended for type safety and better tooling.

### Naming

| Element | Convention | Example |
|---------|------------|---------|
| Functions / Variables | camelCase | `callHttpApi`, `userId` |
| Classes / Vue Components | PascalCase | `FileListEntry` |
| Vue sub-components | `ComponentNameSubpart` | `FileListEntryIcon` |
| Abbreviations | Capitalize first letter only | `callHttpApi` (not `callHTTPAPI`) |

### Best Practices

- **No globals**: Avoid creating global variables; use a namespace like `OCA.YourApp` instead.
- Use strict mode.
- Write modular code.
- Avoid single-word component names to prevent conflicts with HTML tags.
- Use Vue.js conventions for file naming and component organization.

## Unit Tests

### PHP Tests

- All unit tests must extend `\Test\TestCase`.
- Data providers must be named with a `Data` suffix (e.g., `dummyData`) and must not start with `test`.
- Use the `@dataProvider` annotation:

```php
namespace Test;

class Dummy extends \Test\TestCase {
    public function dummyData(): array {
        return [
            [1, true],
            [2, false],
        ];
    }

    /** @dataProvider dummyData */
    public function testDummy(int $input, bool $expected): void {
        $this->assertEquals($expected, \Dummy::method($input));
    }
}
```

### JavaScript Tests

- Use Jest for JavaScript/TypeScript unit tests.
- Follow Vue Test Utils conventions for component testing.

## Automated Tooling

| Tool | Purpose |
|------|---------|
| `nextcloud/coding-standard` | PHP CS Fixer rules for PHP |
| ESLint (Nextcloud config) | Linting for JavaScript/TypeScript |
| Psalm / PHPStan | Static analysis for PHP |

### PHP CS Fixer Integration

Install via Composer:
```bash
composer require --dev nextcloud/coding-standard
```

Run checks:
```bash
composer run cs:check
composer run cs:fix
```

## References

- [Nextcloud Coding Style & General Guidelines](https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/index.html)
- [Nextcloud PHP Coding Standards](https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/php.html)
- [Nextcloud JavaScript/TypeScript Standards](https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/javascript.html)
- [nextcloud/coding-standard on GitHub](https://github.com/nextcloud/coding-standard)
