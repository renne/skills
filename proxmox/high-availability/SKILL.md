---
name: high-availability
description: Proxmox VE High Availability (HA) cluster setup and management. Covers HA manager, fencing, resource agents, HA groups, configuring HA for VMs and containers, testing failover, and the ha-manager CLI. Use when configuring automatic VM/container restart on node failure, setting up HA resources and groups, understanding Proxmox HA fencing mechanisms, or troubleshooting HA cluster behavior.
---
# Proxmox VE High Availability (HA)

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE High Availability automatically restarts VMs and containers on a different node when their host fails. HA is built on:

- **Proxmox Cluster** (Corosync + pmxcfs) — required; all nodes must be in a cluster
- **HA Manager** (`pve-ha-manager`) — monitors resources and manages failover
- **Fencing** — ensures failed nodes cannot corrupt shared data (STONITH)
- **Shared storage** — required for VMs/CTs to be migratable across nodes

**Minimum requirement:** 3 nodes (or 2 nodes + external QDevice for quorum).

---

## How HA Works

1. The **HA Manager** runs as a distributed service across all cluster nodes.
2. If a node becomes unresponsive, the HA Manager:
   a. Waits for the `watchdog` timeout.
   b. **Fences** (powers off) the failed node using configured fence agents.
   c. Starts the HA-enabled VMs/CTs on a surviving node.

The **watchdog** timer is refreshed by the HA manager daemon. If it cannot be refreshed (e.g., node isolated), the node self-reboots (self-fencing).

---

## Fencing

Fencing prevents a failed node from accessing shared storage and causing data corruption. Proxmox uses **watchdog-based self-fencing** by default.

For more reliable fencing, configure external fence agents (e.g., IPMI, ACPI, power switches):

```bash
# In the web UI: Datacenter → HA → Resources → Fencing
# Or edit /etc/pve/ha/manager_status
```

Supported fence types include `agent:fence_ipmilan`, `agent:fence_virsh`, and many STONITH agents from the `fence-agents-*` packages.

---

## Configuring HA for a VM or Container

### Via Web UI

1. Select the VM or CT → **More** → **Manage HA**.
2. Set:
   - **Request state**: `started` (default)
   - **Max restart count**: attempts before marking as error
   - **Max relocate count**: times to try migrating before restarting in place
   - **Group**: optional HA group for node preferences

### Via CLI (`ha-manager`)

```bash
# Enable HA for a VM
ha-manager add vm:<vmid>

# Enable HA for a container
ha-manager add ct:<ctid>

# Set requested state
ha-manager set vm:<vmid> --state started

# Disable HA for a resource (but keep config)
ha-manager set vm:<vmid> --state disabled

# Remove from HA management
ha-manager remove vm:<vmid>

# List all HA resources
ha-manager status

# Detailed status
ha-manager status --verbose
```

---

## HA Resource States

| State | Meaning |
|-------|---------|
| `started` | Should be running; HA will start/restart it |
| `stopped` | Should be stopped; HA will stop it if running |
| `disabled` | HA does not manage this resource |
| `ignored` | Completely ignored by HA manager |
| `error` | Max restarts exceeded; manual intervention required |

To recover from error state:
```bash
ha-manager set vm:<vmid> --state started
```

---

## HA Groups

Groups allow you to control node preferences and constraints for HA resources.

```bash
# Create an HA group
ha-manager groupadd mygroup \
  --nodes pve1:2,pve2:1,pve3:1 \
  --restricted 1 \
  --nofailback 0

# List groups
ha-manager grouplist

# Assign VM to a group
ha-manager set vm:<vmid> --group mygroup
```

### Group Options

| Option | Description |
|--------|-------------|
| `nodes` | Comma-separated list of `node:priority` pairs (higher = preferred) |
| `restricted` | `1`: Only run on nodes in the group; `0`: group nodes preferred but not required |
| `nofailback` | `1`: Do not migrate back when preferred node recovers |

---

## HA Manager Services

```bash
# Status of HA services
systemctl status pve-ha-lrm    # Local Resource Manager (manages local VMs/CTs)
systemctl status pve-ha-crm    # Cluster Resource Manager (runs on master node)

# View HA logs
journalctl -u pve-ha-lrm -f
journalctl -u pve-ha-crm -f
```

---

## Maintenance Mode

Before performing maintenance on a node (updates, reboots):

```bash
# Put node in maintenance mode (migrates HA resources away)
ha-manager crm-command node-maintenance enable <node>

# Resume normal operation
ha-manager crm-command node-maintenance disable <node>
```

Or in the web UI: Node → **More** → **Maintenance Mode**.

---

## Testing HA Failover

```bash
# Simulate a node failure (dangerous — only in test environments)
# Option 1: Abruptly power off the node
# Option 2: Isolate node from cluster network and observe

# Manually trigger HA migrate
ha-manager crm-command migrate vm:<vmid> <target-node>

# Manually trigger HA relocate (graceful)
ha-manager crm-command relocate vm:<vmid> <target-node>
```

---

## HA with Shared Storage

HA requires VMs/CTs to be stored on shared storage so they can start on any node:

- Ceph RBD or CephFS (recommended for hyper-converged setups)
- NFS or CIFS
- iSCSI with multipath

Local storage cannot be used for HA-enabled VMs/CTs.

---

## Key Configuration Files

| File | Description |
|------|-------------|
| `/etc/pve/ha/resources.cfg` | HA resource definitions |
| `/etc/pve/ha/groups.cfg` | HA group definitions |
| `/etc/pve/ha/manager_status` | HA manager state (do not edit manually) |

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-ha-manager.html
- https://pve.proxmox.com/pve-docs/ha-manager.8.html
