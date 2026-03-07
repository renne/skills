---
name: access-control
description: Matter Access Control Lists (ACLs) covering privileges, authentication modes, subjects, targets, and managing ACLs with chip-tool. Use when implementing Matter device access control, commissioning multi-admin setups, granting fabric access to controllers, or troubleshooting Matter permission errors.
---
# Matter Access Control

Source: https://project-chip.github.io/connectedhomeip-doc/guides/access-control-guide.html

## Overview

All Interaction Model operations in Matter must be verified by the Access Control mechanism. This includes reading/subscribing to attributes or events, writing attributes, and invoking commands.

If a client lacks sufficient privilege, the operation is denied with status `0x7E Access Denied`.

---

## Initial Commissioning Privileges

During commissioning, the commissioner gets an implicit `Administer` privilege over the PASE channel. As part of commissioning, when the commissioner invokes `AddNOC`, it provides a `CaseAdminNode` argument. This automatically installs an ACL granting `Administer` privilege to the specified node (or CAT).

After commissioning, administrators manage ACLs by reading/writing the fabric-scoped `ACL` attribute of the **Access Control Cluster on endpoint 0**.

---

## Access Control List (ACL) Fields

Each ACL entry contains:

| Field | Description |
|-------|-------------|
| `Privilege` | Level of access granted |
| `AuthMode` | Authentication mode (CASE or Group) |
| `Subjects` | List of node IDs or group IDs |
| `Targets` | List of cluster/endpoint/device-type targets |

### Privilege Field

| Privilege | Value | Description |
|-----------|-------|-------------|
| `View` | 1 | Read attributes and events (default minimum) |
| `Operate` | 3 | Write attributes and invoke commands (default for writes) |
| `Manage` | 4 | Includes Operate + management operations |
| `Administer` | 5 | Full control, including ACL management |

> **Note:** Privileges are cumulative — `Manage` also grants `Operate` and `View`; `Administer` is the highest privilege and grants all others.

### AuthMode Field

| AuthMode | Value | Description |
|----------|-------|-------------|
| `CASE` | 2 | Certificate-Authenticated Session (unicast) |
| `Group` | 3 | Group messaging (multicast) |

### Subjects Field

- For `CASE` AuthMode: list of **Node IDs** (or CATs — Compressed Attribute Tags)
- For `Group` AuthMode: list of **Group IDs**
- Empty list = applies to any subject with the given AuthMode

### Targets Field

Each target entry has `cluster`, `endpoint`, and `deviceType` (mutually exclusive: use either `endpoint` or `deviceType`):

```json
{"cluster": 6, "endpoint": null, "deviceType": null}      // on/off cluster on any endpoint
{"cluster": null, "endpoint": 1, "deviceType": null}      // any cluster on endpoint 1
{"cluster": 8, "endpoint": 2, "deviceType": null}         // level control on endpoint 2
```

Empty targets list = applies to all targets on the node.

---

## Limitations (Minimum Requirements)

Per the Matter specification, implementations must support at least:
- 3 ACL entries per fabric
- 4 subjects per ACL entry
- 3 targets per ACL entry

Configure limits in `CHIPConfig.h` or `CHIPProjectAppConfig.h`:
```cpp
CHIP_CONFIG_EXAMPLE_ACCESS_CONTROL_MAX_ENTRIES_PER_FABRIC
CHIP_CONFIG_EXAMPLE_ACCESS_CONTROL_MAX_SUBJECTS_PER_ENTRY
CHIP_CONFIG_EXAMPLE_ACCESS_CONTROL_MAX_TARGETS_PER_ENTRY
```

---

## Use Case Examples

### Single Administrator

Commissioner assigns itself as the sole administrator:
```json
{
  "privilege": 5,
  "authMode": 2,
  "subjects": [112233],
  "targets": []
}
```

### Multiple Administrators (using CAT)

Commissioner provides a CAT (Compressed Attribute Tag) so multiple controllers share admin rights:
```json
{
  "privilege": 5,
  "authMode": 2,
  "subjects": ["0xFFFFFFFD00010001"],
  "targets": []
}
```

### Non-Administrative Controllers (View Only)

```json
{
  "privilege": 1,
  "authMode": 2,
  "subjects": [4444, 5555, 6666],
  "targets": []
}
```

### Group Messaging

```json
{
  "privilege": 3,
  "authMode": 3,
  "subjects": [123, 456],
  "targets": [
    {"cluster": 6, "endpoint": null, "deviceType": null},
    {"cluster": null, "endpoint": 1, "deviceType": null},
    {"cluster": 8, "endpoint": 2, "deviceType": null}
  ]
}
```

---

## Managing ACLs with chip-tool

### Important Notes

1. **Entire list must be written** — list operations for single entries (append, update, delete) are not supported; the entire ACL list must be written at once.
2. **Always include an admin ACL first** — if writing a new ACL list, ensure the first entry grants `Administer` to yourself to avoid losing access mid-write.
3. **All fields required** — even null fields must be explicitly provided.
4. **Fabric index** — required in the write payload but is ignored (the fabric index is determined by the session).

### Enum Reference

| Privilege | Value |
|-----------|-------|
| View | 1 |
| Operate | 3 |
| Manage | 4 |
| Administer | 5 |

| AuthMode | Value |
|----------|-------|
| CASE | 2 |
| Group | 3 |

| Common Clusters | ID |
|-----------------|-----|
| On/Off | 6 |
| Level Control | 8 |
| Descriptor | 29 |
| Binding | 30 |
| Access Control | 31 |
| Basic Information | 40 |

### Read Current ACLs

```bash
chip-tool accesscontrol read acl <node-id> 0
```

Example output for a freshly commissioned device:
```
Endpoint: 0 Cluster: 0x0000_001F Attribute 0x0000_0000
  ACL: 1 entry
  [1]: {
    Privilege: 5 (Administer), AuthMode: 2 (CASE),
    Subjects: [112233], Targets: []
  }
```

### Write a New ACL List

Write a 2-entry ACL: admin for node 112233, view for nodes 4444 and 5555:

```bash
chip-tool accesscontrol write acl '[
  {"fabricIndex": 1, "privilege": 5, "authMode": 2, "subjects": [112233], "targets": []},
  {"fabricIndex": 1, "privilege": 1, "authMode": 2, "subjects": [4444, 5555], "targets": []}
]' <node-id> 0
```

### Write ACL with Targeted Access

Grant `Operate` privilege to node 7777 for the On/Off cluster only:

```bash
chip-tool accesscontrol write acl '[
  {"fabricIndex": 1, "privilege": 5, "authMode": 2, "subjects": [112233], "targets": []},
  {"fabricIndex": 1, "privilege": 3, "authMode": 2, "subjects": [7777],
   "targets": [{"cluster": 6, "endpoint": null, "deviceType": null}]}
]' <node-id> 0
```

---

## Common Errors

| Error | Cause |
|-------|-------|
| `0x7E Access Denied` | Client lacks required privilege for the operation |
| ACL write drops admin access | First ACL entry didn't grant `Administer` to self |
| Write partially applied | ACL list writes can use multiple messages; ACLs take effect as processed |

---

## References

- [Access Control Guide](https://project-chip.github.io/connectedhomeip-doc/guides/access-control-guide.html)
- [Matter Specification – Access Control Cluster](https://csa-iot.org/developer-resource/specifications-download-request/)
- [chip-tool Documentation](https://github.com/project-chip/connectedhomeip/tree/master/examples/chip-tool)
