---
name: php-coding-standards
description: Nextcloud PHP coding standards enforced by PHP CS Fixer with the nextcloud/coding-standard package. Use when writing, reviewing, or auto-formatting Nextcloud PHP code, checking naming conventions, strict comparisons, control structures, PHPDoc, and namespace imports.
---
# Nextcloud PHP Coding Standards

Source: https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/php.html

## Overview

Nextcloud PHP coding standards are enforced using **PHP CS Fixer** with a shared configuration from the `nextcloud/coding-standard` package. These standards align with PSR-1 and PSR-12 with some Nextcloud-specific conventions.

## File Formatting

- Begin PHP files with `<?php`.
- Omit the closing `?>` tag for files containing only PHP to avoid accidental whitespace output.
- Files must end with a single blank line.
- A blank line must follow the namespace declaration.
- All PHP files must use UTF-8 encoding.

```php
<?php

namespace OCA\MyApp\Controller;

use OCP\AppFramework\Controller;

class MyController extends Controller {
    // ...
}
```

## Indentation

- Use **tabs** for indentation (Nextcloud-specific, unlike many PSR-12 codebases that use spaces).
- Multiline arrays must be properly indented.

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Classes | UpperCamelCase (PascalCase) | `MyController`, `UserService` |
| Interfaces | UpperCamelCase | `IUserService` |
| Methods | lowerCamelCase | `getUserName()`, `findById()` |
| Variables | lowerCamelCase | `$userId`, `$currentUser` |
| Constants | UPPER_CASE_WITH_UNDERSCORES | `MAX_SIZE_LIMIT` |
| Private members | No underscore prefix | `$this->name` (not `$this->_name`) |

## Operators and Expressions

### Strict Comparison

Always use strict comparison operators to avoid PHP's type coercion:

```php
// Good
if ($userId === $currentUser->getId()) {
    // ...
}

// Bad
if ($userId == $currentUser->getId()) {
    // ...
}
```

### Spaces

- No spaces after function names or inside parentheses.
- Single space around binary operators.

```php
// Good
$result = $a + $b;
doSomething($arg1, $arg2);

// Bad
$result=$a+$b;
doSomething( $arg1 , $arg2 );
```

### Avoid Yoda Conditions

```php
// Good
if ($variable === null) { }

// Bad (Yoda)
if (null === $variable) { }
```

## Control Structures

- Always use curly braces `{}` even for single-line bodies.
- Split long `if`/`else` conditions across multiple lines for readability.
- In `switch` statements, always use `break`.

```php
// Good
if ($condition) {
    doSomething();
}

// Bad — no braces
if ($condition)
    doSomething();

// Switch with break
switch ($value) {
    case 'foo':
        doFoo();
        break;
    case 'bar':
        doBar();
        break;
    default:
        doDefault();
        break;
}
```

## Long Conditionals

Split long conditions across multiple lines:

```php
// Good
if ($this->isLoggedIn()
    && $this->hasPermission('read')
    && $resource->isAccessible()
) {
    // ...
}
```

## Documentation

Use PHPDoc for public methods and APIs:

```php
/**
 * Returns the user by ID.
 *
 * @param int $userId The user's ID
 * @return IUser|null The user or null if not found
 * @throws \OCP\AppFramework\Db\DoesNotExistException
 */
public function getUserById(int $userId): ?IUser {
    return $this->userMapper->find($userId);
}
```

## Unit Tests

### Test Class Structure

All unit tests must extend `\Test\TestCase`, which automatically handles cleanups after each test:

```php
<?php

namespace Test;

class Dummy extends \Test\TestCase {

    public function setUp(): void {
        parent::setUp();
        // Set up fixtures
    }

    public function testBasicBehavior(): void {
        $this->assertTrue(true);
    }
}
```

### Data Providers

When testing with multiple input sets, use data providers:
- The data provider method name must end with `Data`.
- The data provider method must **not** start with `test`.
- Use the `@dataProvider` annotation.

```php
public function dummyData(): array {
    return [
        [1, true],
        [2, false],
        [3, true],
    ];
}

/** @dataProvider dummyData */
public function testDummy(int $input, bool $expected): void {
    $this->assertEquals($expected, Dummy::method($input));
}
```

## Automated Formatting with PHP CS Fixer

### Installation

```bash
composer require --dev nextcloud/coding-standard
```

### Usage

Add scripts to `composer.json`:
```json
{
    "scripts": {
        "cs:check": "php-cs-fixer fix --dry-run --diff",
        "cs:fix": "php-cs-fixer fix"
    }
}
```

Run:
```bash
composer run cs:check  # Check without modifying
composer run cs:fix    # Auto-fix issues
```

### What It Enforces

- Array indentation
- Proper ordering of imports (`use` statements)
- Removal of unused imports
- Single blank line after imports
- PSR conformance
- Correct spacing

## Namespace and Imports

```php
<?php

namespace OCA\MyApp\Service;

use OCP\IUser;
use OCP\IUserManager;

class UserService {
    public function __construct(
        private IUserManager $userManager,
    ) {
    }
}
```

- Group `use` statements together.
- Order imports alphabetically.
- Remove unused imports.

## References

- [Nextcloud PHP Coding Standards](https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/php.html)
- [nextcloud/coding-standard on GitHub](https://github.com/nextcloud/coding-standard)
- [PHP CS Fixer Rules](https://cs.symfony.com/doc/rules/index.html)
- [PSR-12: Extended Coding Style](https://www.php-fig.org/psr/psr-12/)
