---
name: api
description: Gramps Web REST API reference covering authentication (JWT), all endpoint groups (people, families, events, places, media, sources, citations, notes, tags, search, chat, users, trees), data model conventions, query parameters, and LLM/AI chat integration. Use when building integrations with the Gramps Web API, writing backend code, or querying genealogical data programmatically.
---
# Gramps Web REST API

Source: https://github.com/gramps-project/gramps-web-api

The Gramps Web API is a Python Flask REST API that exposes a Gramps genealogical database over HTTP. It is the backend for [Gramps Web](https://www.grampsweb.org/) but can also be used standalone.

All endpoints are prefixed with `/api/` (e.g. `https://your-host/api/people/`).

The interactive API documentation (Swagger/OpenAPI) is available at `/api/` on a running server.

## Authentication

The API uses **JWT (JSON Web Tokens)**.

### Obtain a Token

```http
POST /api/token/
Content-Type: application/json

{"username": "myuser", "password": "mypassword"}
```

Response:
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

The `access_token` expires in **15 minutes**. The `refresh_token` does not expire by default.

### Use the Token

Pass the access token as a Bearer header or query parameter:
```http
GET /api/people/
Authorization: Bearer eyJ...
```

Or as a query string: `GET /api/people/?jwt=eyJ...`

### Refresh the Token

```http
POST /api/token/refresh/
Authorization: Bearer <refresh_token>
```

### JWT Claims

The JWT payload contains:
- `sub` – username
- `tree` – tree ID (for multi-tree mode)
- `permissions` – object with permission flags (`canAdd`, `canEdit`, `canDelete`, `canUseChat`, …)
- `level` – numeric role level (0=viewer … 4=owner)

## Data Model

Gramps objects are serialized as JSON following the Gramps data model. Every object has:
- `handle` – unique string identifier (immutable)
- `gramps_id` – human-readable ID (e.g. `I0001` for people, `F0001` for families)
- `_class` – object class string (e.g. `"Person"`, `"Family"`, `"Event"`)
- `change` – Unix timestamp of last modification
- `private` – boolean privacy flag

### Core Object Types

| Class | Endpoint | Description |
|-------|----------|-------------|
| `Person` | `/api/people/` | Individual persons |
| `Family` | `/api/families/` | Family units (couple + children) |
| `Event` | `/api/events/` | Life events (birth, marriage, death, …) |
| `Place` | `/api/places/` | Geographic places |
| `Source` | `/api/sources/` | Source documents |
| `Citation` | `/api/citations/` | Citations linking records to sources |
| `Repository` | `/api/repositories/` | Archives, libraries, collections |
| `Media` | `/api/media/` | Media objects (photos, documents, …) |
| `Note` | `/api/notes/` | Free-text notes |
| `Tag` | `/api/tags/` | Classification tags |

### Person Object (excerpt)

```json
{
  "_class": "Person",
  "handle": "ABCDEF1234567890",
  "gramps_id": "I0001",
  "primary_name": {
    "_class": "Name",
    "first_name": "John",
    "surname_list": [
      {"_class": "Surname", "surname": "Smith", "primary": true}
    ],
    "type": {"_class": "NameType", "string": "Birth Name"}
  },
  "gender": 1,
  "birth_ref_index": 0,
  "death_ref_index": -1,
  "event_ref_list": [
    {"_class": "EventRef", "ref": "<event_handle>", "role": {"string": "Primary"}}
  ],
  "family_list": ["<family_handle>"],
  "parent_family_list": ["<parent_family_handle>"],
  "media_list": [],
  "note_list": [],
  "tag_list": [],
  "private": false
}
```

`gender`: `0`=Unknown, `1`=Male, `2`=Female

### Family Object (excerpt)

```json
{
  "_class": "Family",
  "handle": "...",
  "gramps_id": "F0001",
  "father_handle": "<person_handle>",
  "mother_handle": "<person_handle>",
  "child_ref_list": [
    {"_class": "ChildRef", "ref": "<person_handle>"}
  ],
  "relationship": {"_class": "FamilyRelType", "string": "Married"},
  "event_ref_list": []
}
```

## Endpoint Reference

### People

```http
GET    /api/people/                        # List all people
GET    /api/people/<handle>               # Get one person
POST   /api/people/                        # Create a person
PUT    /api/people/<handle>               # Replace a person
PATCH  /api/people/<handle>               # Update a person (partial)
DELETE /api/people/<handle>               # Delete a person
GET    /api/people/<handle>/timeline      # Person's life timeline
GET    /api/people/<handle>/dna/matches   # DNA matches
GET    /api/people/<handle>/ydna          # Y-DNA lineage
```

### Families

```http
GET    /api/families/
GET    /api/families/<handle>
POST   /api/families/
PUT    /api/families/<handle>
PATCH  /api/families/<handle>
DELETE /api/families/<handle>
GET    /api/families/<handle>/timeline
```

### Events, Places, Sources, Citations, Repositories, Notes, Tags, Media

All follow the same CRUD pattern:
```http
GET    /api/<type>/
GET    /api/<type>/<handle>
POST   /api/<type>/
PUT    /api/<type>/<handle>
PATCH  /api/<type>/<handle>
DELETE /api/<type>/<handle>
```

Types: `events`, `places`, `sources`, `citations`, `repositories`, `notes`, `tags`, `media`

### Media Files

```http
GET /api/media/<handle>/file                       # Download original file
GET /api/media/<handle>/thumbnail/<size>           # Get thumbnail (square pixels)
GET /api/media/<handle>/cropped/<x1>/<y1>/<x2>/<y2>  # Cropped region
GET /api/media/<handle>/ocr                        # OCR text extraction
GET /api/media/<handle>/face_detection             # Face detection
POST /api/media/archive/                           # Trigger export of all media
GET  /api/media/archive/<filename>                 # Download media archive
POST /api/media/archive/upload/zip                 # Upload media zip
```

### Bulk Operations

```http
POST /api/objects/        # Create multiple objects at once
POST /api/objects/delete/ # Delete multiple objects by handle
POST /api/transactions/   # Batch update with a single undo entry
```

### Search

```http
GET /api/search/?query=<text>&type=<object_type>&page=1&pagesize=20
POST /api/search/index/   # Trigger reindex (admin only)
```

Query parameters:
- `query` – search text (supports wildcards `*` and logical operators `AND`, `OR`, `NOT`)
- `type` – filter by object type (`Person`, `Family`, `Event`, etc.)
- `page` / `pagesize` – pagination

### Timelines

```http
GET /api/timelines/people/?handles[]=<h1>&handles[]=<h2>   # Merged timeline for multiple people
GET /api/timelines/families/?handles[]=<h1>
```

### Relations

```http
GET /api/relations/<handle1>/<handle2>        # Closest relationship
GET /api/relations/<handle1>/<handle2>/all    # All relationships
```

### AI Chat

```http
POST /api/chat/
Content-Type: application/json

{
  "messages": [
    {"role": "user", "content": "Who are the children of John Smith?"}
  ]
}
```

The response uses **server-sent events (SSE)** streaming:
```
data: {"type": "delta", "content": "John Smith "}
data: {"type": "delta", "content": "had three children..."}
data: {"type": "done"}
```

Requires the server to be configured with `LLM_BASE_URL` and `LLM_MODEL`. The user must have `canUseChat` permission.

### Reports & Exporters

```http
GET    /api/reports/                        # List available Gramps reports
GET    /api/reports/<report_id>
POST   /api/reports/<report_id>/file        # Generate a report (async task)
GET    /api/reports/<report_id>/file/processed/<filename>  # Download result

GET    /api/exporters/                      # List available export formats
POST   /api/exporters/<extension>/file      # Export database (async task)
GET    /api/exporters/<extension>/file/processed/<filename>
```

### Importers

```http
GET    /api/importers/                      # List available import formats
POST   /api/importers/<extension>/file      # Import file (e.g. .gramps XML)
```

Supported extensions: `gramps`, `gedcom`, `csv`, `vcard`, `gpkg`, etc.

### Async Tasks

Long-running operations (reports, exports, imports, indexing) return a task ID:
```json
{"task": {"id": "abc123def456"}}
```

Poll for completion:
```http
GET /api/tasks/<task_id>
```

Response:
```json
{"state": "SUCCESS", "result": {...}}
```
States: `PENDING`, `STARTED`, `SUCCESS`, `FAILURE`

### Users

```http
GET    /api/users/                          # List users (admin only)
GET    /api/users/<username>/
PUT    /api/users/<username>/               # Update user (admin or self)
DELETE /api/users/<username>/              # Delete user (admin only)
POST   /api/users/<username>/register/     # Register new user (if self-registration enabled)
POST   /api/users/<username>/create_owner/ # Create first owner account
POST   /api/users/<username>/password/change
POST   /api/users/<username>/password/reset/trigger/
```

### Trees (Multi-tree Mode)

```http
GET    /api/trees/           # List all trees
GET    /api/trees/<tree_id>
POST   /api/trees/           # Create new tree
DELETE /api/trees/<tree_id>
POST   /api/trees/<tree_id>/repair    # Check and repair DB
POST   /api/trees/<tree_id>/migrate   # Upgrade DB schema
POST   /api/trees/<tree_id>/disable   # Disable (hide) tree
POST   /api/trees/<tree_id>/enable
```

### Metadata & Config

```http
GET /api/metadata/           # Server metadata, Gramps version, features, permissions
GET /api/config/             # List all config keys (admin only)
GET /api/config/<key>/       # Get a config value (admin only)
POST /api/config/<key>/      # Set a config value (admin only)
```

## Common Query Parameters

Most list endpoints support:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `page` | Page number (1-based) | `?page=2` |
| `pagesize` | Results per page | `?pagesize=50` |
| `keys` | Comma-separated fields to include | `?keys=handle,gramps_id,name_given` |
| `handles[]` | Filter by specific handles | `?handles[]=H1&handles[]=H2` |
| `gramps_id` | Filter by Gramps ID | `?gramps_id=I0001` |
| `extended` | Include referenced objects inline | `?extended=1` |
| `profile` | Include computed profile fields | `?profile=all` |
| `sort` | Sort field | `?sort=-change` (prefix `-` for descending) |
| `filter` | Apply a saved filter by name | `?filter=MyFilter` |

The response includes an `X-Total-Count` header with the total number of matching objects.

## Error Responses

```json
{"message": "Error description"}
```

HTTP status codes:
- `400` – Bad request / validation error
- `401` – Missing or invalid token
- `403` – Insufficient permissions
- `404` – Object not found
- `409` – Conflict (e.g. handle already exists)
- `500` – Server error

## References

- [Gramps Web API Repository](https://github.com/gramps-project/gramps-web-api)
- [API Documentation (Swagger)](https://gramps-project.github.io/gramps-web-api/)
- [Backend Developer Docs](https://www.grampsweb.org/dev-backend/)
- [Gramps Data Model](https://gramps-project.org/wiki/index.php/Using_database_API)
