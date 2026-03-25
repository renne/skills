---
name: local-idp
description: NetBird built-in local user management (since v0.62) using embedded Dex OIDC server — no external identity provider required. Covers first-time setup, creating users, sending invite links, managing invites, switching to an external IdP, and encryption key backup. Use when deploying a self-hosted NetBird instance without Zitadel/Keycloak/Auth0, managing local users, or inviting new team members to a self-hosted deployment.
---
# NetBird Local Identity Provider (Built-in User Management)

Sources:
- https://docs.netbird.io/selfhosted/selfhosted-quickstart
- https://docs.netbird.io/about-netbird/self-hosted-vs-cloud
- https://docs.netbird.io/manage/team/add-users-to-your-network

## Overview

Since **NetBird v0.62**, self-hosted deployments no longer require an external identity provider (IdP) like Zitadel, Keycloak, or Auth0. The Management service includes an **embedded [Dex](https://dexidp.io/) OIDC server** that handles local user accounts directly.

### Resource Comparison

| | v0.62+ (local IdP) | Pre-v0.62 (external IdP) |
|---|---|---|
| **Containers** | 4–5 | 7+ |
| **RAM requirement** | ~1 GB | 2–4 GB |
| **External IdP needed** | No | Yes |
| **External IdP supported** | Optional | Required |

---

## First-Time Setup

After starting the NetBird stack for the first time, **no users exist**. Navigate to:

```
https://netbird.example.com/setup
```

This creates the initial admin account. Use this account to log in to the dashboard and manage the installation.

---

## Creating Users (Invite Flow)

To add more users to a self-hosted deployment:

1. Log in as admin → **Team → Users → Add User**
2. Switch to the **Invite User** tab
3. Fill in the new user's **name** and **email**
4. Copy the generated **invite link** (single-use by default)
5. Send the link to the user

**User experience:**
1. User opens the invite link in a browser
2. Sets their password
3. Is redirected to the NetBird dashboard and logs in

---

## Managing Pending Invites

View and manage invites that have not yet been accepted:

**Team → Users → Show Invites**

From here you can:
- **Regenerate** an invite link (the old link is invalidated)
- **Delete** a pending invite (removes the user before they finish registration)

---

## External IdPs (Optional)

External identity providers can be added **in addition to** or **instead of** local user management:

- Supported providers: Google, Microsoft (Azure AD), Okta, GitHub, and any OIDC-compatible IdP
- **Multiple IdPs** can be configured simultaneously (since v0.62)
- Configure via the dashboard: **Settings → Identity Providers → Add Provider**

Cloud-only features enabled by external IdPs:
- **IdP group sync / SCIM provisioning** (sync groups from Okta, Azure AD, etc.)
- **User invites** (built into the cloud dashboard)

---

## Encryption Key

The encryption key protects stored credentials and actor identities in the event log. It is critical infrastructure.

### Where It Is Configured

In `config.yaml` (combined container, v0.62+):

```yaml
server:
  store:
    encryptionKey: "<hex-encoded-32-byte-key>"
```

### Generating a Key

```bash
openssl rand -hex 32
```

### CRITICAL: Backup This Key

- If the encryption key is lost, **all stored credentials and audit log actor names become permanently unreadable**.
- Store the key in a password manager or secrets vault (e.g., Bitwarden, HashiCorp Vault, 1Password).
- The key must be consistent across all management container instances (for HA deployments).

---

## When You Still Need an External IdP

Use an external IdP when:
- Your organization requires **SSO** with an existing directory (Azure AD, Okta, Google Workspace)
- You need **IdP group sync** or **SCIM provisioning** to automate user/group management
- You need **user approval workflows** (cloud-only feature)
- You have many users and want centralized lifecycle management

For small teams, homelabs, or PoC deployments, the built-in local IdP is sufficient.

---

## Cloud-Only User Management Features

The following features are available in NetBird **cloud** but **not** in self-hosted deployments:

| Feature | Notes |
|---------|-------|
| IdP group sync / SCIM | Syncs groups from Okta, Azure AD, etc. |
| Peer approval | Require admin approval before a peer joins |
| MSP portal | Manage multiple tenant networks |
| EDR integrations (CrowdStrike) | Endpoint detection via posture checks |

---

## References

- [Self-Hosted Quickstart](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
- [Self-hosted vs Cloud](https://docs.netbird.io/about-netbird/self-hosted-vs-cloud)
- [Add Users to Your Network](https://docs.netbird.io/manage/team/add-users-to-your-network)
