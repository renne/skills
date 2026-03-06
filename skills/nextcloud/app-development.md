# Nextcloud App Development Guide

Source: https://docs.nextcloud.com/server/19/developer_manual/app/index.html

> **Note**: This skill is based on Nextcloud Server 19 documentation. Core concepts remain applicable to current versions; check the [stable docs](https://docs.nextcloud.com/server/stable/developer_manual/) for the latest API details.

## Overview

Nextcloud apps extend the Nextcloud platform with new features. They follow an MVC (Model-View-Controller) architecture and integrate with Nextcloud's App Framework for routing, dependency injection, and database access.

## App Structure

A typical Nextcloud app has the following directory layout:

```
your-app/
├── appinfo/
│   ├── info.xml          # App metadata (name, version, dependencies)
│   └── routes.php        # URL routing configuration
├── lib/
│   ├── Controller/       # Request handlers
│   ├── Db/               # Database models and mappers
│   ├── Service/          # Business logic
│   └── AppInfo/
│       └── Application.php  # App bootstrap / DI container
├── templates/            # HTML templates
├── js/                   # JavaScript files
├── css/                  # Stylesheets
├── tests/                # Unit and integration tests
└── l10n/                 # Translations
```

## App Metadata (`appinfo/info.xml`)

The `info.xml` file contains essential app metadata:

```xml
<?xml version="1.0"?>
<info>
    <id>mynotes</id>
    <name>My Notes</name>
    <description>A simple notes app</description>
    <version>1.0.0</version>
    <licence>agpl</licence>
    <author>Your Name</author>
    <namespace>MyNotes</namespace>
    <dependencies>
        <nextcloud min-version="25" max-version="32"/>
    </dependencies>
</info>
```

## App Bootstrap (`lib/AppInfo/Application.php`)

The `Application` class registers services and sets up the dependency injection container:

```php
<?php

namespace OCA\MyNotes\AppInfo;

use OCP\AppFramework\App;
use OCP\AppFramework\Bootstrap\IBootContext;
use OCP\AppFramework\Bootstrap\IBootstrap;
use OCP\AppFramework\Bootstrap\IRegistrationContext;

class Application extends App implements IBootstrap {

    public const APP_ID = 'mynotes';

    public function __construct() {
        parent::__construct(self::APP_ID);
    }

    public function register(IRegistrationContext $context): void {
        // Register services, middleware, etc.
    }

    public function boot(IBootContext $context): void {
        // Bootstrap code run at Nextcloud startup
    }
}
```

## Routing (`appinfo/routes.php`)

Define URL routes that map to controller methods:

```php
<?php

return [
    'routes' => [
        // Page routes (render templates)
        ['name' => 'page#index', 'url' => '/', 'verb' => 'GET'],

        // API routes (return JSON)
        ['name' => 'note_api#index', 'url' => '/api/1.0/notes', 'verb' => 'GET'],
        ['name' => 'note_api#create', 'url' => '/api/1.0/notes', 'verb' => 'POST'],
        ['name' => 'note_api#show', 'url' => '/api/1.0/notes/{id}', 'verb' => 'GET'],
        ['name' => 'note_api#update', 'url' => '/api/1.0/notes/{id}', 'verb' => 'PUT'],
        ['name' => 'note_api#destroy', 'url' => '/api/1.0/notes/{id}', 'verb' => 'DELETE'],
    ],
];
```

## Controllers (`lib/Controller/`)

Controllers handle HTTP requests and return responses. They extend `OCP\AppFramework\Controller` or `OCP\AppFramework\ApiController`:

```php
<?php

namespace OCA\MyNotes\Controller;

use OCP\AppFramework\Controller;
use OCP\AppFramework\Http\JSONResponse;
use OCP\IRequest;
use OCA\MyNotes\Service\NoteService;

class NoteApiController extends Controller {

    public function __construct(
        string $appName,
        IRequest $request,
        private NoteService $service,
        private string $userId,
    ) {
        parent::__construct($appName, $request);
    }

    /**
     * @NoAdminRequired
     * @NoCSRFRequired
     */
    public function index(): JSONResponse {
        return new JSONResponse($this->service->findAll($this->userId));
    }

    /**
     * @NoAdminRequired
     * @NoCSRFRequired
     */
    public function create(string $title, string $content): JSONResponse {
        $note = $this->service->create($title, $content, $this->userId);
        return new JSONResponse($note);
    }
}
```

### Common Annotations

| Annotation | Description |
|------------|-------------|
| `@NoAdminRequired` | Accessible to all logged-in users (not just admins) |
| `@NoCSRFRequired` | Disables CSRF token check (for API endpoints) |
| `@PublicPage` | Accessible without login |

## Database (`lib/Db/`)

### Entity

Define a database entity (model):

```php
<?php

namespace OCA\MyNotes\Db;

use OCP\AppFramework\Db\Entity;

class Note extends Entity {

    protected string $title = '';
    protected string $content = '';
    protected string $userId = '';

    public function __construct() {
        $this->addType('id', 'integer');
    }
}
```

### Mapper

The mapper handles database queries for an entity:

```php
<?php

namespace OCA\MyNotes\Db;

use OCP\DB\QueryBuilder\IQueryBuilder;
use OCP\IDBConnection;
use OCP\AppFramework\Db\QBMapper;

class NoteMapper extends QBMapper {

    public function __construct(IDBConnection $db) {
        parent::__construct($db, 'mynotes_notes', Note::class);
    }

    public function findAll(string $userId): array {
        $qb = $this->db->getQueryBuilder();
        $qb->select('*')
            ->from($this->tableName)
            ->where($qb->expr()->eq('user_id', $qb->createNamedParameter($userId)));
        return $this->findEntities($qb);
    }
}
```

### Database Schema (`appinfo/database.xml`)

Define the database schema (used for installation/upgrades):

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<database>
    <name>*dbname*</name>
    <create>true</create>
    <overwrite>false</overwrite>
    <charset>utf8</charset>
    <table>
        <name>*dbprefix*mynotes_notes</name>
        <declaration>
            <field>
                <name>id</name>
                <type>integer</type>
                <autoincrement>1</autoincrement>
                <notnull>true</notnull>
                <length>4</length>
            </field>
            <field>
                <name>user_id</name>
                <type>text</type>
                <notnull>true</notnull>
                <length>200</length>
            </field>
            <field>
                <name>title</name>
                <type>text</type>
                <notnull>true</notnull>
                <length>200</length>
            </field>
            <field>
                <name>content</name>
                <type>clob</type>
                <notnull>false</notnull>
            </field>
        </declaration>
    </table>
</database>
```

## Templates (`templates/`)

Templates render HTML pages:

```php
<?php
/** @var \OCP\IL10N $l */
/** @var array $_ */
script('mynotes', 'script');
style('mynotes', 'style');
?>
<div id="app">
    <div id="app-content">
        <div id="app-content-wrapper">
            <!-- Vue.js app mounts here -->
        </div>
    </div>
</div>
```

## Frontend (JavaScript / Vue.js)

Modern Nextcloud apps use Vue.js for the frontend:

- Place source code in `src/`.
- Compiled assets go in `js/` and `css/`.
- Use webpack to bundle assets.
- Use the Nextcloud Vue component library (`@nextcloud/vue`) for UI components.

```javascript
// src/main.js
import Vue from 'vue'
import App from './App.vue'

new Vue({
    el: '#app',
    render: h => h(App),
})
```

## Code Signing

Apps published to the Nextcloud App Store must be code-signed. The classloader verifies signatures to ensure app integrity.

## Testing

### PHP Unit Tests

```bash
./vendor/bin/phpunit tests/
```

### JavaScript Tests

```bash
npm test
```

## Publishing to the App Store

1. Add app metadata to `appinfo/info.xml`.
2. Create a release on GitHub.
3. Submit the release URL to the [Nextcloud App Store](https://apps.nextcloud.com/).
4. Sign your app with your code signing certificate.

## References

- [Nextcloud App Development — Server 19](https://docs.nextcloud.com/server/19/developer_manual/app/index.html)
- [Nextcloud App Tutorial (GitHub)](https://github.com/nextcloud/app-tutorial)
- [Nextcloud Vue Component Library](https://github.com/nextcloud/nextcloud-vue)
- [Nextcloud App Store](https://apps.nextcloud.com/)
