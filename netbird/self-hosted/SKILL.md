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
2. The reverse proxy to use (Traefik recommended for automatic TLS)
3. Whether to enable the NetBird relay/proxy service

After completion, start the stack:

```bash
cd netbird   # directory created by the script
docker compose up -d
```

The dashboard is accessible at `https://netbird.example.com`.

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

```bash
# Tail logs for all services
docker compose logs -f

# Restart a specific service
docker compose restart management
```

---

## References

- [Self-Hosting Quickstart Guide](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
- [Advanced Self-Hosted Guide](https://docs.netbird.io/selfhosted/selfhosted-guide)
- [Configuration Files Reference](https://docs.netbird.io/selfhosted/configuration-files)
- [Identity Providers](https://docs.netbird.io/selfhosted/identity-providers)
- [NetBird Releases](https://github.com/netbirdio/netbird/releases)
