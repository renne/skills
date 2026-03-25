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

Use the `occ` CLI to create a client. The correct command is **`oidc:create`** with **positional arguments** (NOT `--id`/`--uri`/`--signing-alg` flags):

```bash
docker exec -u www-data <nextcloud-container> php occ oidc:create \
  [-a|--algorithm <RS256|ES256>] \
  [-f|--flow <code|implicit>] \
  [-t|--type <public|confidential>] \
  [--client_id [<client-id>]] \
  [--client_secret [<client-secret>]] \
  -- "<display-name>" "<redirect-uri-1>" ["<redirect-uri-2>" ...]
```

**The name and redirect URIs are positional arguments, not named flags.**

Example (confidential client for Dex federation):
```bash
docker exec -u www-data nextcloud-aio-nextcloud php occ oidc:create \
  --algorithm RS256 \
  --flow code \
  --type confidential \
  -- "NetBird-Dex" "https://netbird.example.de/oauth2/callback"
```

### Client ID / Secret constraints

| Constraint | Rule |
|------------|------|
| Characters | Only `A-Za-z0-9` (no hyphens, underscores, or special characters) |
| Minimum length | 32 characters |
| Maximum length | 64 characters |

⚠️ Violating these constraints gives the error: `The client ID is invalid`.

### List registered clients

```bash
docker exec -u www-data <nextcloud-container> php occ oidc:list
```

### Remove a client

```bash
# The command is oidc:remove (NOT oidc:delete)
docker exec -u www-data <nextcloud-container> php occ oidc:remove <client-id>
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

## Using Nextcloud as IdP for NetBird (via Dex connector federation)

The self-hosted NetBird combined image (`netbirdio/netbird-server`) embeds Dex as its OIDC issuer and **cannot** bypass it. To add Nextcloud login to NetBird, federate Dex to Nextcloud via a Dex OIDC connector:

```
NetBird Dashboard → Embedded Dex (https://netbird.example.de/oauth2)
                          ↕  OIDC connector
                     Nextcloud (https://nextcloud.example.de)
```

### 1. Create a confidential Nextcloud OIDC client for Dex

```bash
# Run on the host that runs Nextcloud
ssh docker "docker exec -u www-data nextcloud-aio-nextcloud php occ oidc:create \
  --algorithm RS256 \
  --flow code \
  --type confidential \
  -- 'NetBird-Dex' 'https://netbird.example.de/oauth2/callback'"
```

Save the printed `client_id` and `client_secret` immediately — they are shown only once.

**Client ID constraints:** alphanumeric only (A-Za-z0-9), 32–64 characters.

### 2. Insert the Dex OIDC connector directly into `idp.db`

NetBird's management API and the NetBird dashboard have no UI for managing Dex connectors. The only way to add one is to insert directly into Dex's SQLite database:

```bash
# On vps1 (where NetBird runs)
DB=/var/lib/docker/volumes/netbird_netbird_data/_data/idp.db

CONFIG=$(cat <<'EOF'
{"issuer":"https://nextcloud.example.de","clientID":"<client-id>","clientSecret":"<client-secret>","redirectURI":"https://netbird.example.de/oauth2/callback","scopes":["openid","profile","email"],"insecureEnableGroups":true,"insecureSkipEmailVerified":true}
EOF
)

sqlite3 "$DB" "INSERT INTO connector (id, type, name, resource_version, config) VALUES ('nextcloud', 'oidc', 'Nextcloud', '', '$CONFIG');"

# Verify
sqlite3 "$DB" "SELECT id, type, name FROM connector;"
# Expected: local|local|Email  AND  nextcloud|oidc|Nextcloud
```

Then restart the NetBird containers:
```bash
cd /srv/docker/netbird && docker compose restart
```

### 3. Verify the button appears

```bash
curl -s https://netbird.example.de/oauth2/auth?client_id=netbird-dashboard\&response_type=code\&redirect_uri=https://netbird.example.de | grep -i nextcloud
# Should show: /oauth2/auth/nextcloud and "Continue with Nextcloud"
```

### Important: `insecureSkipEmailVerified: true` is REQUIRED

Nextcloud tokens never include `email_verified: true`. Without this flag, Dex will reject every Nextcloud login.

### Dex connector table schema

```
connector(id TEXT PK, type TEXT, name TEXT, resource_version TEXT, config BLOB)
```

- `config` is a JSON blob; the `local` connector uses `{}`
- Connector type for Nextcloud (and any generic OIDC provider): `"oidc"`
- Named types (not needed here): `zitadel`, `entra`, `okta`, `pocketid`, `authentik`, `keycloak`, `google`, `microsoft`
- Connectors persist across Dex restarts; `store.db` (NetBird management DB) has **no** `identity_providers` table and does not overwrite Dex connectors

### config.yaml must keep embedded Dex as issuer

```yaml
server:
  auth:
    issuer: "https://netbird.example.de/oauth2"   # ← ALWAYS the embedded Dex URL
    localAuthDisabled: false                        # keep true to allow local login too
```

Setting `issuer` to the Nextcloud URL breaks everything — do not do it.

### dashboard.env must point at embedded Dex

```env
AUTH_CLIENT_ID=netbird-dashboard
AUTH_AUDIENCE=netbird-dashboard
AUTH_AUTHORITY=https://netbird.example.de/oauth2
AUTH_CLIENT_SECRET=
```

## References

- [Nextcloud user_oidc app (GitHub)](https://github.com/nextcloud/user_oidc)
- [Nextcloud OIDC IdP documentation](https://github.com/nextcloud/user_oidc#openid-connect-identity-provider)
- [oauth2-proxy OIDC provider docs](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/openid_connect)
- [oauth2-proxy Docker Compose example](https://oauth2-proxy.github.io/oauth2-proxy/installation#docker-compose)
