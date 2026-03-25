---
name: self-hosted
description: NetBird self-hosted deployment guide covering Docker Compose quickstart, required ports and firewall rules, configuration files (config.yaml, management.json), identity provider (IdP) integration for OIDC/SSO, reverse proxy setup with Caddy or Traefik, and upgrade procedures. Use when deploying a private NetBird server, configuring a custom identity provider, or managing a self-hosted NetBird installation.
---
# NetBird Self-Hosted Deployment

Sources: https://docs.netbird.io/selfhosted/selfhosted-quickstart and https://docs.netbird.io/selfhosted/selfhosted-guide

## Overview

Self-hosting NetBird gives you full control over all components: management server, signal server, STUN/TURN (coturn), dashboard, and optionally the identity provider. The recommended method uses Docker Compose with either the automated quickstart script or a manual configuration.

### Cloud vs Self-Hosted Feature Comparison

Some features are **cloud-only** and not available in self-hosted deployments:

| Feature | Cloud | Self-Hosted |
|---------|-------|-------------|
| IdP group sync / SCIM provisioning | ✅ | ❌ |
| SIEM event streaming (Datadog, S3, Firehose, HTTP) | ✅ | ❌ |
| EDR integrations (CrowdStrike) | ✅ | ❌ |
| Peer approval workflows | ✅ | ❌ |
| MSP portal (multi-tenant management) | ✅ | ❌ |
| All networking, ACL, DNS, routing features | ✅ | ✅ |
| Reverse proxy (beta) | ✅ | ✅ |
| Activity logs | ✅ | ✅ |
| REST API | ✅ | ✅ |

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| **Linux server** | Minimum 1 vCPU, 2 GB RAM |
| **Docker** | Engine 20.10+ with Compose plugin |
| **Domain name** | Public DNS record pointing to the server IP |
| **Open ports** | See table below |

### Required Firewall Ports

| Port | Protocol | Component | Purpose |
|------|----------|-----------|---------|
| 80 | TCP | Traefik/Caddy | HTTP (ACME challenge / redirect) |
| 443 | TCP | Traefik/Caddy | HTTPS dashboard and API |
| 3478 | UDP | coturn | STUN/TURN negotiation |
| 49152–65535 | UDP | coturn | TURN relay media ports |

---

## Quickstart (Automated Script)

The quickstart script generates all configuration files and a ready-to-run `docker-compose.yml`:

```bash
curl -fsSL https://github.com/netbirdio/netbird/releases/latest/download/getting-started-with-zitadel.sh | bash
```

The script prompts for:
1. Your public domain (e.g., `netbird.example.com`)
2. The reverse proxy to use:
   - `[0]` **Traefik** (default/recommended — automatic TLS via Let's Encrypt)
   - `[2-4]` Options for using an existing proxy (nginx, Caddy, etc.)
3. Whether to enable the NetBird Proxy service (requires a separate subdomain + wildcard DNS record, e.g., `*.proxy.example.com`)

The script generates: `docker-compose.yml`, `config.yaml`, `dashboard.env`, and optionally `proxy.env`.

After completion, start the stack:

```bash
cd netbird   # directory created by the script
docker compose up -d
```

### First-time Admin Setup

After the stack starts, **no users exist by default**. Navigate to `https://netbird.example.com/setup` to create the first admin account.

The installation includes a **built-in local user management** system via an embedded [Dex](https://dexidp.io/) OIDC server — **no external identity provider is required**. You can add more users from the dashboard or switch to an external IdP later.

### Required Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 80 | TCP | HTTP (ACME challenge / redirect) |
| 443 | TCP | HTTPS dashboard and API |
| 3478 | UDP | STUN/TURN negotiation (coturn) |

The dashboard is accessible at `https://netbird.example.com`.

---

## Separate-Host Reverse Proxy Behind Existing Traefik

When the NetBird reverse-proxy runs on a different host than the management stack, the management-side reverse proxy must expose the gRPC proxy service path explicitly. With Traefik, add an HTTP router matching:

```text
Host(`netbird.example.com`) && PathPrefix(`/management.ProxyService/`)
```

and send it to the same h2c management backend used for `ManagementService`.

If an existing Traefik instance is still serving live HTTPS applications on the same `:443` entrypoint, do not add a literal `HostSNI(*)` TCP passthrough yet. Traefik evaluates TCP routers before HTTP routers, so a catch-all TCP rule will steal all HTTPS traffic. Use a dedicated coexistence domain first, for example:

- `proxy.example.com`
- `*.proxy.example.com`

and add TCP passthrough only for those names.

For wildcard DNS-01 certificates issued outside NetBird, use the reverse-proxy static certificate mode:

Custom domain verification for the NetBird reverse proxy checks the wildcard `*.<domain>`, not the apex itself. That means an apex base domain like `example.com` is still usable even though the apex cannot be a `CNAME` — the required record is:

```text
*.example.com CNAME proxy.example.com
```

This is especially useful when the public apex must remain a normal `A`/`AAAA` record (for example, behind an existing Traefik ingress) while NetBird validates and serves subdomains under the same base domain.


```text
NB_PROXY_CERTIFICATE_DIRECTORY=/certs/live/proxy-shared
NB_PROXY_CERTIFICATE_FILE=fullchain.pem
NB_PROXY_CERTIFICATE_KEY_FILE=privkey.pem
```

If the key is mounted with standard Certbot permissions (`0600` root-owned), the container may need to run as root or the mounted files need adjusted read permissions.

For day-2 operations on a self-hosted deployment, prefer the supported NetBird interfaces:

- the NetBird MCP server, if it exposes the required resource
- the NetBird management API / dashboard for resources not covered by MCP

Do **not** use direct `store.db` or other database writes for normal management changes. Database edits bypass NetBird validation and API-side invariants and should be treated as disaster-recovery work only, not routine administration.

---

## Docker Compose Structure

### v0.62+ Combined Image (Recommended)

Since **v0.62**, NetBird ships a single combined server image (`netbirdio/netbird-server`) that bundles management, signal, relay, and the embedded **Dex** OIDC identity provider. This replaces the old separate images and the Zitadel container.

**Resource requirements:** ~1 GB RAM (down from 2–4 GB with the legacy 7-container stack).

A typical v0.62+ deployment (generated by the quickstart script):

```yaml
services:
  traefik:
    image: traefik:v3.6
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik:/etc/traefik
      - traefik-acme:/acme

  netbird-server:
    image: netbirdio/netbird-server:latest
    volumes:
      - netbird_data:/var/lib/netbird
    environment:
      - NB_CONFIG=/etc/netbird/config.yaml
    # config.yaml replaces management.json + relay.env

  dashboard:
    image: netbirdio/dashboard:latest
    env_file:
      - dashboard.env

  coturn:
    image: coturn/coturn:latest
    ports:
      - "3478:3478/udp"
      - "49152-65535:49152-65535/udp"

volumes:
  traefik-acme:
  netbird_data:
```

### Legacy Stack (pre-v0.62)

The old stack used 6–7 separate containers: `management`, `signal`, `relay`, `zitadel`, `coturn`, `dashboard`, and optionally `postgres` for Zitadel. This layout is still supported but no longer recommended for new deployments.

---

## Configuration Files

### config.yaml (v0.62+)

Since v0.62, the combined `netbird-server` image uses a single **`config.yaml`** file (mounted at `/etc/netbird/config.yaml`) instead of `management.json` + `relay.env`. The quickstart script generates this file automatically. Key sections:

```yaml
server:
  listenAddress: ":80"
  exposedAddress: "https://netbird.example.com:443"

  # Shared secret used for peer (client agent) authentication — separate from user OIDC auth
  authSecret: "<random-secret>"

  auth:
    issuer: "https://netbird.example.com/oauth2"  # embedded Dex endpoint — do not change
    signKeyRefreshEnabled: true                    # management dynamically fetches JWKS from Dex
    # Redirect URIs must exactly match dashboard.env AUTH_REDIRECT_URI / AUTH_SILENT_REDIRECT_URI
    # If the dashboard moves to a different hostname, update these URIs accordingly
    dashboardRedirectURIs:
      - "https://netbird.example.com/nb-auth"
      - "https://netbird.example.com/nb-silent-auth"
    cliRedirectURIs:
      - "http://localhost:53000/"

  store:
    engine: "sqlite"                               # or "postgres" for HA deployments
    encryptionKey: "<random-32-byte-base64>"       # required — keep a secure backup
```

> **Important:** `server.auth.issuer` must always point to the **embedded Dex** endpoint (`/oauth2` path), even when federating an external IdP via a Dex connector. Direct external-OIDC configuration is not supported in the combined image.

> **Encryption key backup:** If `server.store.encryptionKey` is lost, audit-log entries will show `unknown` for user names/emails. Back this value up securely.

> **authSecret vs OIDC:** These are two independent auth mechanisms. `authSecret` authenticates peer agents (client daemons) during their gRPC management calls. OIDC JWTs authenticate human users via the dashboard. Both go through the management API on different endpoints.

---

### management.json (legacy — pre-v0.62)

> **Note:** For v0.62+ deployments using the combined `netbird-server` image, `config.yaml` supersedes `management.json`. The JSON format below applies to older split-image deployments.

The core management server configuration. Located at the path specified by `NB_MGMT_CONFIG` (default: `/etc/netbird/management.json`).

Key fields:

```json
{
  "Stuns": [
    {"Proto": "udp", "URI": "stun:netbird.example.com:3478"}
  ],
  "TURNConfig": {
    "Turns": [
      {
        "Proto": "udp",
        "URI": "turn:netbird.example.com:3478",
        "Username": "netbird",
        "Password": "your-turn-secret"
      }
    ],
    "TimeBasedCredentials": false
  },
  "Signal": {
    "Proto": "https",
    "URI": "netbird.example.com:443"
  },
  "HttpConfig": {
    "AuthIssuer": "https://netbird.example.com/realms/netbird",
    "AuthAudience": "netbird-client",
    "AuthKeysLocation": "https://netbird.example.com/realms/netbird/protocol/openid-connect/certs",
    "OIDCConfigEndpoint": "https://netbird.example.com/realms/netbird/.well-known/openid-configuration"
  },
  "IdpManagerConfig": {
    "ManagerType": "zitadel",
    "ClientConfig": {
      "Issuer": "https://netbird.example.com",
      "ClientID": "netbird-backend",
      "ClientSecret": "your-client-secret",
      "GrantType": "client_credentials"
    }
  }
}
```

### dashboard.env

Environment variables for the web dashboard:

```env
NETBIRD_MGMT_API_ENDPOINT=https://netbird.example.com:443
NETBIRD_MGMT_GRPC_API_ENDPOINT=https://netbird.example.com:443
# OIDC client config — matches embedded Dex (v0.62+)
AUTH_CLIENT_ID=netbird-dashboard
AUTH_CLIENT_SECRET=                                  # empty — dashboard is a public OIDC client
AUTH_AUDIENCE=netbird-dashboard
AUTH_AUTHORITY=https://netbird.example.com/oauth2    # embedded Dex; NOT /realms/... or /issuer
USE_AUTH0=false
AUTH_SUPPORTED_SCOPES="openid profile email groups"
AUTH_REDIRECT_URI=/nb-auth                           # must match config.yaml dashboardRedirectURIs
AUTH_SILENT_REDIRECT_URI=/nb-silent-auth             # must match config.yaml dashboardRedirectURIs
```

> **v0.62 change:** In v0.62+ with embedded Dex the auth authority path is `/oauth2`, not `/issuer` or a Zitadel/Keycloak realm path. `AUTH_AUDIENCE` matches `AUTH_CLIENT_ID` for embedded Dex (both `netbird-dashboard`).

> **Redirect URI coupling:** `AUTH_REDIRECT_URI` and `AUTH_SILENT_REDIRECT_URI` are relative paths resolved against the dashboard's origin hostname. If the dashboard container moves to a different host, these values must change **and** `config.yaml:server.auth.dashboardRedirectURIs` must be updated to match.

---

## Authentication Architecture

NetBird uses **two independent authentication mechanisms** — one for human users (via the dashboard) and one for peer agents (client daemons).

### User authentication (OIDC)

The embedded Dex OIDC server (served at `https://<domain>/oauth2`) is the identity provider for both the dashboard and the management API:

1. The browser opens the dashboard (nginx serving a Next.js app).
2. The dashboard JS initiates an OIDC authorization code flow to Dex (`/oauth2/auth`).
3. Dex authenticates the user — either directly or via a federated connector (e.g. Nextcloud, Authentik).
4. Dex issues a JWT; the browser is redirected back to the dashboard (`/nb-auth` or `/nb-silent-auth`).
5. The dashboard uses the JWT as a Bearer token in every `https://<domain>/api/...` call.
6. The management API validates the JWT signature against Dex's JWKS (`signKeyRefreshEnabled: true`).

**Key implication:** The OIDC flow always redirects through the domain where the management server and Dex live. Even if the dashboard were hosted on a different domain, the OIDC redirect URI must be registered in `config.yaml:server.auth.dashboardRedirectURIs` *for that dashboard hostname* — and Dex will redirect the browser back there after login.

### Peer authentication (authSecret)

Peer agents (the `netbird` daemon running on each device) authenticate to the management gRPC endpoint using the `server.authSecret` from `config.yaml`. This is a symmetric shared secret, not OIDC. Setup keys (used for initial enrollment) are separate and managed via the management API.

### Traefik path routing on a single hostname

All NetBird traffic on port 443 can share one hostname by using path-prefix routing with different priorities:

| Router | Priority | Paths | Backend | Protocol |
|--------|----------|-------|---------|----------|
| gRPC   | 100 | `/signalexchange.SignalExchange/`, `/management.ManagementService/`, `/management.ProxyService/` | `netbird-server:80` | h2c |
| Backend | 100 | `/relay`, `/ws-proxy/`, `/api`, `/oauth2` | `netbird-server:80` | HTTP |
| Dashboard | 1 (catch-all) | everything else | `netbird-dashboard:80` | HTTP |

The dashboard router's priority 1 means any more-specific rule (priority 100) wins, letting signal/management/relay/API traffic bypass the dashboard container entirely while all unmatched paths serve the Next.js UI.



NetBird supports any OIDC-compliant identity provider. The quickstart uses **Zitadel** (bundled). For production, you can switch to an external IdP.

### Supported IdPs

| Provider | Notes |
|----------|-------|
| Zitadel | Bundled in quickstart; open-source, self-hostable |
| Keycloak | Popular open-source IdP; configure a realm and client |
| Authentik | Modern open-source IdP |
| Okta | SaaS IdP; requires OIDC app configuration |
| Microsoft Entra ID (Azure AD) | Register an app in Azure portal |
| Google Workspace | Create an OAuth 2.0 client in Google Cloud Console |
| JumpCloud | Supports OIDC SSO |

### Configuring an External OIDC Provider

In `management.json`, update the `HttpConfig` section:

```json
"HttpConfig": {
  "AuthIssuer": "<OIDC_ISSUER_URL>",
  "AuthAudience": "<CLIENT_ID>",
  "AuthKeysLocation": "<JWKS_URI>",
  "OIDCConfigEndpoint": "<OIDC_DISCOVERY_URL>"
}
```

And update `IdpManagerConfig` with the provider's management API credentials so NetBird can sync users and groups.

See https://docs.netbird.io/selfhosted/identity-providers for provider-specific instructions.

---

## Upgrade

```bash
cd netbird
docker compose pull
docker compose up -d --remove-orphans
```

Always check the [release notes](https://github.com/netbirdio/netbird/releases) for breaking changes before upgrading.

---

## Connecting Clients to a Self-Hosted Server

When running `netbird up` on client devices, pass the management URL:

```bash
netbird up --setup-key <SETUP_KEY> --management-url https://netbird.example.com:443
```

Or set it in the client configuration file (`/etc/netbird/config.json`):

```json
{
  "ManagementURL": "https://netbird.example.com:443"
}
```

---

## Troubleshooting

| Problem | Check |
|---------|-------|
| Dashboard unreachable | Traefik/Caddy logs, DNS resolution, port 443 open |
| Peers not connecting | coturn logs, UDP 3478 and relay ports open |
| Authentication failing | IdP logs, `AuthIssuer` and `AuthAudience` values in `management.json` |
| Peers not appearing in dashboard | Management container logs (`docker compose logs management`) |

### ⚠️ env_file changes don't take effect (stale container env vars)

**Symptom:** You updated an `env_file` (e.g., `dashboard.env`) on disk, ran `docker compose up -d`, and the container still uses the old values. `docker inspect` shows stale vars.

**Root cause:** Docker bakes environment variables **at container creation time**. If the container already exists and the image hasn't changed, `docker compose up -d` does NOT recreate it — it leaves the existing container running with its original env vars.

**Fix:** Force recreation:
```bash
docker compose up -d --force-recreate dashboard
```

Verify the new values are live:
```bash
docker inspect netbird-dashboard | grep -A5 AUTH_AUTHORITY
```

This is especially important after changing `AUTH_AUTHORITY`, `AUTH_CLIENT_ID`, or `USE_AUTH0` in `dashboard.env`.

### ⚠️ Volume name: `netbird_netbird_data` (double project name)

When the Docker Compose project is named `netbird` (default from directory name) and the volume is defined as `netbird_data:` in the YAML, Docker names the actual volume `netbird_netbird_data`. The host path is:

```
/var/lib/docker/volumes/netbird_netbird_data/_data/
```

This contains `idp.db` (Dex embedded IdP database) and `store.db` (NetBird management). Use this path for direct sqlite3 access.

### ⚠️ Dashboard redirects to the wrong IdP (stale OIDC config in browser)

**Symptom:** Dashboard redirects to an old OIDC URL (e.g., `https://bartschnet.de/apps/oidc/authorize`) even after updating `dashboard.env`. Occurs even in private/incognito windows after cache clearing.

**Root cause:** Dashboard container was still using stale env vars baked in at creation time (see above). The browser is correctly following the JS config served by the container.

**Fix:** `docker compose up -d --force-recreate dashboard`, then hard-refresh (`Ctrl+Shift+R`).

### ⚠️ Dashboard returns 404 until Ctrl+Shift+R (hard refresh)

**Symptom:** Opening `https://netbird.example.com/` in a browser shows HTTP 404. After a hard refresh (Ctrl+Shift+R / Cmd+Shift+R), the dashboard loads normally. Happens repeatedly across sessions.

**Root cause:** The NetBird dashboard container serves the Next.js frontend via an internal **nginx**. nginx returns 404 responses **without a `Cache-Control` header**, while 200 responses carry `no-store, no-cache, must-revalidate`. When the stack briefly goes down (e.g. Watchtower auto-update, `docker compose up -d`), nginx returns a 404. The browser caches that 404 (heuristic caching based on the `etag` header). On every subsequent visit the browser serves the cached 404 — until a hard refresh, which sends `Cache-Control: no-cache` in the request and bypasses the browser cache.

**Important:** Ctrl+Shift+R (hard refresh) bypasses the cache for that single load but does **not delete** the cached 404 from the browser's cache store. After the hard refresh returns 200 (which itself is `no-store`, so not stored), the next normal navigation hits the browser cache again and finds the old 404 entry — showing 404 indefinitely. The service itself may be running fine the entire time.

**Immediate workaround** (clears the stuck 404 from the browser cache):
- Chrome/Edge: DevTools → Application → Storage → **Clear site data**
- Or: browser Settings → Privacy → Clear browsing data → filter to the NetBird domain

**True root cause (nginx):** The dashboard container's internal nginx config (`docker/default.conf`) uses `add_header Cache-Control "no-store, ..." ` **without the `always` keyword** in both `location /` and `location = /404.html`. Per nginx semantics, `add_header` without `always` only applies to 2xx/3xx responses — headers are silently dropped on 4xx/5xx. This is an upstream bug tracked in **[netbirdio/dashboard PR #549](https://github.com/netbirdio/dashboard/pull/549)** ("Add CSP headers to nginx configuration"), which already contains the fix (`always` added to both `Cache-Control` lines). Once PR #549 is merged and Watchtower pulls the new image, the bug is resolved automatically.

**Correct nginx fix (from PR #549):**
```nginx
add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0" always;
```

**Traefik workaround** (while waiting for upstream PR to merge): Add a Traefik headers middleware to force `Cache-Control: no-store` on all responses from the dashboard router. This prevents the browser from ever caching a transient 404.

> ✅ **Applied on vps1** (`/srv/docker/netbird/docker-compose.yml`) as of 2026-03-25. Verified: both HTTP 200 and 404 responses now include `Cache-Control: no-store, no-cache, must-revalidate`. This workaround will remain in place until upstream PR #549 is merged and Watchtower auto-updates the image.

Via Docker Compose labels on the `dashboard` service:

```yaml
labels:
  - "traefik.http.middlewares.netbird-nocache.headers.customResponseHeaders.Cache-Control=no-store, no-cache, must-revalidate"
  - "traefik.http.routers.netbird-dashboard.middlewares=netbird-nocache"  # append to existing list
```

Or via a Traefik dynamic config file:

```yaml
http:
  middlewares:
    netbird-nocache:
      headers:
        customResponseHeaders:
          Cache-Control: "no-store, no-cache, must-revalidate"
```

> ⚠️ Traefik's `customResponseHeaders` **overwrites** any existing `Cache-Control` header from the upstream. Verify after applying that the 200 response still has the expected value.

```bash
# Tail logs for all services
docker compose logs -f

# Restart a specific service
docker compose restart management
```

### DNS Bootstrap on the Self-Hosted Server

If the self-hosted server also runs a NetBird client, separate the host DNS path from the container DNS path during troubleshooting:

- the host may use a NetBird-managed `/etc/resolv.conf`
- the management container may use Docker's embedded resolver `127.0.0.11`
- startup tasks may still require public DNS before the whole NetBird path is stable

Recommended pattern for infrastructure hosts:

1. keep the local NetBird resolver first on the host
2. add public fallback resolvers after it on the host
3. if the management container still fails through `127.0.0.11`, add a per-service `dns:` override in `docker-compose.yml`

Example (CoreDNS first with ISP fallbacks):

```yaml
services:
  netbird-server:
    dns:
      - 100.115.218.142   # CoreDNS on NetBird overlay (preferred)
      - 86.54.11.100      # ISP resolver fallback
      - 86.54.11.200      # ISP resolver fallback
```

This override can remain necessary even when the host itself already has working fallback resolvers.

#### IPv6 / split-brain DNS fix: `extra_hosts`

If a service (e.g., Nextcloud, an IdP) is reachable on the internal LAN at an IPv4 address but public DNS returns an IPv6 AAAA record, TLS inside the container will fail (the certificate won't match the LAN IP, or the IPv6 address is unreachable).

**Fix:** Use `extra_hosts` to hard-code the IPv4 address. Docker writes these entries to `/etc/hosts` inside the container, which takes precedence over DNS entirely:

```yaml
services:
  netbird-server:
    extra_hosts:
      - "nextcloud.example.de:10.0.0.29"   # force IPv4; bypass public DNS
```

This is the most reliable fix and is not affected by DNS bootstrap timing or resolver ordering.

### Symptom: geolocation startup crash

One concrete symptom of broken container-side DNS is:

```text
lookup pkgs.netbird.io on 127.0.0.11:53: server misbehaving
```

Validate these paths separately:

- host DNS resolution
- container namespace DNS resolution
- Docker embedded resolver behavior
- provider firewall handling of UDP/53

---

## Adding an External OIDC IdP via Dex Connector Federation

The combined server image (`netbirdio/netbird-server`) embeds Dex and **cannot** use an external OIDC IdP directly. The correct architecture is to federate Dex to the external IdP using a Dex OIDC connector:

```
Dashboard → Embedded Dex (https://netbird.example.de/oauth2)
                  ↕  OIDC connector
            External IdP (e.g., Nextcloud, Authentik, Keycloak)
```

The dashboard and management server always talk to embedded Dex — do not change `config.yaml` `server.auth.issuer` or `dashboard.env` AUTH_* vars.

### Supported connector types

Named types: `zitadel`, `entra`, `okta`, `pocketid`, `authentik`, `keycloak`, `google`, `microsoft`  
Generic: `oidc` (use for any OIDC-compliant provider, including Nextcloud)

### Inserting a connector directly into `idp.db`

NetBird management API and dashboard have no UI for Dex connectors. Insert directly:

```bash
DB=/var/lib/docker/volumes/netbird_netbird_data/_data/idp.db

# Connector config (JSON)
CONFIG='{"issuer":"https://idp.example.de","clientID":"<client-id>","clientSecret":"<client-secret>","redirectURI":"https://netbird.example.de/oauth2/callback","scopes":["openid","profile","email"],"insecureEnableGroups":true,"insecureSkipEmailVerified":true}'

sqlite3 "$DB" "INSERT INTO connector (id, type, name, resource_version, config) VALUES ('myidp', 'oidc', 'My IdP', '', '$CONFIG');"

# Verify
sqlite3 "$DB" "SELECT id, type, name FROM connector;"
```

Then restart all NetBird containers:
```bash
cd /srv/docker/netbird && docker compose restart
```

### Connector table schema

```
connector(id TEXT PK, type TEXT, name TEXT, resource_version TEXT, config BLOB)
```

`store.db` (NetBird management) has **no** `identity_providers` table and never overwrites Dex connectors — connectors persist across restarts.

### Keeping local login enabled

Set `server.auth.localAuthDisabled: false` (default) in `config.yaml`. The "Email" local connector in `idp.db` handles this — do not remove it.

### Nextcloud-specific notes

- Nextcloud tokens never include `email_verified: true` — `insecureSkipEmailVerified: true` is **required** in the connector config
- Client ID must be alphanumeric only (A-Za-z0-9), 32–64 characters
- Create a **confidential** client (not public) so Dex can use a `client_secret`
- See the Nextcloud OIDC IdP skill for the correct `occ oidc:create` syntax

### ⚠️ Fundamental Circular Dependency: IdP Behind the Overlay Network

**The problem:** If your external IdP (e.g., Nextcloud) is only reachable *through* the NetBird overlay (e.g., on a Docker host accessible via a subnet router peer), management cannot start:

1. Management/Dex tries to fetch `/.well-known/openid-configuration` from the IdP at startup
2. The NetBird overlay is not yet up (no peers registered yet)
3. Management/Dex fails to reach the IdP → startup crash or boot loop

**This is a known open issue:** [netbirdio/netbird#5084](https://github.com/netbirdio/netbird/issues/5084) (filed Jan 2026, `triage-needed`, no upstream fix yet). A `--skip-oidc-startup-check` flag was requested but not implemented as of early 2026.

**Removing the `HttpConfig` block to bypass the check causes a nil pointer panic:**
```
panic: runtime error: invalid memory address or nil pointer dereference
  github.com/netbirdio/netbird/management/cmd.loadMgmtConfig
```
Management requires a valid, reachable IdP config at startup — there is no way around this without a code change.

> **⚠️ v0.62+ config incompatibility (from NetBird team, issue #5084):** `HttpConfig` is part of the **legacy pre-v0.62 IdP configuration** and must **not** be combined with `EmbeddedIdP` config in v0.62+. Mixing them causes unexpected behavior. If you see the nil pointer panic above, first check whether you're running v0.62+ — if so, the fix is to remove `HttpConfig` entirely and use only the `EmbeddedIdP` block. The panic in v0.61.x and earlier occurs because `HttpConfig` was required; in v0.62+ it should be absent when using embedded Dex.

**Workaround options (no perfect solution):**

| Approach | Notes |
|---|---|
| **Use embedded Dex with local auth** | ✅ Recommended. No circular dep. Simplest. |
| **IdP has a public endpoint** | ✅ Works if IdP is internet-reachable (e.g., Nextcloud is already public). Management reaches it via internet, not overlay. Watch the `extra_hosts` routing — see below. |
| **Standalone external Dex** | Dex in a separate container starts before management; Dex federates to your IdP. Breaks circular dep, but the NetBird dashboard hides the "Identity Providers" UI tab (only shown for EmbeddedIdP). |
| **Two-phase startup script** | Documented below. Fragile, abandoned — keep as historical reference only. |

**Key insight for publicly-reachable IdPs:** If the IdP is accessible via the internet (e.g., `https://bartschnet.de`), the management server on vps1 can reach it directly without going through the overlay. The circular dependency only exists if the IdP hostname resolves to an IP that is **only reachable via the overlay**. If `extra_hosts` or `/etc/hosts` routes the IdP hostname to the overlay IP (e.g., `10.0.0.29`), that overrides the public route — remove such entries from the `netbird-server` service when using a publicly-reachable IdP.

### ⚠️ Dex connector startup race condition (WireGuard route deadlock)

> **Historical note:** The two-phase startup script below was an attempt to work around the circular dependency above. It was ultimately abandoned because the problem is architectural, not a timing issue. **For new setups, use embedded Dex with local auth or an IdP with a public endpoint.**

**Symptom:** After recreating the `netbird-server` container, Dex logs show:

```
ERRO failed to create connector nextcloud: dial tcp 10.0.0.29:443: connect: no route to host
```

The Nextcloud login button disappears. The connector never recovers — Dex does **not** retry failed connector initialization.

**Root cause:**

1. NetBird policy routing table 7120 contains routes to internal subnets (e.g., `10.0.0.0/24`) advertised by subnet router peers.
2. Table 7120 is populated by the NetBird daemon **after** it re-establishes peer connections — which only happens after the management server is running and healthy.
3. When the `netbird-server` container is recreated, the management server briefly goes offline. The daemon loses peers. Routes are re-added only after the daemon reconnects to management.
4. Dex starts with the management server process, tries to initialize the OIDC connector immediately, and fails because the WireGuard route isn't in table 7120 yet.

**Wrong fix — waiting loop before starting server (creates a deadlock):**

A `/dev/tcp` loop checking TCP connectivity to the IdP before starting the server deadlocks:
- Loop blocks management server start
- Daemon can't reconnect (no management server)
- Routes never populate
- TCP never works
- Loop never exits

**Two-phase startup script (historical — abandoned):**

1. Start management server in **background** → daemon reconnects → WireGuard routes populate
2. Wait for TCP to the IdP host (proves route is live)
3. Gracefully stop background server
4. `exec` server again → Dex initializes connector with route available

```bash
#!/usr/bin/bash
# Phase 1: start server in background so WireGuard routes establish
/go/bin/netbird-server --config /etc/netbird/config.yaml &
BG_PID=$!
echo "[start-server] Phase 1: server running in background (PID=$BG_PID)"
echo "[start-server] Waiting for TCP connectivity to IdP at 10.0.0.29:443..."
until (echo > /dev/tcp/10.0.0.29/443) 2>/dev/null; do
  echo "[start-server] Route not ready yet, sleeping 2s..."
  sleep 2
done
echo "[start-server] Route ready. Stopping background server..."
python3 -c "import os,signal; os.kill($BG_PID, signal.SIGTERM)"
sleep 3
echo "[start-server] Phase 2: restarting server so Dex can initialize connectors"
exec /go/bin/netbird-server --config /etc/netbird/config.yaml
```

**Why Python for the stop signal:** The `kill` bash builtin is blocked in some environments. `python3 -c "import os,signal; os.kill($BG_PID, signal.SIGTERM)"` is safe — bash expands `$BG_PID` to an integer before Python sees it.

**`/dev/tcp` requires bash:** `/dev/tcp` is a bash builtin, not available in `sh`. The shebang and entrypoint must use `bash`.

**Prerequisites in container:** The `netbirdio/netbird-server` image is Ubuntu 24.04 LTS. `/usr/bin/bash` and `python3` are available.

---

## Store Engine / Database Backend

NetBird management supports three database backends (via GORM):

| Engine | Notes |
|--------|-------|
| `sqlite` | Default. File stored in `Datadir` as `store.db`. |
| `postgres` | Requires external PostgreSQL instance. |
| `mysql` | Requires external MySQL/MariaDB instance. |

**Configuration (v0.62+ `config.yaml`):**

```yaml
StoreConfig:
  Engine: postgres
```

**Required env vars for PostgreSQL:**

```bash
NB_STORE_ENGINE_POSTGRES_DSN="host=db user=netbird password=secret dbname=netbird sslmode=disable"
# Legacy alias (still accepted):
NETBIRD_STORE_ENGINE_POSTGRES_DSN="..."
```

**Required env vars for MySQL:**

```bash
NB_STORE_ENGINE_MYSQL_DSN="netbird:secret@tcp(db:3306)/netbird?charset=utf8mb4&parseTime=True&loc=Local"
# Legacy alias:
NETBIRD_STORE_ENGINE_MYSQL_DSN="..."
```

> **Important:** The quickstart script (`getting-started.sh`) does **not** prompt for or configure the store engine. It generates a `config.yaml`/`management.json` with **no `StoreConfig` field**, which defaults to SQLite. PostgreSQL/MySQL must be added manually post-install.

> **Common confusion:** In the legacy `getting-started-with-zitadel.sh` script, a PostgreSQL container is provisioned — but it is **exclusively for Zitadel (the IdP)**. NetBird management data in that script still goes to SQLite. This can be switched to CockroachDB for the Zitadel database via `export ZITADEL_DATABASE=cockroach` before running the script, but this has no effect on management data.

### Manual steps to switch to PostgreSQL (post-install)

1. Provision a PostgreSQL instance (external or as an additional Docker Compose service).
2. Add the `StoreConfig` block to `config.yaml`:

   ```yaml
   StoreConfig:
     Engine: postgres
   ```

3. Set the DSN env var in `docker-compose.yml` for the `management` service:

   ```yaml
   environment:
     NB_STORE_ENGINE_POSTGRES_DSN: "host=db user=netbird password=secret dbname=netbird sslmode=disable"
   ```

4. Restart the management container: `docker compose up -d management`

### Database HA Options

> **⚠️ Important caveat:** Database HA makes the *data store* redundant. The **NetBird management server process
> is still single-instance** — if it crashes, there is an application-level outage regardless of DB HA. Full
> management-server HA (shared DB + LB + multiple management replicas) is **not officially supported**.

#### Recommended: PostgreSQL + Patroni

PostgreSQL is the preferred choice for NetBird HA because its single-primary model matches NetBird's
single-instance management server perfectly. No write-conflict risk, excellent tooling, and the PostgreSQL
License (permissive).

**Architecture:**

```
NetBird Management ──► PgBouncer / HAProxy (stable VIP)
                              │
              ┌───────────────┴───────────────┐
         [Patroni + PG primary]   [Patroni + PG replica(s)]
                    │  ↕ leader election  │
                 [etcd cluster (3 nodes, odd quorum)]
```

| Component | Role |
|---|---|
| **PostgreSQL streaming replication** | Built-in WAL-based replication from primary to replica(s). Provides data redundancy and read scaling. Does **not** auto-failover by itself. |
| **Patroni** | Python daemon on each PostgreSQL node. Uses etcd/Consul/ZooKeeper for distributed leader election. Detects primary failure and auto-promotes a replica. RTO ~10–30 seconds. Battle-tested on k8s and bare-metal. |
| **etcd** (3-node cluster) | Patroni's distributed lock store for leader election. Must have odd quorum (3, 5, …). |
| **PgBouncer / HAProxy** | Provides a stable connection endpoint. `NB_STORE_ENGINE_POSTGRES_DSN` points here — never changes regardless of which node is primary. PgBouncer adds connection pooling; HAProxy works purely at L4. |

**DSN configuration:**

```bash
# Points at PgBouncer/HAProxy VIP, not a specific PostgreSQL node
NB_STORE_ENGINE_POSTGRES_DSN="host=pg-vip.internal user=netbird password=secret dbname=netbird sslmode=require"
```

**Kubernetes option:** [CloudNativePG](https://cloudnative-pg.io/) (CNCF sandbox) bundles Patroni, manages PVC
storage, and handles failover automatically — the simplest path for k8s deployments.

**Alternative orchestrators (less common):**
- **repmgr**: lighter-weight, PostgreSQL-native, less automated. Good for simple two-node setups.
- **pg_auto_failover**: dedicated monitor + two PostgreSQL nodes; simpler than Patroni for minimal clusters.

#### MySQL InnoDB Cluster (if MySQL ecosystem preferred)

MySQL InnoDB Cluster = Group Replication (built into MySQL 8.0+) + MySQL Router + MySQL Shell.

| Component | Role |
|---|---|
| **Group Replication** | Consensus-based synchronous replication; single-primary by default (correct for NetBird). |
| **MySQL Router** | Monitors the cluster and auto-reroutes connections on failover. `NB_STORE_ENGINE_MYSQL_DSN` points at the Router endpoint. |
| **MySQL Shell** | Admin/bootstrapping tool for the cluster. |

```bash
# Points at MySQL Router, not a specific cluster node
NB_STORE_ENGINE_MYSQL_DSN="netbird:secret@tcp(mysql-router.internal:6446)/netbird?charset=utf8mb4&parseTime=True&loc=Local"
```

#### MariaDB Galera Cluster (avoid for NetBird)

Galera is a synchronous multi-master cluster. All nodes accept writes simultaneously. While Galera is production-proven, it is a **poor fit for NetBird** because:

1. **Multi-master overhead for a single writer.** NetBird management is always single-instance — Galera's
   write-set certification runs on every node for every write, adding latency with zero benefit.
2. **Write certification conflicts.** Galera can reject (rollback) transactions when concurrent writes conflict.
   Although rare for NetBird's serialised workload, it is an unnecessary risk that GORM and NetBird do **not**
   handle with automatic retries.
3. **No built-in client router.** Galera requires HAProxy or ProxySQL as an external routing layer (unlike
   MySQL Router, which ships with InnoDB Cluster).
4. **MaxScale license.** MariaDB's most capable proxy (MaxScale) switched to the Business Source License (BSL) in
   2023 — commercial use at scale may require a paid agreement.

If you already operate MariaDB and want HA, consider standard **MariaDB semi-sync replication + MaxScale** (single-primary, similar to Patroni) rather than Galera.

#### Summary comparison

| | PostgreSQL + Patroni | MySQL InnoDB Cluster | MariaDB Galera |
|---|---|---|---|
| **Replication model** | Single-primary (WAL streaming) | Single-primary (Group Replication) | Multi-master |
| **Auto-failover** | ✅ Patroni ~10–30 s | ✅ MySQL Router ~seconds | ⚠️ Needs HAProxy/ProxySQL |
| **Write-conflict risk** | ✅ None | ✅ None | ⚠️ Certification failures possible |
| **NetBird fit** | ✅ Excellent | ✅ Good | ⚠️ Suboptimal |
| **Client routing** | PgBouncer / HAProxy VIP | MySQL Router endpoint | HAProxy / ProxySQL |
| **Kubernetes operator** | CloudNativePG (CNCF) | MySQL Operator | Percona PXC Operator |
| **License** | PostgreSQL (permissive) | GPL-2.0 | GPL-2.0 + BSL proxy |
| **Recommendation** | ✅ **Preferred** | ✅ If MySQL ecosystem | ❌ Avoid |

---

## High Availability (HA)

### Management server

**HA clustering for the management server is NOT officially supported or documented by NetBird,
but is achievable in practice because management is stateless (all state in PostgreSQL).**

- Both quickstart scripts deploy a **single management container** with no clustering options.
- However: running multiple management instances against the same PostgreSQL database works —
  there is no coordination needed between them (each instance is read/write to the DB directly).
- Full HA reduces to: **making the database fault-tolerant**.

#### 3-VPS Active-Active Architecture (community pattern)

Full architecture documented at `~/.copilot/networks/netbird/ha-cluster/SKILL.md`.

**Infrastructure:**

| Node | Provider | AS | RAM | Notes |
|---|---|---|---|---|
| netbird-1 | Netcup | AS197540 | 1 GB | Existing node |
| netbird-2 | IONOS | AS8560 | 1 GB | Second node |
| netbird-3 | Hetzner | AS24940 | 2 GB | Third node (recommended) |

**Critical constraint — circular dependency:**
NetBird **cannot** be used to network its own cluster nodes. Use a static WireGuard mesh instead:
```
NetBird needs management → management needs DB → DB needs inter-VPS network → breaks if NetBird is down
```

**Inter-VPS network:** Static WireGuard mesh at `10.0.0.1/2/3` on all 3 nodes.

**Database HA:** PostgreSQL + Patroni + etcd (3-node Raft) + HAProxy-on-WireGuard:
- Each node runs HAProxy; polls Patroni REST (`GET /primary`) to detect current primary
- NetBird management DSN points to local HAProxy (`host=10.0.0.x`) → transparently forwarded to primary
- Failover RTO: ~25–40 s; no DNS API scripting required
- PostgreSQL tuning for 1 GB nodes: `shared_buffers=64MB`, `max_connections=20`, `wal_buffers=4MB`

**Dex (IdP):** Sticky DNS — `idp.nb.example.de` → single A record for OAuth state consistency;
`insecureSkipEmailVerified: true` required for Nextcloud OIDC.

**Auto-updates:** Watchtower with `WATCHTOWER_LABEL_ENABLE=true`; PostgreSQL and Patroni
containers labelled `watchtower.enable=false` (major PG upgrades require manual `pg_upgrade`).

**Migration from SQLite:** NetBird auto-migrates on first run with `NB_STORE_ENGINE=postgres`.
Migration plan: `~/.copilot/networks/netbird/ha-cluster/migration-plan.md`.

### Relay server (TURN)

Relay servers **are** horizontally scalable:

- Deploy multiple `relay` containers, all using the **same relay token** (shared credentials).
- NetBird peers auto-select among available relays via the signal server.
- This is the recommended way to scale relay capacity.

### Reverse proxy / ingress

Multiple reverse-proxy instances can front the same management/signal/relay backends using standard load-balancing (DNS round-robin, hardware/software LB, etc.). NetBird has no built-in awareness of proxy replicas.

### Signal server

The signal server is also stateless and can be replicated, though this is not officially documented.

---

## References

- [Self-Hosting Quickstart Guide](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
- [Advanced Self-Hosted Guide](https://docs.netbird.io/selfhosted/selfhosted-guide)
- [Configuration Files Reference](https://docs.netbird.io/selfhosted/configuration-files)
- [Identity Providers](https://docs.netbird.io/selfhosted/identity-providers)
- [NetBird Releases](https://github.com/netbirdio/netbird/releases)
- [Issue #5084 — IdP behind reverse proxy / overlay: startup deadlock, request for `--skip-oidc-startup-check`](https://github.com/netbirdio/netbird/issues/5084)
