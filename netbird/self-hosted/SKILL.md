---
name: self-hosted
description: NetBird self-hosted deployment guide covering Docker Compose quickstart, required ports and firewall rules, configuration files (config.yaml, management.json), identity provider (IdP) integration for OIDC/SSO, reverse proxy setup with Caddy or Traefik, and upgrade procedures. Use when deploying a private NetBird server, configuring a custom identity provider, or managing a self-hosted NetBird installation.
---
# NetBird Self-Hosted Deployment

Sources: https://docs.netbird.io/selfhosted/selfhosted-quickstart and https://docs.netbird.io/selfhosted/selfhosted-guide

## Overview

Self-hosting NetBird gives you full control over all components: management server, signal server, STUN/TURN (coturn), dashboard, and optionally the identity provider. The recommended method uses Docker Compose with either the automated quickstart script or a manual configuration.

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

A typical self-hosted deployment includes these services:

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

  zitadel:
    image: ghcr.io/zitadel/zitadel:latest
    # Built-in identity provider
    ...

  management:
    image: netbirdio/management:latest
    volumes:
      - netbird-mgmt:/var/lib/netbird
    environment:
      - NB_MGMT_CONFIG=/etc/netbird/management.json

  signal:
    image: netbirdio/signal:latest

  relay:
    image: netbirdio/relay:latest

  coturn:
    image: coturn/coturn:latest
    ports:
      - "3478:3478/udp"
      - "49152-65535:49152-65535/udp"

  dashboard:
    image: netbirdio/dashboard:latest

volumes:
  traefik-acme:
  netbird-mgmt:
```

---

## Configuration Files

### management.json

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
AUTH_AUDIENCE=netbird-client
AUTH_CLIENT_ID=netbird-dashboard
AUTH_AUTHORITY=https://netbird.example.com/realms/netbird
USE_AUTH0=false
AUTH_SUPPORTED_SCOPES="openid profile email offline_access api groups"
```

---

## Identity Provider (IdP) Integration

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

Example:

```yaml
services:
  netbird-server:
    dns:
      - 86.54.11.100
      - 86.54.11.200
```

This override can remain necessary even when the host itself already has working fallback resolvers.

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

## References

- [Self-Hosting Quickstart Guide](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
- [Advanced Self-Hosted Guide](https://docs.netbird.io/selfhosted/selfhosted-guide)
- [Configuration Files Reference](https://docs.netbird.io/selfhosted/configuration-files)
- [Identity Providers](https://docs.netbird.io/selfhosted/identity-providers)
- [NetBird Releases](https://github.com/netbirdio/netbird/releases)
