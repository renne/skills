---
name: admin-page
description: Vaultwarden admin page guide covering how to enable the admin panel, secure the ADMIN_TOKEN with argon2id hashing, manage users and organizations, configure settings via the admin UI, and troubleshoot token issues. Use when setting up the Vaultwarden admin panel, securing the admin token, managing user accounts, inviting users when signups are disabled, or accessing diagnostics.
---
# Vaultwarden Admin Page

Sources: https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page and https://github.com/dani-garcia/vaultwarden/wiki/Disable-admin-token

## Overview

The Vaultwarden admin panel (`/admin`) allows server administrators to:
- Configure Vaultwarden settings (generating `config.json`)
- View, invite, disable, and delete users
- Manage organizations
- Access diagnostics and generate the Support String

> **Important:** Enable HTTPS before enabling the admin page to avoid MITM attacks.

---

## Enabling the Admin Page

Set `ADMIN_TOKEN` to a long, randomly-generated secret:

```bash
# Generate a secure random token
openssl rand -base64 48
```

```bash
docker run -d --name vaultwarden \
  -e ADMIN_TOKEN=your_generated_token \
  -v /vw-data/:/data/ \
  -p 80:80 \
  vaultwarden/server:latest
```

Or in `compose.yaml`:

```yaml
services:
  vaultwarden:
    environment:
      ADMIN_TOKEN: "your_generated_token"
```

Access the admin panel at `https://your-domain/admin`.

---

## Securing the ADMIN_TOKEN (Argon2id Hashing)

Storing the token in plaintext is insecure. Hash it with argon2id to store only the PHC string.

### Generate a PHC hash using Vaultwarden

```bash
# Via the binary directly
./vaultwarden hash

# Via Docker (temporary container)
docker run --rm -it vaultwarden/server /vaultwarden hash

# Via a running container
docker exec -it vaultwarden /vaultwarden hash --preset owasp
```

The command asks for a password twice and outputs a PHC string like:
```
$argon2id$v=19$m=65540,t=3,p=4$...
```

### Generate using the `argon2` CLI

```bash
# Bitwarden defaults
echo -n "MySecretPassword" | argon2 "$(openssl rand -base64 32)" -e -id -k 65540 -t 3 -p 4

# OWASP minimum recommended settings
echo -n "MySecretPassword" | argon2 "$(openssl rand -base64 32)" -e -id -k 19456 -t 2 -p 1
```

### Using the PHC hash

**Environment variable:**

```bash
-e ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'
```

**Docker Compose `environment:` block** — escape every `$` as `$$`:

```yaml
environment:
  ADMIN_TOKEN: $$argon2id$$v=19$$m=65540,t=3,p=4$$...
```

> This escaping prevents Docker Compose from interpreting `$` as variable interpolation.
> You can automate the escaping: `| sed 's#\$#\$\$#g'`

**Docker Compose `.env` file** — use single `$` (no escaping needed):

```
# .env file (use single quotes)
VAULTWARDEN_ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'
```

```yaml
# compose.yaml
environment:
  - ADMIN_TOKEN=${VAULTWARDEN_ADMIN_TOKEN}
```

### Login with the hashed token

Use the **original password** (e.g., `MySecretPassword`) to log in — not the PHC string itself.

---

## Disabling the Admin Page

To disable the admin panel, ensure that:
- `ADMIN_TOKEN` environment variable is **not set**
- `DISABLE_ADMIN_TOKEN` environment variable is **not set**
- No `"admin_token"` key exists in `config.json`

Then recreate and restart the container.

---

## Admin Panel: Settings

The first time you save settings in the admin page, a `config.json` file is created in `DATA_FOLDER`. After this, **config.json takes highest precedence** — environment variables for those settings are ignored.

> Options in the **Read-Only Config** section cannot be modified via the admin page; they require a server restart and will be removed from `config.json` if you save.

### Session Management

- Default admin session length: **20 minutes** (configurable via `ADMIN_SESSION_LIFETIME`)
- To invalidate all admin sessions: delete `rsa_key.pem` from `DATA_FOLDER` and restart

---

## Admin Panel: Users

The Users overview lets you:
- See all registered users, their registration status, and organization memberships
- **Remove 2FA** providers for a user
- **Deauthorize sessions** for a user
- **Disable** or **delete** a user
- **Invite users** even when `SIGNUPS_ALLOWED=false`
- Change a user's organization role (blue=User, green=Manager/Custom, violet=Admin, orange=Owner)

> You cannot **add** a user to an organization via the admin panel — only promote existing members.

---

## Admin Panel: Organizations

The Organizations overview lets you:
- View all organizations
- Delete organizations (must delete the owner first if they have no other owners)

---

## Troubleshooting

**"You are using a plain text ADMIN_TOKEN which is insecure."**
- The PHC string is not being passed correctly; check for extra quotes or double `$$`
- Verify: `docker exec vaultwarden printenv ADMIN_TOKEN` — should show the raw `$argon2id$...` string
- If using `config.json`: `grep admin_token data/config.json`

**Cannot access `/admin`:**
- Verify `ADMIN_TOKEN` is set and the container was **recreated** (not just restarted)
- Check that HTTPS is working

**Admin token changes don't affect logged-in sessions:**
- JWT sessions remain valid until expiry; delete `rsa_key.pem` to invalidate all sessions

---

## References

- [Enabling admin page](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page)
- [Disable admin token](https://github.com/dani-garcia/vaultwarden/wiki/Disable-admin-token)
- [Configuration overview](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)
