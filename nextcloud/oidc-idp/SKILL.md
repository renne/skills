---
name: oidc-idp
description: Using Nextcloud as an OpenID Connect Identity Provider (IdP) — registering OIDC clients, configuring oauth2-proxy or other relying parties, and avoiding Nextcloud-specific quirks. Use when setting up SSO with Nextcloud as the identity source for other services.
---
# Nextcloud as OIDC Identity Provider

Nextcloud can act as an OIDC IdP via the **`user_oidc`** app (installed from the Nextcloud App Store). This enables SSO for external services — browsers authenticate with Nextcloud, which issues OIDC tokens to relying parties.

## Prerequisites

1. **`user_oidc` app** must be installed and enabled in Nextcloud.
2. Nextcloud must be reachable over HTTPS (required for OIDC discovery).

## OIDC Discovery Endpoint

All relying parties need the discovery URL:

```
https://<nextcloud-domain>/.well-known/openid-configuration
```

Example:
```
https://cloud.example.de/.well-known/openid-configuration
```

## Registering an OIDC Client

Use the `occ` CLI to create a client. The correct command is **`oidc:create`**:

```bash
docker exec -u www-data <nextcloud-container> php occ oidc:create \
  --name="<display-name>" \
  --client-id="<client-id>" \
  --client-secret="<client-secret>" \
  --redirect-uri="https://<app-domain>/oauth2/callback" \
  --flow="code" \
  --signing-alg="RS256"
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| `--name` | Human-readable display name shown in Nextcloud's connected apps list |
| `--client-id` | Unique identifier for the client (can be auto-generated or set manually) |
| `--client-secret` | Secret shared between Nextcloud and the relying party |
| `--redirect-uri` | Must match exactly what the relying party sends; can be specified multiple times |
| `--flow` | Use `code` (Authorization Code Flow — most secure and broadly supported) |
| `--signing-alg` | Use `RS256` (asymmetric — preferred) or `HS256` |

### List registered clients

```bash
docker exec -u www-data <nextcloud-container> php occ oidc:list
```

### Delete a client

```bash
docker exec -u www-data <nextcloud-container> php occ oidc:delete <client-id>
```

## Configuring oauth2-proxy

When using Nextcloud as IdP for [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/), set:

```yaml
command:
  - --provider=oidc
  - --oidc-issuer-url=https://<nextcloud-domain>
  - --client-id=<client-id>
  - --client-secret=<client-secret>
  - --redirect-url=https://<app-domain>/oauth2/callback
  - --email-domain=*
  - --upstream=http://<backend-service>:<port>
  - --cookie-secret=<32-byte-base64-secret>
  - --insecure-oidc-allow-unverified-email=true   # ← REQUIRED for Nextcloud
```

### Generate cookie secret

```bash
openssl rand -base64 32
```

## Known Quirks and Limitations

### ⚠️ `email_verified` is never `true`

Nextcloud's OIDC tokens do **not** include `email_verified: true`. oauth2-proxy (and some other relying parties) reject tokens where `email_verified` is absent or `false` with a 500 error.

**Fix:** Always add `--insecure-oidc-allow-unverified-email=true` to oauth2-proxy. Without it, the callback returns HTTP 500 with no useful error in the browser.

### ⚠️ `X-Auth-Request-User` is `preferred_username`, not email

When oauth2-proxy injects headers after OIDC auth, `X-Auth-Request-User` contains the Nextcloud **username** (e.g., `renne`), not the email address. If your backend identifies users by email, you may need to use `X-Auth-Request-Email` instead.

### ⚠️ Command is `oidc:create`, not `user_oidc:client`

The OCC command to create a client is `oidc:create`. Older documentation and forum posts may reference `user_oidc:client` or `user_oidc:register` — these do not exist in current versions.

### ⚠️ Redirect URI must match exactly

Nextcloud validates the redirect URI strictly. Include the exact URI (scheme, domain, port if non-standard, path) when registering. For oauth2-proxy the path is always `/oauth2/callback`.

### ⚠️ No `email_verified` claim — silent 500 in some setups

If your relying party validates the `email_verified` claim and Nextcloud omits it, you'll get a silent 500 error. The workaround differs by client:
- **oauth2-proxy**: `--insecure-oidc-allow-unverified-email=true`
- **Keycloak (bridging)**: map the claim with a default value of `true`
- **Other clients**: check if the library allows skipping email verification

## Example: oauth2-proxy in Docker Compose

```yaml
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:latest
    command:
      - --http-address=0.0.0.0:4180
      - --provider=oidc
      - --oidc-issuer-url=https://cloud.example.de
      - --client-id=${OIDC_CLIENT_ID}
      - --client-secret=${OIDC_CLIENT_SECRET}
      - --redirect-url=https://app.example.de/oauth2/callback
      - --upstream=http://app-backend:3000
      - --email-domain=*
      - --cookie-secret=${OAUTH2_PROXY_COOKIE_SECRET}
      - --insecure-oidc-allow-unverified-email=true
      - --pass-access-token=true
      - --pass-user-headers=true
      - --set-xauthrequest=true
    environment:
      OIDC_CLIENT_ID: ${OIDC_CLIENT_ID}
      OIDC_CLIENT_SECRET: ${OIDC_CLIENT_SECRET}
      OAUTH2_PROXY_COOKIE_SECRET: ${OAUTH2_PROXY_COOKIE_SECRET}
```

### Injected headers (available to upstream)

After successful authentication, oauth2-proxy injects:

| Header | Content |
|--------|---------|
| `X-Auth-Request-User` | Nextcloud username (`preferred_username`) |
| `X-Auth-Request-Email` | User's email address |
| `X-Auth-Request-Groups` | Comma-separated group memberships |
| `X-Auth-Request-Access-Token` | Raw OIDC access token (requires `--pass-access-token=true`) |

### Security note

oauth2-proxy **strips** any client-sent `X-Auth-Request-*` headers before passing to upstream. This prevents header injection attacks — upstream can trust these headers only come from oauth2-proxy.

## Traefik + oauth2-proxy: ForwardAuth pattern

When placing oauth2-proxy behind Traefik as a ForwardAuth gate:

```yaml
# Traefik labels on the protected service
labels:
  - "traefik.http.routers.myapp.middlewares=oauth2-proxy-auth"
  - "traefik.http.middlewares.oauth2-proxy-auth.forwardauth.address=http://oauth2-proxy:4180"
  - "traefik.http.middlewares.oauth2-proxy-auth.forwardauth.trustForwardHeader=true"
  - "traefik.http.middlewares.oauth2-proxy-auth.forwardauth.authResponseHeaders=X-Auth-Request-User,X-Auth-Request-Email,X-Auth-Request-Groups"
```

⚠️ If you use oauth2-proxy as an **upstream proxy** (not ForwardAuth), do not add ForwardAuth middleware — the proxy handles auth itself. Use ForwardAuth only when the app is behind Traefik directly with oauth2-proxy as a sidecar auth service.

## Token Claims from Nextcloud

A decoded Nextcloud OIDC access token typically includes:

```json
{
  "iss": "https://cloud.example.de",
  "sub": "<user-uuid>",
  "aud": "<client-id>",
  "exp": 1234567890,
  "iat": 1234567890,
  "auth_time": 1234567890,
  "nonce": "...",
  "name": "Full Name",
  "preferred_username": "username",
  "email": "user@example.de",
  "groups": ["admin", "users"]
}
```

Note: `email_verified` is **absent** (not `false`) — this is what triggers oauth2-proxy 500 errors.

## References

- [Nextcloud user_oidc app (GitHub)](https://github.com/nextcloud/user_oidc)
- [Nextcloud OIDC IdP documentation](https://github.com/nextcloud/user_oidc#openid-connect-identity-provider)
- [oauth2-proxy OIDC provider docs](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/openid_connect)
- [oauth2-proxy Docker Compose example](https://oauth2-proxy.github.io/oauth2-proxy/installation#docker-compose)
