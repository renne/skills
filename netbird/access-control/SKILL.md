---
name: access-control
description: NetBird access control covering groups, access policies (source/destination, protocols, ports), posture checks (OS version, client version, geolocation, process validation), and Zero Trust network segmentation. Use when configuring which peers can communicate with each other, restricting access by device compliance, or setting up least-privilege network policies in NetBird.
---
# NetBird Access Control

Source: https://docs.netbird.io/manage/access-control

## Overview

NetBird follows a Zero Trust model: **no peer has access to any other peer by default**. Access is granted exclusively through explicit **Access Policies** that reference **Groups**. Optionally, **Posture Checks** ensure devices meet compliance requirements before policies take effect.

---

## Groups

Groups act as tags that organize peers for policy assignment. A peer can belong to multiple groups; a group can contain multiple peers.

### User Groups vs Peer Groups

While all groups are technically identical in NetBird, there is a useful conceptual distinction:

- **User Groups**: Assigned to user accounts in the dashboard or synced from an IdP. When a user logs in via SSO (e.g., Okta, Azure AD), their devices automatically inherit the groups linked to that user account.
- **Peer Groups**: Assigned directly to infrastructure (servers, databases, IoT devices) — typically via a Setup Key with Auto-groups configured. The group is persistent on the peer regardless of who (if anyone) logged in.

This pattern enables clean Zero Trust segmentation: `Engineers` (user group) can access `Production-Servers` (peer group) via an explicit policy, without exposing those servers to all peers.

### Built-in Group

- **All** – contains every peer in the account. Cannot be modified or deleted. Used for the default onboarding policy.

### Creating a Group

In the dashboard: **Access Control → Groups → Add Group**. Type the group name and press Enter.

### Assigning Peers to Groups

Peers can be added to groups:
- **Manually**: Dashboard → Peers → select peer → Groups tab.
- **Automatically at enrollment**: via a Setup Key with **Auto-groups** set to the desired group(s).
- **Via SSO/IdP provisioning**: groups synced from the identity provider are mapped to NetBird groups.

---

## Access Policies

Policies define which source group(s) can reach which destination group(s), and optionally restrict by protocol and port.

### Default Policy

New accounts include a default **All-to-All** policy (source: `All` → destination: `All`) so every peer can reach every other peer. This policy exists for convenience during onboarding but **undermines Zero Trust principles** — it grants blanket access to all devices. For any serious deployment, **delete this policy** immediately after onboarding and replace it with targeted policies.

### Creating a Policy

Dashboard → **Access Control → Policies → Add Policy**:

1. **Name** – descriptive label.
2. **Source Groups** – groups whose peers initiate connections.
3. **Destination Groups** – groups whose peers receive connections.
4. **Bidirectional** – when enabled, the destination can also initiate connections back to the source. When disabled, traffic flows one-way only.
5. **Protocol** – `All`, `TCP`, `UDP`, `ICMP`.
6. **Ports / Port ranges** – restrict to specific ports (e.g., `22`, `80,443`, `8000-9000`).
7. **Posture Checks** – optional compliance checks that must pass before the policy applies.

### Example Policies

**Allow engineers to SSH into servers:**
```
Source:      Engineers
Destination: Servers
Protocol:    TCP
Ports:       22
Direction:   Unidirectional
```

**Allow all peers to ping each other:**
```
Source:      All
Destination: All
Protocol:    ICMP
Direction:   Bidirectional
```

**Allow web tier to reach database tier:**
```
Source:      WebServers
Destination: DatabaseServers
Protocol:    TCP
Ports:       5432
Direction:   Unidirectional
```

### Policy Evaluation

- Policies are distributed by the management server to all relevant peers.
- A connection is allowed if **at least one** policy permits it.
- New peers added to a group immediately inherit all policies that reference that group.

---

## Posture Checks

Posture checks are compliance verifications attached to policies. A peer must pass **all** posture checks in a policy before the policy grants it access.

Manage in the dashboard: **Access Control → Posture Checks → Add Check**.

### Available Check Types

#### NetBird Client Version

Require peers to run a minimum client version:

```
Minimum version: 0.28.0
```

Peers with older clients are blocked until they upgrade.

#### Operating System Version

Restrict access to peers running approved OS versions:

| OS | Example minimum version |
|----|------------------------|
| Linux kernel | `6.1` |
| macOS | `13.0` (Ventura) |
| Windows | `10.0.19041` (20H1) |
| iOS | `16.0` |
| Android | `11` |

#### Geolocation

Allow access only from approved countries or regions using ISO 3166-1 alpha-2 country codes:

```
Allowed countries: US, DE, GB
```

#### Peer Network Range

Restrict access based on the peer's real IP address range:

```
Allowed ranges: 192.168.1.0/24, 10.0.0.0/8
```

#### Process Check

Verify that a required process is running (e.g., security agent) or that a forbidden process is absent:

```
Required processes:
  - /usr/bin/crowdsec-agent      (Linux)
  - C:\Program Files\CrowdSec\crowdsec.exe  (Windows)
```

### Attaching Posture Checks to a Policy

When creating or editing a policy, select one or more posture checks under the **Posture Checks** field. Peers that fail any attached check cannot use that policy.

---

## Control Center

Dashboard → **Control Center** provides a topological visualization of all peers, groups, and active policies. Use it to:

- See which peers can communicate.
- Quickly identify misconfigured or overly permissive policies.
- Click through to a policy to edit it.

---

## Zero Trust Best Practices

1. Delete the default **All-to-All** policy after onboarding.
2. Create a group per role or function (e.g., `Engineers`, `Servers`, `IoT`).
3. Use **Bidirectional: off** for policies where only one side should initiate.
4. Apply posture checks to sensitive destination groups to enforce endpoint compliance.
5. Use **Ephemeral peers** on setup keys for temporary workloads that should auto-remove on disconnect.

---

## References

- [Understanding Groups and Access Policies](https://docs.netbird.io/manage/access-control)
- [Managing Access with Groups and Policies](https://docs.netbird.io/manage/access-control/manage-network-access)
- [Posture Checks](https://docs.netbird.io/manage/access-control/posture-checks)
- [Control Center](https://docs.netbird.io/manage/control-center)
