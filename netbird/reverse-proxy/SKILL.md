---
name: reverse-proxy
description: NetBird Reverse Proxy feature for exposing internal services to the public internet without opening firewall ports. Covers service creation, targets (peer/host/domain/subnet), domains (free/cluster/custom), authentication (SSO/password/PIN), TLS certificate modes (ACME/static), high availability, path-based routing, backend service configuration (trusted proxies for Jellyfin, Home Assistant, Nextcloud), access logs, and service statuses. Use when setting up public HTTPS access to internal NetBird peers or network resources, configuring authentication for exposed services, managing custom domains, or troubleshooting reverse proxy service provisioning.
---
# NetBird Reverse Proxy

Sources:
- https://docs.netbird.io/manage/reverse-proxy
- https://docs.netbird.io/manage/reverse-proxy/authentication
- https://docs.netbird.io/manage/reverse-proxy/custom-domains
- https://docs.netbird.io/manage/reverse-proxy/access-logs
- https://docs.netbird.io/manage/reverse-proxy/service-configuration

## Overview

NetBird Reverse Proxy exposes internal services running on peers or behind network resources to the public internet. NetBird handles TLS termination, optional authentication, and proxies traffic through the WireGuard mesh — without opening ports or configuring firewalls on internal machines.

**Availability:**
- **Self-hosted** deployments: available (beta). Requires [Traefik](https://docs.netbird.io/selfhosted/external-reverse-proxy) as the external reverse proxy (TLS passthrough is mandatory).
- **Cloud** deployments: coming soon.

**Limitations:** Does not currently support pre-shared keys or Rosenpass.

---

## Core Concepts

### Services

A service maps a public domain to one or more internal targets. Each service has:

| Component | Description |
|-----------|-------------|
| **Domain** | Public URL where the service is reachable |
| **Targets** | One or more backend destinations |
| **Authentication** | Optional SSO, password, or PIN protection |
| **Settings** | Host header forwarding and redirect rewriting |
| **Enabled/Disabled** | Toggle traffic without deleting the service |

### Targets

A target defines where proxied traffic is sent within the NetBird network. Each service can have multiple targets for path-based routing.

| Type | Description | How to select |
|------|-------------|---------------|
| **Peer** | Machine running the NetBird agent | Select from peer list |
| **Host** | Network resource by IP address | Select from network resources |
| **Domain** | Network resource by domain name | Select from network resources |
| **Subnet** | Network resource within a CIDR range | Select resource, then specify IP within range |

Each target also has:
- **Path** (optional) — URL path prefix for path-based routing (e.g., `/api`)
- **Protocol** — `HTTP` or `HTTPS`
- **Port** — defaults to `80` for HTTP, `443` for HTTPS
- **Enabled/Disabled** — toggle individual targets independently

### Path-Based Routing

Multiple targets can be assigned unique path prefixes to route different URLs to different backends:

| Path | Target | Description |
|------|--------|-------------|
| `/` | Peer A (port 3000) | Main web application |
| `/api` | Peer B (port 8080) | API service |
| `/docs` | Resource C (port 80) | Documentation server |

Each path must be unique within a service.

---

## Domains

### Cloud Deployments (Free domains)

Format: `<subdomain>.<nonce>.<region>.proxy.netbird.io`

Example: `myapp.abc123.eu.proxy.netbird.io`

Shown with a **Free** badge in the domain selector. Available immediately, no DNS configuration required.

### Self-Hosted Deployments (Cluster domains)

Format: `<subdomain>.proxy.<your-domain>`

Example: `myapp.proxy.mycompany.com`

Set via the `NB_PROXY_DOMAIN` environment variable on the proxy instance. Shown with a **Cluster** badge.

**DNS records required for self-hosted:** Create an **A** record for your NetBird host and **CNAME** records for `proxy` and `*.proxy` pointing to that host.

### Custom Domains

Use your own domain (e.g., `app.example.com`) by adding a CNAME record pointing to your proxy cluster address.

#### Adding a Custom Domain

1. Navigate to **Reverse Proxy** → **Custom Domains** in the dashboard.
2. Click **Add Domain** and enter your domain name (e.g., `proxy.example.com`).
3. Select the target **proxy cluster**.
4. Click **Save** — the domain appears with **Pending Verification** status.

#### Verifying a Custom Domain

Create a wildcard CNAME record in your DNS provider:

| Record Type | Name | Value (Cloud) | Value (Self-hosted) |
|-------------|------|---------------|---------------------|
| `CNAME` | `*.proxy.example.com` | `eu.proxy.netbird.io` | `proxy.mycompany.com` |

Then in the dashboard, click **Verify Domain** → **Start Verification**. NetBird performs a CNAME lookup. DNS propagation can take up to 24 hours.

#### Custom Domain Troubleshooting

- **Pending Verification**: confirm the wildcard CNAME for `*.<your-domain>` is set and has propagated.
- **CNAME pointing to wrong cluster**: the CNAME must resolve to the cluster selected when adding the domain.
- **Domain already in use**: each custom domain is unique across all NetBird accounts.

---

## Authentication

Protect a service with one or more methods. When multiple methods are enabled, users choose which one to use.

| Method | Description | Best for |
|--------|-------------|----------|
| **SSO (Single Sign-On)** | OIDC via your identity provider; optionally restrict to specific groups | Team services, group-based access control |
| **Password** | Shared password; hashed with Argon2id | External partners, staging environments |
| **PIN Code** | Numeric PIN; hashed with Argon2id | Quick access, kiosk-style interfaces |
| **No authentication** | Publicly accessible to anyone | Public-facing websites, self-authenticating APIs |

**Session duration:** 24 hours for all methods. Sessions are scoped to individual services.

**Session tokens:** JWT signed with Ed25519 key pairs unique to each service.

**SSO on self-hosted with external IdP:** Register the reverse proxy callback URL with your IdP. See the [Enable Reverse Proxy migration guide](https://docs.netbird.io/selfhosted/migration/enable-reverse-proxy#configure-sso-for-external-identity-providers).

**Public service warning:** Saving a service without authentication shows a confirmation dialog warning that it will be publicly accessible.

### Common Authentication Combinations

| Combination | Use case |
|-------------|----------|
| SSO + Password | Team members use SSO; external collaborators use password |
| SSO + PIN Code | Team members use SSO; PIN for specific scenarios |
| Password + PIN Code | Different credentials for different groups |
| SSO + Password + PIN Code | Maximum flexibility |

---

## Service Statuses

| Status | Meaning |
|--------|---------|
| `pending` | Service created, being provisioned |
| `certificate_pending` | TLS certificate being issued (ACME/Let's Encrypt) |
| `active` | Live and routing traffic |
| `tunnel_not_created` | Proxy cluster has not established a WireGuard tunnel to target |
| `certificate_failed` | TLS certificate issuance failed — check ACME challenge port accessibility (443 for `tls-alpn-01`, 80 for `http-01`) and DNS resolution |
| `error` | Generic error — check service configuration and target availability |

---

## Quick Start

### Prerequisites

- At least one **peer** connected to your NetBird network, OR at least one **network** with resources and routing peers.
- A domain: Free (cloud) or Cluster (self-hosted) domains are available automatically, or configure a custom domain.
- **Self-hosted only:** At least one proxy instance (`netbirdio/netbird-proxy`) deployed and connected to your management server.
- Account with **Network Admin** role or higher.

### Step 1: Open Reverse Proxy page

Navigate to **Reverse Proxy** → **Services** and click **Add Service**.

### Step 2: Configure service details (Details tab)

1. Enter a **subdomain** (e.g., `myapp`).
2. Select a **base domain** (Free/Cluster badge, or a custom domain).
3. Click **Add Target** — select target type, then choose the peer or resource.
4. Set **protocol** and **port** for the target.
5. Optionally enter a **path** for path-based routing.

### Step 3: Configure authentication (Authentication tab)

- Enable **SSO** and optionally restrict to specific groups.
- Enable **Password** and enter a shared password.
- Enable **PIN Code** and enter a numeric PIN.
- Leave all disabled for public access (confirmation required).

### Step 4: Configure advanced settings (Settings tab)

| Setting | Description |
|---------|-------------|
| **Pass Host Header** | Forwards the original `Host` header to the backend instead of the target hostname. Enable when the backend needs to know its public domain. |
| **Rewrite Redirects** | Rewrites `Location` headers in backend responses to use the public domain, preventing users from being redirected to unreachable internal URLs. |

### Step 5: Create the service

Click **Add Service**. Monitor the service status until it shows `active`.

---

## Managing Services

- **Edit**: click a service in the list to open the edit modal.
- **Enable/Disable**: toggle the service on/off without deleting the configuration.
- **Delete**: permanently removes the service, domain, and TLS certificate. Cannot be undone.

### Managing Targets

- Add targets by clicking **Add Target** within a service.
- Remove targets to stop routing traffic to that backend.
- Toggle individual targets on/off independently.

### Expose from Networks Page

On the **Networks** page, click **Expose Service** on any resource to open the reverse proxy creation modal with that resource pre-populated as a target.

---

## Self-Hosted Proxy Setup

Self-hosted deployments require a separate `netbirdio/netbird-proxy` container connecting to your management server via gRPC.

**New deployments (v0.65.0+):** If you used `getting-started.sh` and selected built-in Traefik, the proxy container is already included.

**Existing deployments:** Follow the [Enable Reverse Proxy migration guide](https://docs.netbird.io/selfhosted/migration/enable-reverse-proxy).

### TLS Certificate Configuration

**ACME mode (Let's Encrypt):**

| Environment variable | Description |
|----------------------|-------------|
| `NB_PROXY_ACME_CERTIFICATES` | Set to `true` to enable automatic certificate provisioning |
| `NB_PROXY_ACME_CHALLENGE_TYPE` | Challenge type: `tls-alpn-01` (default, uses port 443) or `http-01` (requires port 80) |

**Static certificate mode:**

| Environment variable | Description |
|----------------------|-------------|
| `NB_PROXY_CERTIFICATE_FILE` | TLS certificate filename (default: `tls.crt`) |
| `NB_PROXY_CERTIFICATE_KEY_FILE` | TLS private key filename (default: `tls.key`) |
| `NB_PROXY_CERTIFICATE_DIRECTORY` | Directory for certificate files (default: `./certs`) |

Static certificates support hot-reload via file watching — no restart required when the certificate changes on disk.

### High Availability

Multiple proxy instances with the same `NB_PROXY_DOMAIN` value form a single proxy cluster with automatic failover. See [Running Multiple Proxy Instances](https://docs.netbird.io/selfhosted/maintenance/scaling/multiple-proxy-instances).

---

## Backend Service Configuration (Trusted Proxies)

When a backend service sits behind the NetBird Reverse Proxy, it sees the proxy's **NetBird IP** (from the `100.64.0.0/10` CGNAT range) as the client IP instead of the real user's IP. Many applications require configuring trusted proxies to correctly read the real client IP from the `X-Forwarded-For` header.

**Cloud-hosted usage limits:** NetBird's cloud-hosted Reverse Proxy is provided on a shared, best-effort basis. High-volume or sustained data transfers may be subject to bandwidth limits, rate limiting, and fair-use policies under the [NetBird Terms of Service](https://netbird.io/terms). These restrictions do not apply if you self-host NetBird.

**Important:** Do not hardcode a specific NetBird IP (e.g., `100.64.0.47`) — NetBird IPs can change on peer restart. Instead, trust the entire CGNAT range:

```
100.64.0.0/10
```

If you prefer a narrower range, you can use your account's specific network range (typically `100.64.0.0/16`). Find it via the API: `GET /api/accounts/{id}` — look for the `network_range` field. The full `/10` range is recommended for simplicity and resilience.

**Docker bridge networks:** If the NetBird peer and backend run as containers on the same Docker bridge network (e.g., Home Assistant NetBird add-on), traffic does not traverse the WireGuard tunnel. The backend sees the NetBird container's **Docker bridge IP** (typically from `172.16.0.0/12`) instead. Add both ranges:

```
100.64.0.0/10
172.16.0.0/12
```

The default Docker bridge subnet is `172.17.0.0/16`, but Docker and add-on managers may assign addresses from anywhere in the `172.16.0.0/12` range. Run `docker network inspect <network>` to find the exact subnet if you need a narrower entry.

### Jellyfin

1. Go to **Dashboard** → **Networking**.
2. In the **Known Proxies** field, enter `100.64.0.0/10`.
3. Click **Save**.

Jellyfin also supports a **Base URL** setting, which can be used in conjunction with a path prefix configured on the reverse proxy service.

### Home Assistant

In `configuration.yaml`:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 100.64.0.0/10
    # Add if using the NetBird Docker add-on on the same host:
    - 172.16.0.0/12
```

Restart Home Assistant after making this change.

### Nextcloud

In `config/config.php`:

```php
'trusted_proxies' =>
  array (
    0 => '100.64.0.0/10',
  ),
```

If you're running **Nextcloud AIO**, trusted proxies are configured automatically during setup. If manual adjustment is needed, exec into the container:

```bash
sudo docker exec -u 0 -it nextcloud-aio-nextcloud /bin/bash
```

### Verifying Trusted Proxy Configuration

1. Access your service through the NetBird reverse proxy URL.
2. Check the backend's access logs — the logged client IP should be the real user's IP, not a `100.64.x.x` address.
3. If the backend still shows a NetBird IP, verify that **Pass Host Header** is enabled in the service Settings tab and that the backend was restarted after changing its configuration.

---

## Access Logs

Access logs are available at **Activity** → **Proxy Events** in the dashboard.

| Field | Description |
|-------|-------------|
| **Timestamp** | When the request occurred |
| **Method** | HTTP method (GET, POST, PUT, DELETE, etc.) |
| **Host** | Domain the request was made to |
| **Path** | URL path requested |
| **Status Code** | HTTP status code returned |
| **Duration** | Request duration in milliseconds |
| **Source IP** | Client's IP address |
| **Auth Method** | Authentication method used (SSO, password, PIN, or none) |
| **User ID** | Authenticated user's ID (SSO only) |
| **Country** | Country of origin based on source IP geolocation |
| **City** | City of origin based on source IP geolocation |
| **Reason** | Reason for denial (if applicable) |

**Log categories:**
- **Allowed** (`2xx`): successful requests with the auth method used.
- **Denied** (`401`/`403`): failed authentication with reason.
- **Errors** (`5xx`): backend errors or proxy issues.

---

---

## REST API — Target Type Constraints (Operational)

The management REST API `POST /api/reverse-proxies/services` only accepts **two valid `target_type` values**:

| `target_type` | Dashboard label | Requires |
|---------------|-----------------|----------|
| `peer` | Peer | A registered NetBird WireGuard peer ID (from `/api/peers`) |
| `host` | Host / Domain / Subnet | A NetBird network resource ID (from `/api/networks/{id}/resources`) |

The dashboard shows four target types (`Peer`, `Host`, `Domain`, `Subnet`) but all non-peer types resolve to network resource IDs and use `target_type=host` in the API. Using `target_type=network_resource` returns `422 invalid target_type`.

### `target_type=peer`

Works when the backend machine is a registered NetBird peer. Management resolves the peer's WireGuard IP and pushes it as the proxy target URL.

**Prerequisite:** The host running the backend service must be enrolled in NetBird (installed `netbird` client, `netbird up` run, peer visible in `/api/peers`). If the host is not a registered peer, this type cannot be used.

### `target_type=host`

Works when a NetBird **network resource** has already been created for the target subnet/IP and a routing peer has been assigned to that network. Management resolves the resource via the routing peer.

**Prerequisite:**
1. A NetBird network must exist with a routing peer (a registered peer that has network access to the backend).
2. A network resource must exist for the target subnet or IP within that network.
3. The `id` field of the network resource is passed as `target_id`.

If the resource or network does not exist, the API returns `422 resource target not found in account`.

---

## Migration Blocker: Backend Host Not a Peer

If the host running the Docker backend containers (e.g., `docker`) is **not** a registered NetBird peer, the service API has no valid target:

- `target_type=peer` — fails: peer ID does not exist in peer list.
- `target_type=host` — fails: requires a routing peer with network access to the subnet.
- No direct URL target type exists in the REST API.

Even if the proxy container runs on the same Docker bridge network as the backend containers and could reach them directly by IP, the management server has no API mechanism to express a direct URL target without going through a peer or network resource.

**Resolution options (pick one):**

1. **Register `docker` as a NetBird peer** — install `netbird` client on the `docker` host, run `netbird up`, wait for it to appear in the peer list, then use `target_type=peer` with the peer ID.

2. **Create a network resource via a routing peer** — add `docker` as a routing peer in a NetBird network, create a network resource for `172.0.x.x/24` (or a specific container IP), then use `target_type=host` with the resource ID.

---

## Management Source Location

The reverse-proxy management code lives at:

```
netbirdio/netbird / management/internals/modules/reverseproxy/
```

*Not* at `management/server/http/handlers/reverseproxy/` — that path does not exist.

The proxy container embeds a WireGuard client (`github.com/netbirdio/netbird/client/embed`) and receives `ProxyMapping → PathMapping{path, target, options}` from the management server via gRPC stream.

---

---

## Operational: TCP SNI Routing Conflict with Traefik

When the NetBird proxy container is on a Docker host also running **Traefik**, services are exposed via **TCP SNI passthrough** labels on the proxy container. These labels route HTTPS traffic at the TCP layer (before Traefik's HTTP routers see it).

**Conflict:** If you later revert a service back to Traefik (HTTP router with certresolver), the TCP SNI passthrough rule will still intercept the traffic first. Users see the **NetBird proxy's error page** instead of the Traefik-routed service.

**Fix:** Remove the TCP SNI passthrough label for any hostname being reverted to Traefik. Example — removing `vault.bartschnet.de`:

```yaml
# REMOVE these labels from the proxy container:
- "traefik.tcp.routers.netbird-vault.rule=HostSNI(`vault.bartschnet.de`)"
- "traefik.tcp.routers.netbird-vault.tls.passthrough=true"
- "traefik.tcp.routers.netbird-vault.service=netbird-proxy-tls"
```

Keep only the base and wildcard proxy labels (`proxy.bartschnet.de`, `*.proxy.bartschnet.de`).

After removing the TCP labels, redeploy the proxy container so Traefik picks up the change. The Traefik HTTP router for that hostname can then take over.

---

## Operational: Proxy Container Running as Root — nbnet.NewDialer() Failure

The NetBird proxy container embeds a WireGuard-aware dialer (`nbnet.NewDialer()`) used when making gRPC calls **and the process runs as root on Linux**. The `nbnet.NewDialer()` requires a WireGuard interface; the proxy container has none (it is not a full NetBird peer).

**Symptom:** Services return **504 Gateway Timeout**. Proxy logs show:
```
context deadline exceeded
```
at `client/internal/auth/auth.go` — specifically the `AddPeer` gRPC call that triggers `embed.Client.Start()` with a 30-second timeout.

**Root cause:** `client/grpc/dialer_generic.go` selects `nbnet.NewDialer()` when `os.Getuid() == 0` on Linux. As non-root, it falls back to the standard `net.Dialer`, which works correctly.

**Fix:** Run the proxy service as a non-root user:

```yaml
services:
  proxy:
    image: netbirdio/reverse-proxy:latest
    user: "1000:1000"   # ← non-root; falls back to net.Dialer, not nbnet.NewDialer
    volumes:
      - ./letsencrypt:/certs:ro
```

Ensure cert files in the mounted volume are readable by uid 1000 (`chmod o+r` on the cert directory and key/cert files).

---

## Operational: Domain-type Resources Log "invalid IP" — Reverse-Proxy Targets Silently Broken

**Symptom:** After creating a reverse-proxy service linked to a network resource, the proxy logs show:

```
ERRO failed to parse target URL for path, skipping
error: parse "http://invalid%20IP:8000/": invalid URL escape "%20"
target: http://invalid%20IP:8000/
```

The proxy receives a mapping update with `target:"http://invalid%20IP:8000/"` — the literal string `"invalid IP"` URL-encoded.

**Root cause (confirmed from NetBird source — `management/internals/modules/reverseproxy/service/manager/manager.go`):**

The management server's `replaceHostByLookup()` function resolves each service target's `host` field based on `target_type`:

```go
case service.TargetTypeHost:
    resource, err := m.store.GetNetworkResourceByID(...)
    target.Host = resource.Prefix.Addr().String()   // ← uses IP prefix field

case service.TargetTypeDomain:
    resource, err := m.store.GetNetworkResourceByID(...)
    target.Host = resource.Domain                   // ← uses DNS name field
```

When a service is created with **`target_type: host`** but points to a **`domain`-type network resource** (one created with a DNS hostname like `grampsweb.proxy`):

1. The resource's `Prefix` field (a `netip.Prefix`) is zero-valued — domain resources don't populate the IP prefix
2. `netip.Prefix{}.Addr()` returns `netip.Addr{}` (Go zero value)
3. `netip.Addr{}.String()` returns the literal string `"invalid IP"` — this is Go's standard library behavior for the zero `netip.Addr`
4. The proxy receives `http://invalid%20IP:8000/` — space URL-encoded
5. `url.Parse()` fails: `invalid URL escape "%20"` in a host position → route skipped entirely

**This is independent of the root-user/nbnet.NewDialer() issue** — even if the proxy runs as non-root, domain-type resource targets used with `target_type=host` will always fail.

**Two valid fixes:**

**Option A — Use `target_type: host` with an IP-address resource (recommended for static backends):**

```
# Create resource with actual IP address
address: 172.0.4.6    # ← IP address, type auto-set to "host"
# → management calls resource.Prefix.Addr().String() → "172.0.4.6" ✅
```

**Option B — Use `target_type: domain` with a domain-type resource (works when proxy can resolve DNS):**

```
# Create resource with DNS hostname
address: grampsweb.proxy    # ← Docker DNS name, type auto-set to "domain"
# → management calls resource.Domain → "grampsweb.proxy" ✅
# → proxy container must be on the same Docker network to resolve this DNS name
```

**Root cause in resource configuration:**

```
# BAD — target_type mismatch: host service → domain resource
Service:  target_type: host,   target_id: <resource with address="grampsweb.proxy">
Result:   resource.Prefix is zero → "invalid IP" → URL parse error → 404

# GOOD — matching target_type: host service → host resource (IP address)
Service:  target_type: host,   target_id: <resource with address="172.0.4.6">
Result:   resource.Prefix.Addr() → "172.0.4.6" → http://172.0.4.6:5000/ ✅

# GOOD — matching target_type: domain service → domain resource (DNS name)
Service:  target_type: domain, target_id: <resource with address="grampsweb.proxy">
Result:   resource.Domain → "grampsweb.proxy" → http://grampsweb.proxy:5000/ ✅
```

**Fix:** Either create network resources with IP addresses and use `target_type: host`, or use the correct `target_type: domain` when the resource has a DNS hostname. The `target_type` in the service must match the address type of the linked network resource.

```bash
# Create a host-type resource with IP address (use target_type: host in service)
curl -s -X POST "https://netbird.bartschnet.de/api/networks/<network_id>/resources" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "grampsweb",
    "address": "172.0.4.6",   ← IP address → type auto-detected as "host"
    "enabled": true,
    "groups": [{"id": "<all-group-id>"}]
  }'

# OR create a domain-type resource (use target_type: domain in service)
# (when proxy container can resolve the DNS name)
curl -s -X POST "https://netbird.bartschnet.de/api/networks/<network_id>/resources" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "grampsweb",
    "address": "grampsweb.proxy",   ← DNS name → type auto-detected as "domain"
    "enabled": true,
    "groups": [{"id": "<all-group-id>"}]
  }'
```

**Verify the resource type and service target_type alignment** before deploying:
```bash
# Check resource types
curl -s "https://netbird.bartschnet.de/api/networks/<id>/resources" \
  -H "Authorization: Token <token>" | jq '.[] | {name, address, type}'
# host resource:   "type": "host",   "address": "172.x.x.x"
# domain resource: "type": "domain", "address": "hostname.local"

# Check service targets match resource type
curl -s "https://netbird.bartschnet.de/api/reverse-proxies/services/<id>" \
  -H "Authorization: Token <token>" | jq '.targets[] | {target_type, host, port}'
# If "host" in targets is "invalid IP" → target_type/resource type mismatch!
```

---

## References

- [Reverse Proxy Overview](https://docs.netbird.io/manage/reverse-proxy)
- [Authentication](https://docs.netbird.io/manage/reverse-proxy/authentication)
- [Custom Domains](https://docs.netbird.io/manage/reverse-proxy/custom-domains)
- [Access Logs](https://docs.netbird.io/manage/reverse-proxy/access-logs)
- [Backend Service Configuration](https://docs.netbird.io/manage/reverse-proxy/service-configuration)
- [Expose from CLI](https://docs.netbird.io/manage/reverse-proxy/expose-from-cli)
- [Enable Reverse Proxy migration guide](https://docs.netbird.io/selfhosted/migration/enable-reverse-proxy)
- [Multiple Proxy Instances](https://docs.netbird.io/selfhosted/maintenance/scaling/multiple-proxy-instances)
- [Networks](https://docs.netbird.io/manage/networks)
