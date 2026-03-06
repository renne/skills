---
name: cluster-networking
description: Proxmox VE cluster management, Software-Defined Networking (SDN), and firewall configuration. Covers creating and joining clusters with pvecm, Corosync quorum, live migration, SDN zones and VNets, Linux bridges, VLANs, bonds, and the distributed PVE firewall with rules, IP sets, and security groups. Use when setting up a Proxmox cluster, configuring network infrastructure for VMs/containers, or managing firewall rules at the datacenter, node, or VM level.
---
# Proxmox VE Cluster Manager, Networking, and Firewall

Source: https://pve.proxmox.com/pve-docs/

## Cluster Manager

### Overview

A Proxmox VE cluster enables centralized management of multiple nodes via a single web UI, live VM/container migration between nodes, shared storage, and High Availability (HA). Cluster communication uses **Corosync** for reliable low-latency messaging and **pmxcfs** (Proxmox Cluster Filesystem) to distribute configuration in real time.

### Requirements

- All nodes must have synchronized clocks (use NTP)
- Dedicated low-latency network link for Corosync traffic (recommended)
- UDP ports 5405–5412 open between nodes
- Root SSH access between nodes for initial setup

### Creating a Cluster

```bash
# On the first node (becomes the initial cluster master)
pvecm create mycluster

# Check cluster status
pvecm status
```

### Joining a Node

```bash
# On each additional node:
pvecm add <existing-node-ip>

# Verify all nodes appear
pvecm nodes
pvecm status
```

### Removing a Node

```bash
# From the remaining cluster node
pvecm delnode <nodename>
```

### Cluster Configuration File

`/etc/pve/corosync.conf` — managed automatically; do not edit manually unless recovering.

### Quorum

- Requires a majority (quorum) of nodes to be online to prevent split-brain
- With 2 nodes, add a **QDevice** (lightweight third vote) to maintain quorum

```bash
# Add external QDevice (on a separate small machine)
pvecm qdevice setup <qdevice-host-ip>
```

---

## Live Migration

```bash
# Migrate VM online (requires shared storage or storage migration)
qm migrate <vmid> <target-node> --online

# Migrate container
pct migrate <ctid> <target-node> --online

# Offline migration with local storage
qm migrate <vmid> <target-node> --with-local-disks
```

---

## Networking

### Network Configuration File

All interface definitions reside in `/etc/network/interfaces`. Changes made in the web UI are written here and applied with `ifreload -a`.

### Linux Bridge

Default bridge connecting VMs/CTs to physical network:
```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

### VLAN-Aware Bridge

Allows passing VLAN tags to VMs without creating separate bridges per VLAN:
```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

### Bond (NIC Teaming)

```
auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2
    bond-miimon 100
    bond-mode active-backup

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
```

Bond modes: `active-backup`, `balance-rr`, `balance-xor`, `802.3ad` (LACP), `balance-tlb`, `balance-alb`.

### Multiple Networks

Best practice for production clusters:
- **vmbr0** — management and VM traffic
- **corosync network** — dedicated Corosync cluster link
- **storage network** — dedicated Ceph or iSCSI traffic

---

## Software-Defined Networking (SDN)

SDN extends the networking capabilities with VXLANs, VRFs, and BGP routing across nodes.

### SDN Components

- **Zone** — Defines the SDN technology (Simple, VLAN, QinQ, VXLAN, EVPN)
- **VNet** — A virtual network attached to a zone
- **Subnet** — IP range assigned to a VNet (for IPAM)

### Simple Zone Example

```bash
# Create a simple zone (equivalent to a standard bridge across nodes)
# Via web UI: Datacenter → SDN → Zones → Add → Simple
# Name: localzone, nodes: all

# Create a VNet in that zone
# Datacenter → SDN → VNets → Add
# Name: vnet10, Zone: localzone, Tag: 10

# Apply SDN config
pvesh create /cluster/sdn
```

VMs and CTs can then use `vnet10` as their bridge interface.

---

## Firewall

### Overview

The Proxmox VE firewall is a distributed, cluster-wide firewall implemented using nftables/iptables. Rules are stored in pmxcfs and propagated to all nodes automatically.

### Hierarchy

```
Datacenter firewall  (/etc/pve/firewall/cluster.fw)
  └─ Node firewall   (/etc/pve/nodes/<node>/host.fw)
       └─ VM/CT firewall  (/etc/pve/firewall/<VMID>.fw)
```

Higher-level rules can restrict what lower levels allow.

### Enabling the Firewall

In the web UI:
- **Datacenter** → Firewall → Options → Firewall: **Enable**
- **Node** → Firewall → Options → Firewall: **Enable**
- **VM/CT** → Firewall → Options → Firewall: **Enable** (and enable on each NIC)

### Firewall Rule Syntax

Firewall files use INI-style sections:

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
IN ACCEPT -source 192.168.1.0/24 -dport 22 -proto tcp -log info   # Allow SSH from LAN
IN ACCEPT -dport 8006 -proto tcp                                    # Allow web UI
IN DROP -log nolog                                                  # Drop everything else
```

### IP Sets

Define reusable groups of IP addresses:

```ini
[IPSET management]
192.168.1.0/24
10.0.0.5
```

Reference in rules: `-source +management`

### Security Groups

Reusable rule sets applied to multiple VMs:

```ini
[group webservers]
IN ACCEPT -dport 80 -proto tcp
IN ACCEPT -dport 443 -proto tcp
```

Apply with: `GROUP webservers -i net0`

### Common Datacenter Firewall Configuration

```ini
[OPTIONS]
enable: 1

[ALIASES]
management_net 192.168.1.0/24

[IPSET management]
192.168.1.0/24

[RULES]
IN ACCEPT -source +management -dport 8006 -proto tcp    # Web UI
IN ACCEPT -source +management -dport 22 -proto tcp      # SSH
IN ACCEPT -proto icmp                                   # ICMP/ping
```

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-pvecm.html
- https://pve.proxmox.com/pve-docs-8/chapter-sdn.html
- https://pve.proxmox.com/pve-docs-8/chapter-pve-firewall.html
