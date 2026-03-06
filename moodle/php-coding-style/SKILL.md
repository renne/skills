---
name: php-coding-style
description: Moodle PHP coding style guide based on PSR-12/PSR-1 with Moodle-specific rules. Use when writing or reviewing PHP code in Moodle core or plugins, checking indentation, naming conventions, braces, control structures, PHPDoc comments, and SQL formatting.
---
# Moodle PHP Coding Style

Source: https://moodledev.io/general/development/policies/codingstyle

## Overview

Moodle's PHP coding style is based on PSR-12 and PSR-1 with Moodle-specific adaptations. The primary goals are consistency, readability, and tool-friendliness to promote IDE integration and code maintenance across many contributors.

## File Formatting

- Always use `<?php` opening tags. Never use short tags (`<?`).
- **Never** include the closing `?>` tag at the end of pure PHP files to avoid accidental whitespace output.
- Use UTF-8 encoding without BOM.
- Use Unix-style LF line endings only.
- No trailing whitespace at the end of any line (spaces or tabs before the newline character are not allowed).
- Files must end with a **single LF newline** — no extra blank lines at the end. (This final LF is not trailing whitespace; it terminates the last line.)

## Indentation

- Always use **4 spaces** for indentation — never tabs.
- Editors must be configured to replace tab characters with spaces.
- Do not indent at the main script level; only indent inside functions, classes, or control structures.
- Continuation lines (e.g., wrapped method arguments, chained calls) are indented by 4 extra spaces from their parent.

## Line Length

- Recommended maximum: **132 characters**.
- Absolute maximum: **180 characters** (except for certain strings in language files where a single long string is permitted).
- When breaking lines, indent wrapped lines with an extra 4 spaces.

## Braces

- Opening braces for classes, functions, and control structures go on the **same line** as the declaration.
- Always use braces `{}` even for single-line control structure bodies.

```php
// Correct
if ($condition) {
    do_something();
}

// Incorrect
if ($condition)
    do_something();
```

## Naming Conventions

### Variables
- Use `lowercase_with_underscores` (e.g., `$course_name`, `$user_id`).
- Avoid camelCase and Hungarian notation.
- Make names meaningful and descriptive; avoid abbreviations unless obvious.

### Functions
- Use `lowercase_with_underscores` (e.g., `get_student_data()`, `load_user_profile()`).
- Function names should be descriptive, starting with a verb.

### Classes

> **Moodle-specific deviation from PSR-12:** Moodle uses `lower_case_with_underscores` for class names, **not** PascalCase/StudlyCaps. This is a long-standing Moodle convention; you may encounter older code using PascalCase, but all new code should follow the Moodle style.

```php
// Correct (Moodle style)
class some_custom_class { ... }
class quiz_attempt_manager { ... }

// Incorrect (PSR-12 style — do NOT use in Moodle)
class MyClass { ... }
class QuizAttemptManager { ... }
```

Plugin and component names are typically used as a prefix (e.g., `mod_quiz_attempt`).

### Constants
- Use `ALL_CAPS_WITH_UNDERSCORES` (e.g., `DEFAULT_PAGE_SIZE`, `MAX_USER_COUNT`).
- Use `define('CONSTANT_NAME', value)` or class constants (`const CONSTANT_NAME = value;`).

### Methods and Properties
- Methods use camelCase (following PSR-12).
- Visibility order: visibility keyword, then `static`, then `abstract`/`final`, then `function`.

## Control Structures

- Control keywords (`if`, `for`, `foreach`, `while`, `switch`, etc.) have a **space after them**: `if ($condition) {`.
- Always add a space after commas in function calls and arrays.
- No space before the opening parenthesis in function calls.

```php
// Correct
foreach ($items as $item) {
    process($item);
}

// Function call
myFunction($param1, $param2);
```

## Namespacing and Imports

- The namespace declaration comes first, followed by a blank line, then `use` declarations.
- Group all `use` statements together after the namespace; separate with a blank line before the class definition.

```php
<?php
namespace core\example;

use core\exception\moodle_exception;
use moodle_database;

class my_class {
    // ...
}
```

## Comments and Documentation

- Document all functions, classes, and files with PHPDoc comments.
- Use `//` for short inline comments.
- Multi-line comments use `/* ... */` fully aligned.
- Include `@param`, `@return`, `@throws`, and `@deprecated` tags as appropriate.

```php
/**
 * Returns the user's full name.
 *
 * @param int $userid The user ID
 * @return string The full name
 */
function get_user_fullname(int $userid): string {
    // ...
}
```

## SQL Queries

- Follow Moodle-specific indentation and alignment for complex SQL.
- Queries should be readable with variables safely interpolated.
- Use Moodle's database API (`$DB->get_records()`, `$DB->execute()`, etc.) rather than raw SQL.

```php
$sql = "SELECT u.id, u.firstname, u.lastname
          FROM {user} u
          JOIN {role_assignments} ra ON ra.userid = u.id
         WHERE u.deleted = 0
           AND ra.roleid = :roleid";
$params = ['roleid' => $roleid];
$users = $DB->get_records_sql($sql, $params);
```

## Miscellaneous

- Class files should not execute code outside method definitions (no side effects).
- Use strict type comparisons where appropriate.
- Follow PSR-12 where Moodle's guide does not specify.
- **Variable assignment alignment**: multiple spaces before `=` are allowed to visually align a group of related assignments:
  ```php
  $firstname = 'Jane';
  $lastname  = 'Doe';
  $email     = 'jane@example.com';
  ```
- **Wrapping long conditions**: for multi-line `if` conditions, add 4 extra spaces of indentation to continuation lines to distinguish them from the body. Alternatively, extract sub-expressions into helper variables:
  ```php
  if ($condition_one
          && $condition_two
          && $condition_three) {
      do_something();
  }
  
  // Or use helper variables for clarity:
  $is_valid = $condition_one && $condition_two;
  if ($is_valid && $condition_three) {
      do_something();
  }
  ```

## Tools

- **Moodle Code Checker** (PHP_CodeSniffer): [moodlehq/moodle-cs](https://github.com/moodlehq/moodle-cs) — enforces coding style automatically.
- **Moodle PHPdoc Checker**: validates PHPDoc comments.
- Run both before submitting code contributions.

## References

- [Official Moodle Coding Style Guide](https://moodledev.io/general/development/policies/codingstyle)
- [PSR-12: Extended Coding Style](https://www.php-fig.org/psr/psr-12/)
- [PSR-1: Basic Coding Standard](https://www.php-fig.org/psr/psr-1/)
- [Moodle Coding Style (moodle-cs)](https://github.com/moodlehq/moodle-cs)
