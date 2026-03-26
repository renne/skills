---
name: cq
description: Install, configure, and operate mozilla-ai/cq — a shared agent knowledge commons that uses MCP tools (query, propose, confirm, flag, reflect, status) to prevent AI agents from rediscovering the same failures. Use this skill when setting up cq for Claude Code or OpenCode, deploying the team API/UI with Docker Compose, developing or extending cq, or integrating cq MCP tools into agent workflows.
allowed-tools: cq(cq-status), cq(cq-query), cq(cq-propose), cq(cq-confirm), cq(cq-flag), cq(cq-reflect)
---

# cq — Shared Agent Knowledge Commons

**cq** (*colloquy*) is an open standard for shared agent learning. AI agents query a local SQLite store (and optionally a team API) before acting, propose discoveries after novel insights, and confirm or flag knowledge units based on real-world outcomes. This prevents independent re-discovery of the same failures across agents and teams.

- **Repo:** https://github.com/mozilla-ai/cq
- **Status:** Exploratory PoC — production use at your own risk.
- **License:** Apache 2.0
- **Requires:** `uv` (always); `jq` (OpenCode install only); Docker + Docker Compose (team API)

---

## Deployed Instance — bartschnet.de

| Item | Value |
|------|-------|
| UI | `https://cq.bartschnet.de` (OIDC-protected via oauth2-proxy) |
| MCP | `https://cq-mcp.bartschnet.de/mcp` (HTTP BasicAuth via Traefik) |
| Host | `docker` (`/srv/docker/mozilla-cq/`) |
| Images | `ghcr.io/renne/cq-team-api:latest`, `ghcr.io/renne/cq-team-ui:latest` |
| DB volume | `mozilla-cq_cq-db` → `/data/team.db` inside container |
| Secrets | `/srv/docker/mozilla-cq/.env` (chmod 600) |

**Auth layers:**
- **UI** (`cq.bartschnet.de`): gated by `oauth2-proxy` → Nextcloud OIDC (`https://bartschnet.de`). Browser is redirected to Nextcloud login; any Nextcloud user can sign in. OIDC client name: `cq`. After OIDC auth, CQ auto-logs in via `X-Auth-Request-User` header (SSO bypass — no second login screen).
- **Agent MCP** (`cq-mcp.bartschnet.de`): HTTP BasicAuth at Traefik level (user `cq`). The MCP container talks to `cq-team-api` on the internal Docker network — no external API route exists.
- **`/api` path** on `cq.bartschnet.de`: forwarded to `cq-team-api` via `cq-forwardauth` middleware (oauth2-proxy validates session). Used by the UI React app for SSO token exchange.

**Traffic flow:**
```
Browser → Traefik → cq-oauth2-proxy:4180 (OIDC, injects X-Auth-Request-User)
        → cq-team-ui:3000 (React app; on mount calls /api/auth/sso-token)
        → cq-team-api:8742/auth/sso-token (reads header, issues JWT)
Browser → Traefik → /api/* (priority 10) → cq-forwardauth → cq-team-api:8742 (strip /api)
Agent   → Traefik → cq-mcp.bartschnet.de (BasicAuth) → cq-mcp:8080/mcp (FastMCP) → cq-team-api:8742 (internal, no auth)
```

**SSO auto-login flow:**
1. User hits `cq.bartschnet.de` → oauth2-proxy → Nextcloud OIDC
2. React `AuthProvider` mounts → calls `GET /api/auth/sso-token` with `isLoading=true`
3. oauth2-proxy injects `X-Auth-Request-User`; backend issues HS256 JWT
4. User authenticated → app renders; login screen never shown

**Redeploy:** `ssh docker "cd /srv/docker/mozilla-cq && docker compose pull && docker compose up -d"`
**Auto-updates:** Watchtower runs daily at 04:00 Europe/Berlin; all 3 containers have `com.centurylinklabs.watchtower.enable=true`.
**Seed users:** `ssh docker "cd /srv/docker/mozilla-cq && docker compose exec cq-team-api make seed-users USER=<name> PASS=<pass>"`
*(Note: SSO bypass makes seed users unnecessary for normal use — only needed if `CQ_PASSWORD_AUTH_ENABLED=true`.)*

---

## Architecture

cq spans three runtime boundaries:

```
Claude Code / OpenCode (agent process)
  │  SKILL.md — behavioural instructions loaded at session start
  │  hooks.json — post-error auto-query trigger
  │  /cq:status — store statistics slash command
  │  /cq:reflect — end-of-session knowledge mining slash command
  │
  ↕  stdio / MCP protocol
  │
Local MCP Server (cq_mcp — Python / FastMCP)
  │  Searches local store first, then team API
  │  Stores fallback KUs locally when team is unreachable
  │  Drains fallback KUs to team on next startup
  └─ ~/.cq/local.db  (SQLite)
      │
      ↕  HTTP / REST
      │
Team API  (Docker — Python / FastAPI — localhost:8742)
  └─ /data/team.db  (SQLite, persisted in Docker volume)

Team UI  (Docker — React / Vite — localhost:3000)
  └─ Review dashboard for team knowledge units
```

---

## Environment Variables

| Variable | Required | Default | Purpose |
|----------|----------|---------|---------|
| `CQ_LOCAL_DB_PATH` | No | `~/.cq/local.db` | Override local SQLite path |
| `CQ_TEAM_ADDR` | No | *(disabled)* | Team API URL; enables team sync. **Must be root-origin (no path component)** — e.g. `http://cq-team-api:8742` (internal Docker) or `http://localhost:8742`. See httpx quirk below. |
| `CQ_TEAM_API_KEY` | No | *(disabled)* | API key for team auth. **Removed from deployed instance** — knowledge endpoints are open on the internal Docker network (no external API route). If re-enabled, all knowledge endpoints require `Authorization: Bearer <key>`. Fork patch implements issues #63/#80. |
| `CQ_DB_PATH` | Team API only | — | Path inside the container for team DB (`/data/team.db`) |
| `CQ_JWT_SECRET` | Team API | — | JWT signing secret; must be set before starting Docker Compose |
| `CQ_FORWARD_AUTH_ENABLED` | Team API | `false` | Enable `GET /auth/sso-token` endpoint (fork patch). When `true`, backend issues JWT from `X-Auth-Request-User` header set by upstream proxy (oauth2-proxy). |
| `CQ_PASSWORD_AUTH_ENABLED` | Team API | `true` | Enable `POST /auth/login` (username/password). Set to `false` to disable the login form. |

When `CQ_TEAM_ADDR` is unset, cq runs in **local-only mode** — all knowledge stays on the agent's machine.

---

## Installation

### Claude Code (recommended)

```bash
# Via marketplace (easiest)
claude plugin marketplace add mozilla-ai/cq
claude plugin install cq

# From cloned repo
git clone https://github.com/mozilla-ai/cq.git
cd cq
make install-claude

# Uninstall
claude plugin marketplace remove mozilla-ai/cq
# or: make uninstall-claude
```

### OpenCode (MCP server)

Requires `jq` in addition to `uv`.

```bash
git clone https://github.com/mozilla-ai/cq.git
cd cq

# Global install (~/.config/opencode/)
make install-opencode

# Project-scoped install
make install-opencode PROJECT=/path/to/your/project

# Uninstall
make uninstall-opencode
make uninstall-opencode PROJECT=/path/to/your/project
```

The install script writes the MCP server entry into the appropriate `opencode.json`.

---

## Agent Configuration

### Deployed team instance (bartschnet.de)

Use `CQ_TEAM_ADDR=https://cq-api.bartschnet.de` — **not** `https://cq.bartschnet.de/api`. See httpx quirk in Known Quirks below.
Retrieve `CQ_TEAM_API_KEY` from: `ssh docker "grep CQ_TEAM_API_KEY /srv/docker/mozilla-cq/.env"`.

### Claude Code — connect to team API

Add to `~/.claude/settings.json` under `env`:

```json
{
  "env": {
    "CQ_TEAM_ADDR": "http://localhost:8742",
    "CQ_TEAM_API_KEY": "<your-api-key>"
  }
}
```

To remove team sync, delete `CQ_TEAM_ADDR` and `CQ_TEAM_API_KEY` from that file.

### OpenCode — connect to team API

Add an `environment` block to the cq MCP server entry in `~/.config/opencode/opencode.json` or `<project>/.opencode/opencode.json`:

```json
{
  "mcp": {
    "cq": {
      "type": "local",
      "command": ["uv", "run", "--directory", "/path/to/cq/plugins/cq/server", "cq-mcp-server"],
      "environment": {
        "CQ_TEAM_ADDR": "http://localhost:8742",
        "CQ_TEAM_API_KEY": "<your-api-key>"
      }
    }
  }
}
```

Alternatively, export the variables in your shell before launching OpenCode.

---

## Team API — Docker Compose

### Start services

```bash
export CQ_JWT_SECRET=<strong-secret>
make compose-up          # builds images and starts in foreground
```

Services exposed:
- Team API: `http://localhost:8742`
- Team UI (review dashboard): `http://localhost:3000`

### Manage services

| Command | Purpose |
|---------|---------|
| `make compose-down` | Stop services (data preserved) |
| `make compose-reset` | Stop and **wipe** database volume |
| `make seed-users USER=demo PASS=demo123` | Create a user account |
| `make seed-kus USER=demo PASS=demo123` | Load sample knowledge units |
| `make seed-all USER=demo PASS=demo123` | Create user + load sample KUs |

### docker-compose.yml summary

```yaml
services:
  cq-team-api:
    build: ./team-api
    ports: ["8742:8742"]
    volumes:
      - cq-data:/data            # persistent SQLite store
      - ./scripts:/app/scripts:ro
    environment:
      - CQ_DB_PATH=/data/team.db
      - CQ_JWT_SECRET=${CQ_JWT_SECRET}

  cq-team-ui:
    build: ./team-ui
    ports: ["3000:3000"]
    depends_on: [cq-team-api]

volumes:
  cq-data:
```

---

## MCP Tools Reference

| Tool | When to call | Purpose |
|------|-------------|---------|
| `query` | **Before** acting on unfamiliar territory | Search local + team stores by domain tags |
| `propose` | After discovering something non-obvious | Submit a new knowledge unit |
| `confirm` | After guidance proves correct | Boost confidence score |
| `flag` | When guidance is wrong or stale | Reduce confidence; reason: `stale`/`incorrect`/`duplicate` |
| `reflect` | End of session (`/cq:reflect`) | Mine session context for candidate KUs |
| `status` | On demand (`/cq:status`) | Show store statistics and team connectivity |

### query

```python
query(
    domain=["api", "payments", "stripe"],   # required; at least one tag
    language="python",                       # optional
    framework="fastapi",                     # optional
    limit=5,                                 # default 5, max 50
)
# Returns: { results, source ("local"/"team"/"both"), team: { status } }
```

Confidence interpretation:
- **> 0.7** — Multiple confirmations; verify before relying on it.
- **0.5–0.7** — Treat as a strong hint; verify first.
- **< 0.5** — May be stale or disputed.

Always read `insight.action` for the recommended action and `insight.detail` for full context.

### propose

```python
propose(
    summary="DynamoDB BatchWriteItem silently drops items over batch limit of 25",
    detail="BatchWriteItem returns 200 with no error if batch exceeds 25 items. Verified 2026-03 against AWS docs.",
    action="Split batches to ≤25 items. Verify current limits at the AWS BatchWriteItem docs page.",
    domain=["database", "dynamodb", "aws"],
    language="python",           # optional
    framework=None,              # optional
    pattern="data-access",       # optional
)
# Returns: { id, tier, message, team_id? } or { error }
```

**Proposal flow:**
- Team configured + reachable → goes to team only (not stored locally).
- Team configured but unreachable → falls back to local storage for retry on next startup.
- Team configured but rejects → returns error, nothing stored.
- No team configured → always stored locally.

**Writing good proposals — strip all org-specific details:**
- ✅ `"webpack 5 removes built-in Node.js polyfills — 'stream' imports fail at build time"`
- ❌ `"In the acme-corp monorepo the build fails because..."`

Include: principle > exact version, a verification method, and a timestamp with source.

### confirm / flag

```python
confirm(unit_id="ku_abc123")
flag(unit_id="ku_abc123", reason="stale")   # stale | incorrect | duplicate
```

Both check local store first, then team API. If found in both, updates both.

---

## Core Agent Protocol

Follow this loop for every task involving unfamiliar APIs, libraries, CI/CD, or infrastructure:

1. **Before acting** — call `query` with relevant domain tags.
2. **Apply guidance** — use `insight.action` as a starting point; verify before relying on it. Call `confirm` immediately if it resolves an issue.
3. **After a discovery** — call `propose` for non-obvious insights another agent would benefit from. Do this immediately after stabilising the step; do not defer.
4. **Before marking done** — review what happened:
   - Used guidance that proved correct? → `confirm`.
   - Discovered something novel? → `propose`.
   - Found guidance that was wrong? → `flag`.

### When to query

- About to make an API call to an external service.
- Working with a library or framework not used yet in this session.
- Encountered an error — query **before** retrying or attempting a fix.
- Setting up CI/CD, infrastructure, or build tooling.
- Starting work in an unfamiliar area of the codebase.

### When NOT to query

- Routine file reads/writes/edits within the current project.
- Standard library operations in the project's primary language.
- Tasks already queried for in the current session.
- Simple, well-documented operations with no known pitfalls.

### Domain tag examples

| Scenario | `domain` | `context` |
|----------|----------|-----------|
| Stripe Python integration | `["api", "payments", "stripe"]` | `language: "python"` |
| Webpack build config | `["bundler", "webpack", "configuration"]` | `framework: "react"` |
| GitHub Actions + Rust | `["ci", "github-actions", "rust"]` | `pattern: "ci-pipeline"` |
| PostgreSQL pooling | `["database", "postgresql", "connection-pooling"]` | `language: "go"` |

---

## Development Setup

### Prerequisites

- Python 3.11+
- `uv` — https://docs.astral.sh/uv/
- `pnpm` — https://pnpm.io/
- Docker + Docker Compose
- `jq` (OpenCode install only)

### Repository structure

| Directory | Component | Stack |
|-----------|-----------|-------|
| `plugins/cq/server` | MCP server (`cq_mcp`) | Python, FastMCP |
| `plugins/cq/skills/cq/SKILL.md` | Agent behavioural instructions | Markdown |
| `plugins/cq/commands/` | Slash commands (`/cq:reflect`, `/cq:status`) | Markdown |
| `plugins/cq/hooks/hooks.json` | `SessionStart` hook (runs `uv sync`) | JSON |
| `team-api` | Team knowledge API | Python, FastAPI |
| `team-ui` | Review dashboard | TypeScript, React, Vite |

### Initial setup

```bash
git clone https://github.com/mozilla-ai/cq.git
cd cq
make setup        # uv sync for server + team-api; pnpm install for team-ui
```

### Running locally (isolated, outside Docker)

```bash
# Team API only
cd team-api
CQ_DB_PATH=./dev.db CQ_JWT_SECRET=dev-secret uv run cq-team-api

# Team UI only
cd team-ui
pnpm dev
```

### Validation

| Command | Purpose |
|---------|---------|
| `make lint` | Format, lint, and type-check all components |
| `make test` | Type checks + pytest for server and team-api |
| `make format` | Auto-format Python with ruff |
| `make typecheck` | Run `uvx ty check` on Python packages + `pnpm tsc -b` |

---

## Knowledge Unit Data Model

A **knowledge unit (KU)** has:
- `id` — unique identifier (e.g. `ku_abc123`)
- `domain` — list of tag strings
- `insight.summary` — one-line description
- `insight.detail` — full explanation
- `insight.action` — concrete recommended action
- `context.languages` / `context.frameworks` / `context.pattern` — optional context filters
- `evidence.confidence` — 0.0–1.0 floating-point score (raised by `confirm`, lowered by `flag`)
- `evidence.confirmations` / `evidence.flags` — counts
- `tier` — `local` or `team`

Results are ranked by `relevance × confidence`. Relevance is computed from domain tag overlap and context match.

---

## Publishing Pre-built Images to GitHub Container Registry (ghcr.io)

The upstream `mozilla-ai/cq` repo has **no pre-built Docker images** — `make compose-up` always builds from source. To avoid building on every machine, publish images to `ghcr.io` from a fork.

### 1. Fork the repo

```bash
gh repo fork mozilla-ai/cq --clone
cd cq
```

### 2. Create `.github/workflows/docker-publish.yml`

```yaml
name: Docker Publish

on:
  push:
    branches: [main]
    tags: ["v*"]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  OWNER: ${{ github.repository_owner }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    strategy:
      matrix:
        include:
          - context: ./team-api
            image: ghcr.io/${{ github.repository_owner }}/cq-team-api
          - context: ./team-ui
            image: ghcr.io/${{ github.repository_owner }}/cq-team-ui

    steps:
      - uses: actions/checkout@v4

      - name: Log in to ghcr.io
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ matrix.image }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.context }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### 3. Push — images are published automatically

After pushing to `main`, images are available at:
- `ghcr.io/<owner>/cq-team-api:main`
- `ghcr.io/<owner>/cq-team-ui:main`

### 4. Update `docker-compose.yml` to use pre-built images

Replace `build:` directives with `image:` references:

```yaml
services:
  cq-team-api:
    image: ghcr.io/<owner>/cq-team-api:main
    ports:
      - "8742:8742"
    volumes:
      - cq-data:/data
    environment:
      - CQ_DB_PATH=/data/team.db
      - CQ_JWT_SECRET=${CQ_JWT_SECRET:?Set CQ_JWT_SECRET}

  cq-team-ui:
    image: ghcr.io/<owner>/cq-team-ui:main
    ports:
      - "3000:3000"
    depends_on:
      - cq-team-api

volumes:
  cq-data:
```

Then deploying anywhere requires only `docker-compose.yml` and `CQ_JWT_SECRET` — no source clone needed.

### 5. Make packages public (optional)

By default ghcr.io packages inherit the repo's visibility. To make images publicly pullable without authentication:
- Go to `https://github.com/<owner>/cq-team-api` (the package page)
- **Package settings → Change visibility → Public**
- Repeat for `cq-team-ui`

---

## Known Quirks and Limitations

⚠️ **httpx `base_url` + leading-slash path resolution** — `TeamClient` uses leading-slash paths (`/query`, `/propose`, etc.). httpx follows RFC 3986: a path starting with `/` is absolute and **replaces** the entire path of the base URL. `base_url="https://cq.bartschnet.de/api"` + `client.get("/query")` resolves to `https://cq.bartschnet.de/query` — **not** `/api/query`. Therefore `CQ_TEAM_ADDR` must point to a **root-origin URL with no path component**. Use `https://cq-api.bartschnet.de` (dedicated subdomain, no path) rather than `https://cq.bartschnet.de/api`.

⚠️ **Traefik: multiple routers on one container require a shared service name** — when a container defines two Traefik routers pointing to the same port, both routers MUST reference a single named service. Otherwise Traefik logs `Router X cannot be linked automatically with multiple Services` and both routers fail. Pattern:
```yaml
- "traefik.http.services.cq-api-svc.loadbalancer.server.port=8742"   # one service
- "traefik.http.routers.cq-api.service=cq-api-svc"                   # both routers reference it
- "traefik.http.routers.cq-api-agents.service=cq-api-svc"
```

⚠️ **`CQ_TEAM_API_KEY` bearer auth was removed from the deployed instance** — the `renne/cq` fork implements it (adds `get_agent_auth` to all 5 knowledge endpoints, wires through `TeamClient`), but the deployed team API at `cq-team-api:8742` runs without it. The team API is internal-only (no external Traefik route); external access is only via MCP (`cq-mcp.bartschnet.de`, BasicAuth at Traefik level). When `CQ_TEAM_API_KEY` is unset, behaviour is unchanged (backward-compatible). See `FORK_PATCHES.md` in the fork for details.

⚠️ **SSO bypass (`CQ_FORWARD_AUTH_ENABLED`) is implemented in `renne/cq` fork only** — upstream has no SSO bypass. The fork adds `GET /auth/sso-token` (reads `X-Auth-Request-User`/`X-Auth-Request-Email` from upstream proxy, issues HS256 JWT) and `GET /auth/config` (returns `{forward_auth_enabled, password_auth_enabled}`) plus React auto-login on mount. The Nextcloud OIDC provider requires `--insecure-oidc-allow-unverified-email=true` on oauth2-proxy (Nextcloud does not set `email_verified=true` in tokens). `X-Auth-Request-User` = Nextcloud `preferred_username` (e.g. `renne`).

⚠️ **Fork uses rebase sync strategy** — `renne/cq` syncs upstream with `git rebase upstream/main` + `git push --force-with-lease`. All fork-specific commits are prefixed `[fork]` for easy identification during conflict resolution. If upstream merges a fix for #63/#80, the workflow will fail loudly rather than silently diverging.

⚠️ **`reflect` is a stub in the PoC** — the `reflect` MCP tool always returns an empty candidates list. Actual session mining logic lives in the `/cq:reflect` slash command (tracked in issue #9). Use the slash command, not the tool, for end-of-session reflection.

⚠️ **Startup drain is fire-and-forget** — when `CQ_TEAM_ADDR` is configured, local fallback KUs are promoted to team at MCP server startup. If the team API is still unreachable, they stay local and retry on the next startup. Check `promoted_to_team` in `status()` output to see if drain ran.

⚠️ **`make seed-kus` runs inside the Docker container** — it calls `docker compose exec`. The team API container must already be running before you call `make seed-kus` or `make seed-all`.

⚠️ **`CQ_JWT_SECRET` must be set before `make compose-up`** — Docker Compose will error with `Set CQ_JWT_SECRET` if the variable is missing. Export it in your shell first.

⚠️ **hooks.json only triggers `uv sync` on `SessionStart`** — this ensures the MCP server's dependencies are installed but does not automatically update cq itself. Pull and reinstall manually when upgrading.

---

## HTTP MCP Endpoint — Deployed (bartschnet.de)

The `renne/cq` fork includes a **containerised HTTP MCP server** (`cq-mcp`) that wraps the team API and exposes CQ tools over FastMCP's `streamable-http` transport. This allows MCP clients that cannot run a local subprocess (VS Code GitHub Copilot, remote Copilot CLI sessions) to connect without cloning the repo.

### Endpoint

| Item | Value |
|------|-------|
| URL | `https://cq-mcp.bartschnet.de/mcp` |
| Transport | `streamable-http` (FastMCP default for HTTP) |
| Auth | HTTP BasicAuth at Traefik (user `cq`) — retrieve password from `/srv/docker/mozilla-cq/.env` |
| Image | `ghcr.io/renne/cq-mcp:main` |
| Host | `docker:/srv/docker/mozilla-cq/compose.yml` |

**Always use `:main` tag** — not `:latest`. `docker-metadata-action` only pushes `:latest` for version tags (`refs/tags/`); branch pushes produce `:main` and `:sha-<short>` only. Using `:latest` means the container never updates on branch commits.

### Health check

```bash
# Expect: 406 Not Acceptable (normal — server alive, client needs correct Accept header)
curl -si -H "Authorization: Basic Y3E6YjI1MmVjYWRjNTdjYzA1MjMyN2NiZTA3M2NhM2QwOWYwODhiMmNlMDJiODZkZjdjZTUwZjE5MGRiZjllYzUwZA==" https://cq-mcp.bartschnet.de/mcp | head -5
```

`406` = server alive, rejecting plain GET (missing `Accept: text/event-stream`). `502` = container not running. `401` = wrong credentials.

### Copilot CLI configuration (`~/.copilot/mcp-config.json`)

```json
{
  "mcpServers": {
    "cq": {
      "type": "http",
      "url": "https://cq-mcp.bartschnet.de/mcp",
      "headers": {
        "Authorization": "Basic Y3E6YjI1MmVjYWRjNTdjYzA1MjMyN2NiZTA3M2NhM2QwOWYwODhiMmNlMDJiODZkZjdjZTUwZjE5MGRiZjllYzUwZA=="
      }
    }
  }
}
```

### VS Code configuration (`~/.config/Code/User/mcp.json`)

```json
{
  "servers": {
    "cq": {
      "type": "http",
      "url": "https://cq-mcp.bartschnet.de/mcp",
      "headers": {
        "Authorization": "Basic Y3E6YjI1MmVjYWRjNTdjYzA1MjMyN2NiZTA3M2NhM2QwOWYwODhiMmNlMDJiODZkZjdjZTUwZjE5MGRiZjllYzUwZA=="
      }
    }
  }
}
```

### Architecture

```
VS Code / Copilot CLI (renne)
  └── MCP streamable-http → https://cq-mcp.bartschnet.de/mcp  (Traefik: BasicAuth)
                              ↕ FastMCP streamable-http :8080
                    docker: cq-mcp container (ghcr.io/renne/cq-mcp:main)
                              ↕ HTTP REST, no auth (internal Docker network only)
                    docker: cq-team-api:8742
```

### `cq-mcp` service (compose.yml excerpt)

```yaml
cq-mcp:
  image: ghcr.io/renne/cq-mcp:main   # :main not :latest (see tag quirk below)
  environment:
    CQ_TEAM_ADDR: http://cq-team-api:8742
    CQ_TEAM_API_KEY: ${CQ_TEAM_API_KEY}
    FASTMCP_HOST: 0.0.0.0
    FASTMCP_PORT: "8080"
    MCP_TRANSPORT: streamable-http
  labels:
    - "traefik.http.routers.cq-mcp.rule=Host(`cq-mcp.bartschnet.de`)"
    - "traefik.http.routers.cq-mcp.tls.certresolver=letsencrypt"
    - "traefik.http.services.cq-mcp-svc.loadbalancer.server.port=8080"
    - "traefik.http.routers.cq-mcp.middlewares=cq-mcp-auth"
    - "traefik.http.middlewares.cq-mcp-auth.basicauth.users=cq:$$2y$$..."
    - "com.centurylinklabs.watchtower.enable=true"
```

### FastMCP host/port env var quirk (renne/cq fork fix, commit 249f3eb)

`FastMCP.__init__` passes Python-default `host="127.0.0.1"` and `port=8000` as **explicit kwargs** to `Settings(...)`. In pydantic-settings v2, explicit constructor kwargs always override env var sources — `FASTMCP_HOST` and `FASTMCP_PORT` are silently ignored.

**Fix:** Pass env vars explicitly at the `FastMCP(...)` call site in `server.py`:

```python
mcp = FastMCP(
    "cq",
    host=os.environ.get("FASTMCP_HOST", "127.0.0.1"),
    port=int(os.environ.get("FASTMCP_PORT", "8000")),
    ...
)
```

Also note: `mcp = FastMCP(...)` is evaluated at **module import time**. Any `os.environ.setdefault()` in `main()` is too late — env vars must be set before the module is imported.

### `stateless_http=True` — preventing "Session not found" errors

FastMCP `streamable-http` transport creates a server-side session on first connect and assigns a session ID. After a **server restart**, all session IDs are lost. MCP clients (VS Code, Copilot CLI) cache session IDs and reuse them, causing every subsequent request to return:

```json
{"error": "Session not found"}
```

**Fix:** Pass `stateless_http=True` to `FastMCP()`:

```python
mcp = FastMCP(
    "cq",
    stateless_http=True,
    host=os.environ.get("FASTMCP_HOST", "127.0.0.1"),
    port=int(os.environ.get("FASTMCP_PORT", "8000")),
)
```

This disables server-side session management — each HTTP request is handled independently. Safe for cq because all state lives in SQLite (`cq-db` volume) and the team API, never in MCP session memory.

**After server restart — client reconnect:**
- **Copilot CLI:** Start a new session (stale session IDs cannot be recovered in-session)
- **VS Code:** Open Command Palette → "MCP: Restart Server" (or "MCP: Reconnect")

### MCP server health check (from inside container)

```bash
docker exec mozilla-cq-cq-mcp-1 /app/.venv/bin/python3 - <<'EOF'
import urllib.request, json
data = json.dumps({"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}).encode()
req = urllib.request.Request("http://127.0.0.1:8080/mcp", data=data, headers={"Content-Type":"application/json","Accept":"application/json, text/event-stream"}, method="POST")
r = urllib.request.urlopen(req)
print(r.status, dict(r.headers))
EOF
```

Expected: `200 OK`. `GET /mcp` → `406 Not Acceptable` is also healthy (server alive, wrong headers). `502` = container not running.

Note: Python packages are in `/app/.venv/`, NOT system Python. Use `/app/.venv/bin/python3` for all container-internal checks.

### DNS / Networking

- **Public DNS:** `cq-mcp.bartschnet.de` A record → `10.0.0.29` (VPS/Traefik) at INWX
- **CoreDNS (pve3):** regex rule `^(aio|cq|cq-api|cq-mcp|...)\.bartschnet\.de\.$` → `docker.fritz.box`
- **Netbird:** access via `cq-mcp.bartschnet.de` resolves through CoreDNS → internal IP

---

## References

- Repo: https://github.com/mozilla-ai/cq
- Architecture diagram: https://github.com/mozilla-ai/cq/blob/main/docs/architecture.md
- Full proposal (design doc): https://github.com/mozilla-ai/cq/blob/main/docs/CQ-Proposal.md
- Development guide: https://github.com/mozilla-ai/cq/blob/main/DEVELOPMENT.md
- FastMCP: https://github.com/jlowin/fastmcp
