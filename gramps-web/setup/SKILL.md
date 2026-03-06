---
name: setup
description: Gramps Web installation and setup guide covering Docker Compose deployment, configuration options, first-time owner setup, user roles, and OIDC/SSO integration. Use when deploying or configuring a Gramps Web instance, setting up users, or troubleshooting deployment issues.
---
# Gramps Web – Setup & Deployment

Source: https://www.grampsweb.org/install_setup/setup/

## What is Gramps Web?

Gramps Web is a modern genealogy web application built on top of [Gramps](https://gramps-project.org), the open-source desktop genealogy program. It provides collaborative, browser-based access to a Gramps family tree database with features including interactive charts, maps, DNA tools, a blog, and an AI chat assistant.

Architecture:
- **Frontend** (`gramps-project/gramps-web`): Lit/LitElement web components served as static files via nginx
- **Backend** (`gramps-project/gramps-web-api`): Python Flask REST API running the Gramps genealogy engine

## Docker Compose Deployment

The recommended production deployment uses Docker Compose with two containers: the API (backend) and the frontend (nginx + static files).

### Minimal `docker-compose.yml`

```yaml
services:
  gramps-api:
    image: ghcr.io/gramps-project/grampsweb:latest
    restart: always
    ports:
      - "5555:5000"
    environment:
      GRAMPSWEB_TREE: "Gramps Web"
      GRAMPSWEB_CELERY_CONFIG__broker_url: "redis://gramps-celery-broker:6379/0"
      GRAMPSWEB_CELERY_CONFIG__result_backend: "redis://gramps-celery-broker:6379/0"
      GRAMPSWEB_MEDIA_BASE_DIR: /app/media
      GRAMPSWEB_SEARCH_INDEX_DB_URI: "sqlite:////app/indexdir/search_index.db"
      GRAMPSWEB_USER_DB_URI: "sqlite:////app/users/users.sqlite"
    volumes:
      - gramps_users:/app/users
      - gramps_index:/app/indexdir
      - gramps_db:/root/.gramps/grampsdb
      - gramps_media:/app/media
      - gramps_tmp:/tmp

  gramps-celery:
    image: ghcr.io/gramps-project/grampsweb:latest
    restart: always
    depends_on:
      - gramps-celery-broker
    command: celery -A gramps_webapi.celery worker --loglevel=INFO
    environment:
      GRAMPSWEB_TREE: "Gramps Web"
      GRAMPSWEB_CELERY_CONFIG__broker_url: "redis://gramps-celery-broker:6379/0"
      GRAMPSWEB_CELERY_CONFIG__result_backend: "redis://gramps-celery-broker:6379/0"
      GRAMPSWEB_MEDIA_BASE_DIR: /app/media
      GRAMPSWEB_SEARCH_INDEX_DB_URI: "sqlite:////app/indexdir/search_index.db"
      GRAMPSWEB_USER_DB_URI: "sqlite:////app/users/users.sqlite"
    volumes:
      - gramps_users:/app/users
      - gramps_index:/app/indexdir
      - gramps_db:/root/.gramps/grampsdb
      - gramps_media:/app/media
      - gramps_tmp:/tmp

  gramps-celery-broker:
    image: redis:alpine
    restart: always

volumes:
  gramps_users:
  gramps_index:
  gramps_db:
  gramps_media:
  gramps_tmp:
```

### With a Reverse Proxy (Recommended for Production)

Add a Traefik or nginx reverse proxy container in front of `gramps-api` for TLS termination. Expose port 443 externally; the `gramps-api` service should not expose its port directly.

## Configuration

All configuration is done via **environment variables** prefixed with `GRAMPSWEB_`. Double underscores (`__`) represent nested dict keys.

| Environment Variable | Default | Description |
|---|---|---|
| `GRAMPSWEB_TREE` | *(required)* | Name of the Gramps tree to serve. Use `"*"` for multi-tree mode. |
| `GRAMPSWEB_USER_DB_URI` | *(required)* | SQLAlchemy URI for the user database (e.g. `sqlite:////app/users/users.sqlite`) |
| `GRAMPSWEB_MEDIA_BASE_DIR` | `""` | Directory where media files are stored |
| `GRAMPSWEB_SEARCH_INDEX_DB_URI` | `""` | SQLAlchemy URI for the full-text search index DB |
| `GRAMPSWEB_BASE_URL` | `http://localhost/` | Public URL of your Gramps Web instance (used for email links) |
| `GRAMPSWEB_EMAIL_HOST` | `localhost` | SMTP host for sending emails |
| `GRAMPSWEB_EMAIL_PORT` | `465` | SMTP port |
| `GRAMPSWEB_EMAIL_HOST_USER` | `""` | SMTP username |
| `GRAMPSWEB_EMAIL_HOST_PASSWORD` | `""` | SMTP password |
| `GRAMPSWEB_EMAIL_USE_TLS` | `true` | Use TLS for SMTP |
| `GRAMPSWEB_DEFAULT_FROM_EMAIL` | `""` | From address for outgoing emails |
| `GRAMPSWEB_POSTGRES_USER` | `null` | PostgreSQL user (if using PostgreSQL for the Gramps DB) |
| `GRAMPSWEB_POSTGRES_PASSWORD` | `null` | PostgreSQL password |
| `GRAMPSWEB_CELERY_CONFIG__broker_url` | `""` | Celery broker URL (Redis recommended) |
| `GRAMPSWEB_CELERY_CONFIG__result_backend` | `""` | Celery result backend URL |
| `GRAMPSWEB_LLM_BASE_URL` | `null` | Base URL for OpenAI-compatible LLM API (enables AI chat) |
| `GRAMPSWEB_LLM_MODEL` | `""` | LLM model name (e.g. `gpt-4o`, `llama3`) |
| `GRAMPSWEB_LLM_MAX_CONTEXT_LENGTH` | `50000` | Maximum token context for LLM |
| `GRAMPSWEB_VECTOR_EMBEDDING_MODEL` | `""` | Sentence Transformers model for semantic search |
| `GRAMPSWEB_REGISTRATION_DISABLED` | `false` | Disable self-registration |
| `GRAMPSWEB_IGNORE_DB_LOCK` | `false` | Ignore Gramps DB lock (useful for read-only replicas) |
| `GRAMPSWEB_NEW_DB_BACKEND` | `sqlite` | Backend for new databases (`sqlite` or `postgresql`) |
| `GRAMPSWEB_DISABLE_TELEMETRY` | `false` | Disable anonymous usage telemetry |
| `GRAMPSWEB_LOG_LEVEL` | `INFO` | Python logging level |

### Configuration File (Alternative)

Instead of environment variables, you can use a Python configuration file and point to it with:
```bash
GRAMPSWEB_CONFIG=/path/to/config.py
```

```python
# config.py
TREE = "My Family Tree"
USER_DB_URI = "sqlite:////data/users.sqlite"
MEDIA_BASE_DIR = "/data/media"
SEARCH_INDEX_DB_URI = "sqlite:////data/search_index.db"
BASE_URL = "https://myfamilytree.example.com/"
```

## First-Time Setup

After starting the containers for the first time:

1. **Create the owner account** – Navigate to `http://your-host:5555` and use the "Create owner account" form, or use the CLI:
   ```bash
   docker compose exec gramps-api python3 -m gramps_webapi user add owner_name password --role 4
   ```
   Role 4 = owner (highest privilege).

2. **Migrate user database** – Required after upgrades:
   ```bash
   docker compose exec gramps-api python3 -m gramps_webapi user migrate
   ```

3. **Import an existing Gramps database** – Via the Gramps XML import in the web UI (Admin panel → Import), or by copying your `.gramps` database directory into the `gramps_db` volume.

4. **Build the search index** – After import, rebuild the full-text search index:
   ```bash
   docker compose exec gramps-api python3 -m gramps_webapi search index-full
   ```

## User Roles

| Role | Level | Permissions |
|------|-------|-------------|
| Owner | 4 | Full access, manage users, all admin functions |
| Admin | 3 | Manage users, edit database, all operations except ownership transfer |
| Editor | 2 | Read and write (add, edit, delete records) |
| Contributor | 1 | Read + add new records; cannot edit or delete |
| Viewer | 0 | Read-only access |

Users are added via the Admin panel or the CLI:
```bash
python3 -m gramps_webapi user add <username> <password> --role <0-4> --email user@example.com
```

## OIDC / Single Sign-On

Gramps Web supports OpenID Connect (OIDC) for SSO with providers like Authentik, Keycloak, or Authelia.

```python
# config.py
OIDC_ENABLED = True
OIDC_ISSUER = "https://auth.example.com/application/o/grampsweb/"
OIDC_CLIENT_ID = "grampsweb"
OIDC_CLIENT_SECRET = "your-secret"
OIDC_REDIRECT_URI = "https://grampsweb.example.com/api/oidc/callback/"
OIDC_DISABLE_LOCAL_AUTH = False   # set True to force OIDC only
OIDC_AUTO_REDIRECT = False        # set True to redirect to OIDC login automatically
OIDC_USERNAME_CLAIM = "preferred_username"
OIDC_NAME = "My SSO Provider"     # label shown on the login button
```

## AI Chat Assistant

To enable the AI chat feature, configure an OpenAI-compatible LLM endpoint:

```yaml
environment:
  GRAMPSWEB_LLM_BASE_URL: "https://api.openai.com/v1"   # or local Ollama URL
  GRAMPSWEB_LLM_MODEL: "gpt-4o"
  GRAMPSWEB_SECRET_KEY: "your-openai-api-key"           # passed as Bearer token
```

For local models using [Ollama](https://ollama.com/):
```yaml
  GRAMPSWEB_LLM_BASE_URL: "http://ollama:11434/v1"
  GRAMPSWEB_LLM_MODEL: "llama3"
```

For semantic (vector) search, set `GRAMPSWEB_VECTOR_EMBEDDING_MODEL` to a [Sentence Transformers](https://www.sbert.net/) model name (e.g. `sentence-transformers/all-MiniLM-L6-v2`) and rebuild the index with the `--semantic` flag:
```bash
python3 -m gramps_webapi search index-full --semantic
```

## Multi-Tree Mode

Set `GRAMPSWEB_TREE=*` to enable multiple independent family tree databases served by one instance. Each tree gets its own ID. Use the tree management endpoints or the Admin UI to create and manage trees.

## CLI Reference

```bash
# Run the server
python3 -m gramps_webapi run --port 5000

# User management
python3 -m gramps_webapi user add <name> <password> [--role N] [--email E] [--tree T]
python3 -m gramps_webapi user delete <name>
python3 -m gramps_webapi user migrate

# Search index
python3 -m gramps_webapi search index-full [--semantic] [--tree T]
python3 -m gramps_webapi search index-incremental [--tree T]

# Tree management
python3 -m gramps_webapi tree list

# Database
python3 -m gramps_webapi grampsdb migrate [--tree T]
```

## References

- [Gramps Web Documentation](https://www.grampsweb.org/)
- [Setup Guide](https://www.grampsweb.org/install_setup/setup/)
- [Configuration Reference](https://www.grampsweb.org/install_setup/configuration/)
- [User Management](https://www.grampsweb.org/administration/users/)
- [OIDC/SSO Setup](https://www.grampsweb.org/administration/oidc/)
- [AI Features](https://www.grampsweb.org/administration/ai/)
- [Frontend Repository](https://github.com/gramps-project/gramps-web)
- [Backend Repository](https://github.com/gramps-project/gramps-web-api)
