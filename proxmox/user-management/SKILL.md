---
name: user-management
description: Proxmox VE user management covering authentication realms (PAM, LDAP, Active Directory, OpenID Connect), users, groups, roles, permission paths, API tokens, and two-factor authentication. Use when creating or managing users and groups in Proxmox VE, assigning roles and access permissions, configuring external authentication sources, or generating API tokens for automation.
---
# Proxmox VE User Management

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE has a built-in role-based access control (RBAC) system. Users are authenticated against a **realm** (authentication backend) and assigned **roles** on **resource paths**. Configuration is stored cluster-wide in `/etc/pve/user.cfg`.

---

## Authentication Realms

| Realm | Description |
|-------|-------------|
| `pam` | Local Linux PAM (system accounts) |
| `pve` | Built-in Proxmox VE user database |
| `ldap` | External LDAP server |
| `ad` | Microsoft Active Directory |
| `openid` | OpenID Connect (e.g., Keycloak, Authentik, Azure AD) |

Users are identified as `<username>@<realm>` (e.g., `admin@pve`, `root@pam`).

### Adding an LDAP Realm

In the web UI: **Datacenter** → **Permissions** → **Realms** → **Add** → **LDAP**.

Or via CLI:
```bash
pveum realm add myldap \
  --type ldap \
  --server ldap.example.com \
  --base_dn "ou=users,dc=example,dc=com" \
  --user_attr uid \
  --bind_dn "cn=proxmox,dc=example,dc=com" \
  --bind_password secretpass
```

### Adding an Active Directory Realm

```bash
pveum realm add myad \
  --type ad \
  --domain example.com \
  --server dc1.example.com
```

### Adding an OpenID Connect Realm

```bash
pveum realm add myoidc \
  --type openid \
  --issuer-url https://auth.example.com/realms/myrealm \
  --client-id proxmox \
  --client-key mysecretkey \
  --username-claim email
```

---

## User Management

```bash
# Create a user (built-in PVE realm)
pveum user add alice@pve --password secretpass --email alice@example.com --comment "Alice"

# Create a user (PAM realm — must also exist as Linux user)
useradd -m bob
pveum user add bob@pam

# List users
pveum user list

# Modify user
pveum user modify alice@pve --email newemail@example.com --enable 1

# Delete user
pveum user delete alice@pve
```

---

## Groups

```bash
# Create a group
pveum group add admins --comment "System Administrators"

# List groups
pveum group list

# Add user to group
pveum user modify alice@pve --groups admins

# Delete a group
pveum group delete admins
```

---

## Built-in Roles

| Role | Description |
|------|-------------|
| `Administrator` | Full access to everything |
| `PVEAdmin` | Most admin tasks, no realm/permission management |
| `PVEAuditor` | Read-only access |
| `PVEDatastoreAdmin` | Full storage management |
| `PVEDatastoreUser` | Allocate and use storage |
| `PVEPoolAdmin` | Manage resource pools |
| `PVEPoolUser` | Use objects in a pool |
| `PVESysAdmin` | Node-level system administration |
| `PVETemplateUser` | Clone templates |
| `PVEUserAdmin` | Manage users |
| `PVEVMAdmin` | Full VM/CT management |
| `PVEVMUser` | Use VMs/CTs (start, stop, console) |

---

## Custom Roles

```bash
# Create a custom role with specific privileges
pveum role add Operator --privs "VM.PowerMgmt VM.Console VM.Audit"

# List all privileges
pveum role list
pveum privs

# Modify role
pveum role modify Operator --privs "VM.PowerMgmt VM.Console VM.Audit Datastore.AllocateSpace"

# Delete role
pveum role delete Operator
```

---

## Assigning Permissions

Permissions are assigned as `(path, user/group, role)` triples.

### Permission Paths

| Path | Scope |
|------|-------|
| `/` | All resources |
| `/nodes/<node>` | Specific node |
| `/vms/<vmid>` | Specific VM or CT |
| `/storage/<storage>` | Specific storage |
| `/pool/<pool>` | Resource pool |
| `/access` | User/group/realm management |

```bash
# Grant role to user on entire cluster
pveum acl modify / --users alice@pve --roles PVEVMAdmin

# Grant role to group on specific node
pveum acl modify /nodes/pve1 --groups admins --roles Administrator

# Grant role on specific VM
pveum acl modify /vms/100 --users alice@pve --roles PVEVMUser

# Grant role on storage
pveum acl modify /storage/local-lvm --users alice@pve --roles PVEDatastoreUser

# List ACL rules
pveum acl list

# Propagate permission to child objects (default: propagate=1)
pveum acl modify / --users alice@pve --roles PVEVMAdmin --propagate 1
```

---

## API Tokens

API tokens allow scripts and automation tools to access the Proxmox API without user credentials. Tokens can be granted separate permission sets.

```bash
# Create an API token for a user
pveum user token add alice@pve mytoken --comment "Automation token"
# Returns token secret — save it immediately, it is shown only once

# List tokens
pveum user token list alice@pve

# Assign permission to a token
pveum acl modify /vms/100 \
  --tokens alice@pve!mytoken \
  --roles PVEVMAdmin

# Revoke a token
pveum user token remove alice@pve mytoken
```

API usage with a token:
```bash
curl -H "Authorization: PVEAPIToken=alice@pve!mytoken=<uuid-secret>" \
  https://pve:8006/api2/json/nodes
```

---

## Two-Factor Authentication (2FA)

Proxmox VE supports TOTP, WebAuthn (hardware keys), and recovery keys.

### Enable TOTP for a User

In the web UI: **Datacenter** → **Permissions** → **Two Factor** → **Add TOTP**.

Or via CLI:
```bash
pveum user tfa add alice@pve --type totp
```

### Require 2FA Realm-Wide

```bash
pveum realm modify pve --tfa type=totp
```

---

## Resource Pools

Pools group VMs, containers, and storage for easier permission delegation.

```bash
# Create a pool
pveum pool add devteam --comment "Development team resources"

# Add VM to pool
pveum pool modify devteam --vms 101,102

# Add storage to pool
pveum pool modify devteam --storage local-lvm

# Assign role on pool
pveum acl modify /pool/devteam --groups devs --roles PVEVMUser

# List pools
pveum pool list
```

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-pveum.html
- https://pve.proxmox.com/pve-docs/pveum.1.html
