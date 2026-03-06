# Moodle App Plugins Development Guide

Source: https://moodledev.io/general/app/development/plugins-development-guide

## Overview

To extend the Moodle App's functionality with your plugin, you primarily write PHP — with optional JavaScript for advanced features — just like standard Moodle plugin development. The main interface for exposing your plugin to the mobile app is the `db/mobile.php` file.

## Key Concepts

- **No extra web service functions required** for basic mobile support. Plugin content is served via the `tool_mobile_get_content` web service function.
- Templates and UI use **Ionic components** (not Bootstrap or standard web HTML).
- Advanced features can include JavaScript passed back from PHP.

## Step 1: Create `db/mobile.php`

This file declares which handlers (features or pages) your plugin exposes to the mobile app.

```php
<?php
defined('MOODLE_INTERNAL') || die();

$addons = [
    'local_myplugin' => [
        'handlers' => [
            'main' => [
                'delegate' => 'CoreMainMenuDelegate',
                'method'   => 'view_main',
                'displaydata' => [
                    'title' => 'myplugin',
                    'icon'  => 'aperture', // Ionic icon name
                ],
            ],
        ],
        'lang' => [
            ['myplugin', 'local_myplugin'],
        ],
    ],
];
```

### Handler Properties

| Property | Description |
|----------|-------------|
| `delegate` | Where this handler appears (e.g., `CoreMainMenuDelegate`, `CoreCourseModuleDelegate`) |
| `method` | PHP function in `classes/output/mobile.php` that provides the content |
| `displaydata` | UI details such as title and icon |

## Step 2: Create `classes/output/mobile.php`

This PHP file defines functions the app will call. Each function must return a specific array format.

```php
<?php
namespace local_myplugin\output;

class mobile {

    /**
     * Main page for mobile app.
     *
     * @param array $args Arguments from the app
     * @return array Template and data to render
     */
    public static function view_main(array $args): array {
        global $OUTPUT;

        return [
            'templates' => [
                [
                    'id'   => 'main',
                    'html' => '<ion-content><h1>Hello, Mobile!</h1></ion-content>',
                ],
            ],
            'otherdata' => [
                'examplekey' => 'examplevalue',
            ],
        ];
    }
}
```

### Return Array Keys

| Key | Description |
|-----|-------------|
| `templates` | Array of template objects with `id` and `html` |
| `otherdata` | Additional data passed to the template as variables |
| `files` | Array of files to pass to the template |
| `javascript` | JavaScript code to execute in the app |

## Using Ionic Components in Templates

All HTML in the `html` field must use Ionic components, not Bootstrap or standard HTML forms:

```html
<ion-content>
    <ion-list>
        <ion-item *ngFor="let item of items">
            <ion-label>{{ item.name }}</ion-label>
        </ion-item>
    </ion-list>
</ion-content>
```

## Using Angular Syntax

Templates support Angular template syntax:
- `*ngFor`, `*ngIf` directives
- `{{ variable }}` interpolation
- `(click)` event binding

## Web Services

### Basic Usage

For basic mobile support, you do not need new web service functions. The `tool_mobile_get_content` function handles content delivery automatically.

### Custom Web Services

If you need custom data fetching or actions:
1. Define web service functions in `db/services.php`.
2. Implement the functions in `classes/external/`.
3. Call them from your mobile output methods.

## Plugin Naming and Structure (Frankenstyle)

Moodle plugins follow the **Frankenstyle** naming convention:
- Format: `plugintype_pluginname` (e.g., `mod_example`, `local_myplugin`, `block_myblock`)
- Essential files: `version.php`, `lang/`, `db/`

```
local_myplugin/
├── classes/
│   └── output/
│       └── mobile.php      # Mobile output class
├── db/
│   ├── mobile.php          # Mobile handlers declaration
│   └── services.php        # (Optional) custom web services
├── lang/
│   └── en/
│       └── local_myplugin.php
└── version.php
```

## Testing

- Use hosted Moodle App demo instances for initial testing.
- Use Docker for a local Moodle environment.
- Use the official Moodle App (available on iOS/Android) for device testing.
- Running the app from source is only necessary for advanced custom features.

## Best Practices

- Follow Moodle's PHP coding standards.
- Use the Plugin Skeleton Generator for scaffolding new plugins.
- Check code quality with the Moodle Code Checker and PHPdoc Checker.
- Start simple: try a "Hello World" example before building complex features.
- Consult the Moodle community forums for questions and best practices.

## Plugin Review Checklist

Before submitting to the Moodle Plugin Directory:
- Pass the Moodle Code Checker.
- Pass the Moodle PHPdoc Checker.
- Provide a working `version.php` with correct version numbering.
- Include language string files.
- Write unit tests for your PHP code.

## References

- [Moodle App Plugins Development Guide](https://moodledev.io/general/app/development/plugins-development-guide)
- [Moodle App Plugins API Reference](https://moodledev.io/general/app/development/plugins-development-guide/api-reference)
- [Moodle App Repository](https://github.com/moodlehq/moodleapp)
- [Make Your Plugin Moodle App Compatible (Moodle Academy)](https://www.moodle.academy/course/view.php?id=71)
