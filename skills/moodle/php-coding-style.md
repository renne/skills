# Moodle PHP Coding Style

Source: https://moodledev.io/general/development/policies/codingstyle

## Overview

Moodle's PHP coding style is based on PSR-12 and PSR-1 with Moodle-specific adaptations. The primary goals are consistency, readability, and tool-friendliness to promote IDE integration and code maintenance across many contributors.

## File Formatting

- Always use `<?php` opening tags. Never use short tags (`<?`).
- **Never** include the closing `?>` tag at the end of pure PHP files to avoid accidental whitespace output.
- Use UTF-8 encoding without BOM.
- Use Unix-style LF line endings only.
- No trailing whitespace at the end of lines.

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
- Use StudlyCaps / PascalCase with each word capitalized and no separators (e.g., `MyClass`, `QuizAttemptManager`).
- Prefixing with the plugin name or component is typical for namespacing.

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

## Tools

- **Moodle Code Checker** (PHP_CodeSniffer): [moodlehq/moodle-cs](https://github.com/moodlehq/moodle-cs) — enforces coding style automatically.
- **Moodle PHPdoc Checker**: validates PHPDoc comments.
- Run both before submitting code contributions.

## References

- [Official Moodle Coding Style Guide](https://moodledev.io/general/development/policies/codingstyle)
- [PSR-12: Extended Coding Style](https://www.php-fig.org/psr/psr-12/)
- [PSR-1: Basic Coding Standard](https://www.php-fig.org/psr/psr-1/)
- [Moodle Coding Style (moodle-cs)](https://github.com/moodlehq/moodle-cs)
