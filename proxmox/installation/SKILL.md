---
name: installation
description: Proxmox VE installation and host system administration guide covering ISO setup, initial configuration, package repositories, networking bridges, disk management, and system updates. Use when installing Proxmox VE on bare metal, configuring the host OS after installation, managing Debian packages, setting up network interfaces or bridges, or performing host-level administration tasks.
---
# Proxmox VE Installation and Host System Administration

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE (Virtual Environment) is an open-source server virtualization platform based on Debian GNU/Linux. It integrates KVM hypervisor and LXC containers, a web-based management interface, REST API, and built-in cluster, storage, and backup tools.

The web interface is accessible at `https://<host-ip>:8006` after installation.

---

## Hardware Requirements

- 64-bit CPU with Intel VT-x or AMD-V virtualization extensions enabled in BIOS/UEFI
- Minimum 2 GB RAM (8 GB or more recommended for production)
- At least one NIC
- Fast storage recommended (SSD strongly preferred for ZFS)
- ECC RAM recommended for ZFS workloads

---

## Installation Steps

1. Download the latest Proxmox VE ISO from https://www.proxmox.com/en/downloads.
2. Write the ISO to a USB drive using `dd`, Rufus, or Etcher.
3. Boot from the USB and follow the graphical installer:
   - Accept the EULA.
   - Select the target hard disk.
   - Choose country, timezone, and keyboard layout.
   - Set the root password and an administrator email address.
   - Configure the management network (IP address, netmask, gateway, DNS server).
4. After installation, access the web UI at `https://<ip>:8006`.

---

## Post-Installation: Package Repositories

Proxmox VE ships with enterprise repositories enabled by default. For non-subscribed (free/home lab) installations, switch to the no-subscription repository:

```bash
# Disable enterprise repo (comment out)
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" \
  > /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
```

---

## Networking

### Network Bridges

Proxmox VE uses Linux bridges (`vmbr*`) to connect VMs and containers to the network. The installer creates `vmbr0` by default.

**Configuration file:** `/etc/network/interfaces`

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

Apply changes:
```bash
ifreload -a
# or
systemctl restart networking
```

### VLAN-Aware Bridge

Enable VLAN awareness on a bridge to pass tagged traffic to VMs:

In the web UI: Node → Network → Edit bridge → check "VLAN aware".

Or in `/etc/network/interfaces`:
```
iface vmbr0 inet static
    ...
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

### Bonding (NIC Teaming)

Create a bond for redundancy or higher throughput:
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

---

## Disk Management

### Viewing Disks

Navigate to Node → Disks in the web UI, or use:
```bash
lsblk
pvdisks
```

### S.M.A.R.T. Monitoring

```bash
smartctl -a /dev/sda
```

Enable S.M.A.R.T. monitoring per-disk in Node → Disks → S.M.A.R.T.

---

## System Updates

```bash
apt update && apt full-upgrade -y
pveam update   # update container template list
```

---

## Key Configuration Files

| File/Path | Purpose |
|-----------|---------|
| `/etc/network/interfaces` | Network interface configuration |
| `/etc/apt/sources.list` | Debian package sources |
| `/etc/apt/sources.list.d/pve-*.list` | Proxmox VE package sources |
| `/etc/pve/` | Proxmox cluster configuration (pmxcfs) |
| `/var/log/pveproxy/` | Web UI proxy logs |

---

## Common CLI Tools

| Command | Description |
|---------|-------------|
| `pvesh` | Shell interface for the PVE REST API |
| `pvecm` | Cluster manager CLI |
| `pvesm` | Storage manager CLI |
| `pvenode` | Node management CLI |
| `pveupdate` | Check for updates |

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-sysadmin.html
