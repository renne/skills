---
name: reverse-proxy-expose-cli
description: NetBird `netbird expose` CLI command for ephemeral reverse proxy sessions — exposes a local port to the public internet without dashboard configuration. Supports HTTP, HTTPS, TCP, UDP, and TLS protocols. Covers enabling peer expose in account settings, command arguments and flags (--protocol, --with-external-port, --with-pin, --with-password, --with-user-groups, --with-custom-domain, --with-name-prefix), protocol-specific flag constraints (auth only for http/https; --with-external-port only for tcp/udp/tls), session lifecycle (90-second TTL, 30-second keepalive, max 10 sessions per peer), authentication options, custom domains, activity log events, and troubleshooting. Use when sharing a local development server, creating a temporary webhook endpoint, exposing a raw TCP service, or giving teammates quick access to a running service without creating a permanent dashboard service.
---
# NetBird Expose from CLI (`netbird expose`)

Source: https://docs.netbird.io/manage/reverse-proxy/expose-from-cli

## Overview

`netbird expose` lets a peer expose a local HTTP/HTTPS service to the public internet through the NetBird reverse proxy without using the dashboard. Services created this way are **ephemeral** — they are automatically removed when the command exits or the session expires.

**Requirements:**
- The NetBird client must be connected (`netbird up`).
- The **Peer Expose** feature must be enabled by an account administrator.
- If peer group restrictions are configured, the peer must belong to one of the allowed groups.

---

## Enabling Peer Expose (Administrator)

### From the Dashboard

1. Navigate to **Settings** → **Clients**.
2. Scroll to the **Peer Expose** section.
3. Toggle **Enable Peer Expose** on.
4. Optionally select specific **peer groups** that are allowed to expose services.
5. Click **Save Changes**.

### From the API

Set `peer_expose_enabled: true` in account settings. Use `peer_expose_groups` (array of group IDs) to restrict which peers can expose services.

---

## Basic Usage

```bash
# Expose a local HTTP server running on port 3000
netbird expose 3000
```

On success, the command outputs the public URL and keeps the session alive:

```
Service is now accessible at: https://a1b2c3d4e5f6.proxy.example.com
Press Ctrl+C to stop exposing.
```

Press `Ctrl+C` to stop exposing and remove the service.

---

## Arguments and Flags

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `port` | Yes | Local (internal) port number to expose (1–65535) |

### Flags

| Flag | Type | Default | Protocols | Description |
|------|------|---------|-----------|-------------|
| `--protocol` | string | `http` | all | Protocol: `http`, `https`, `tcp`, `udp`, or `tls` |
| `--with-external-port` | int | same as internal | tcp, udp, tls only | External port exposed to the public (enables port remapping). **Not available for http/https.** |
| `--with-pin` | string | — | http, https only | Protect with a 6-digit numeric PIN (must be exactly 6 digits). **Not available for tcp/udp/tls.** |
| `--with-password` | string | — | http, https only | Protect with a password (cannot be empty). **Not available for tcp/udp/tls.** |
| `--with-user-groups` | strings | — | http, https only | Restrict access to specific SSO user groups (e.g. `devops,Backend`; cannot be empty). **Not available for tcp/udp/tls.** |
| `--with-custom-domain` | string | — | http, https | Use a custom domain pre-configured in your account |
| `--with-name-prefix` | string | — | http, https | Readable prefix for the generated subdomain |

### ⚠️ Protocol-Specific Flag Constraints

The CLI enforces a hard split between **"cluster protocols"** (tcp, udp, tls) and **"web protocols"** (http, https):

| Feature | http / https | tcp / udp / tls |
|---------|-------------|-----------------|
| `--with-external-port` | ❌ rejected | ✅ supported |
| `--with-pin` | ✅ supported | ❌ rejected |
| `--with-password` | ✅ supported | ❌ rejected |
| `--with-user-groups` | ✅ supported | ❌ rejected |
| `--with-name-prefix` | ✅ supported | ❌ (n/a) |
| `--with-custom-domain` | ✅ supported | ❌ (n/a) |

Passing an incompatible flag returns a validation error and the command exits without creating a session.

---

## Authentication Options

By default, exposed services are publicly accessible. Use one or more flags to protect them:

### PIN Protection

```bash
netbird expose 3000 --with-pin 123456
```

Must be exactly 6 digits.

### Password Protection

```bash
netbird expose 3000 --with-password my-secret
```

### SSO User Group Restriction

```bash
netbird expose 3000 --with-user-groups devops,Backend
```

Users must authenticate through your configured IdP (OIDC) and belong to one of the specified groups.

### Combining Multiple Methods

```bash
netbird expose 3000 --with-password staging-pass --with-user-groups devops
```

When multiple methods are enabled, users can choose which one to use.

---

## Custom Domains

By default, `netbird expose` generates a random subdomain (e.g., `a1b2c3d4e5f6.proxy.example.com`).

### Name Prefix

Adds a readable prefix to the generated subdomain:

```bash
netbird expose 3000 --with-name-prefix my-app
# → my-app-a1b2c3.proxy.example.com
```

**Prefix rules:** lowercase letters (`a-z`), numbers (`0-9`), and hyphens (`-`); must start and end with a letter or number; 1–32 characters.

### Custom Domain

```bash
netbird expose 3000 --with-custom-domain myapp.example.com
```

The custom domain must already be configured and verified in your account. See [Custom Domains](https://docs.netbird.io/manage/reverse-proxy/custom-domains).

---

## Session Lifecycle

### How Sessions Work

1. CLI sends the request to the local NetBird daemon.
2. Daemon connects to the management server and creates an ephemeral reverse proxy service.
3. Management server provisions a TLS certificate and domain.
4. Daemon receives the public URL and displays it.
5. Daemon enters a **renewal loop**, sending a keep-alive every **30 seconds**.
6. Management server maintains an in-memory session with a **90-second TTL** that resets on each renewal.

### Session Termination

| Trigger | Service removed? |
|---------|-----------------|
| `Ctrl+C` or SIGTERM | Yes, immediately (5-second timeout for stop request) |
| Renewal failure | Yes, after 90 seconds (session TTL expires) |
| Client disconnects (killed, crash) | Yes, after 90 seconds |
| Session TTL expires | Yes, by the management server reaper |
| Management server restart | Yes, immediately (in-memory sessions lost) |

**Note:** After an unexpected client disconnect, the service remains accessible for up to 90 seconds. For sensitive services, always use authentication.

### What Causes Expiration

A session expires when the management server does not receive a renewal within 90 seconds. Common causes:

- **Network interruption** between the peer and the management server.
- **Client process killed** without graceful shutdown (e.g., `kill -9`, OOM kill, system reboot).
- **Daemon restarted** while an expose session is active.
- **Management server restarted** — all in-memory sessions are cleared immediately.

### Limits

Each peer can have a maximum of **10 active expose sessions** simultaneously.

---

## Comparison: CLI Expose vs Dashboard Services

| Property | Dashboard services | CLI expose services |
|----------|--------------------|---------------------|
| **Created via** | Dashboard or API | `netbird expose` command |
| **Lifecycle** | Permanent until manually deleted | Ephemeral — removed when command exits or session expires |
| **TTL** | None | 90 seconds, auto-renewed every 30 seconds |
| **Multiple targets** | Yes (path-based routing) | Single target (local port) |
| **Custom domains** | Yes | Yes (if pre-configured) |
| **Authentication** | SSO, Password, PIN | SSO (user groups), Password, PIN |
| **Advanced settings** | Host header, redirect rewriting | Not available |
| **Visibility** | Always shown in dashboard | Shown while active |
| **Source label** | Permanent | Ephemeral |
| **Rate limit** | None | Max 10 per peer |

---

## Activity Log Events

All peer expose actions are recorded in **Events** → **Audit**:

| Event | Description |
|-------|-------------|
| **Peer exposed service** | A peer created an expose session. Includes peer name, domain, authentication status. |
| **Peer unexposed service** | A peer stopped a session (Ctrl+C or explicit stop). |
| **Peer expose expired** | Session removed because the renewal TTL expired. |

---

## Examples

### Quick demo server

```bash
netbird expose 8080
```

### Protected staging environment

```bash
netbird expose 8080 --with-password staging-secret-2024
```

### Team-only access (SSO)

```bash
netbird expose 8080 --with-user-groups Engineering,DevOps
```

### Webhook receiver

```bash
netbird expose 8080 --with-name-prefix webhook
```

### HTTPS backend

```bash
netbird expose 8443 --protocol https
```

### Raw TCP service (e.g. PostgreSQL)

```bash
netbird expose 5432 --protocol tcp
```

### TCP service with port remapping

Expose internal port 5432 as external port 5433:

```bash
netbird expose 5432 --protocol tcp --with-external-port 5433
```

### UDP service (e.g. game server)

```bash
netbird expose 7777 --protocol udp
```

### TLS-passthrough service

```bash
netbird expose 443 --protocol tls
```

### Raw TCP service (e.g., PostgreSQL)

```bash
netbird expose 5432 --protocol tcp
```

### TCP with port remapping (internal 5432 → external 5433)

```bash
netbird expose 5432 --protocol tcp --with-external-port 5433
```

### TLS passthrough

```bash
netbird expose 8443 --protocol tls
```

---

## Troubleshooting

### "client is not running, run 'netbird up' first"

Start the NetBird daemon:

```bash
netbird up
```

### "permission denied"

Peer expose is disabled in account settings, or the peer is not in an allowed group. Ask your administrator to:

1. Enable **Peer Expose** in **Settings** → **Clients**.
2. Add the peer's group to allowed peer groups (or leave the list empty to allow all peers).

### "peer has reached the maximum number of active expose sessions (10)"

Stop an existing session before creating a new one. Stale sessions expire within 90 seconds.

### "generate service name: invalid name prefix"

The `--with-name-prefix` value is invalid. Requirements:
- Lowercase letters (`a-z`), numbers (`0-9`), and hyphens (`-`) only
- Must start and end with a letter or number
- 1–32 characters

### "auth flags ... are not supported for tcp/udp/tls protocol"

`--with-pin`, `--with-password`, and `--with-user-groups` are **only valid for http/https**. Remove these flags when using `tcp`, `udp`, or `tls` protocols.

### "--with-external-port is not supported for http/https"

`--with-external-port` is **only valid for tcp, udp, tls** (cluster protocols). For HTTP/HTTPS, the public port is determined by the proxy platform and cannot be remapped.

### Service URL returns connection error after Ctrl+C

Expected behavior — the service is removed when you press `Ctrl+C`. The domain stops resolving shortly after.

### Service still accessible after client crash

Expected behavior — the service remains active for up to 90 seconds until the session TTL expires to tolerate brief network interruptions.

---

## References

- [Expose from CLI](https://docs.netbird.io/manage/reverse-proxy/expose-from-cli)
- [Reverse Proxy Overview](https://docs.netbird.io/manage/reverse-proxy)
- [Authentication](https://docs.netbird.io/manage/reverse-proxy/authentication)
- [Custom Domains](https://docs.netbird.io/manage/reverse-proxy/custom-domains)
- [Access Logs](https://docs.netbird.io/manage/reverse-proxy/access-logs)
- [CLI Reference](https://docs.netbird.io/get-started/cli#expose)
