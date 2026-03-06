---
name: virtual-machines
description: Proxmox VE QEMU/KVM virtual machine management using the qm CLI and web UI. Covers creating, configuring, starting, stopping, cloning, snapshotting, and migrating VMs, as well as disk, network, CPU, memory, PCI passthrough, and cloud-init configuration. Use when creating or managing KVM virtual machines on Proxmox VE, configuring VM hardware options, or scripting VM lifecycle operations with the qm command.
---
# Proxmox VE QEMU/KVM Virtual Machines

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE uses QEMU/KVM to provide full hardware virtualization. Each VM is identified by a numeric ID (VMID). VM configurations are stored in `/etc/pve/qemu-server/<VMID>.conf`.

---

## Creating a VM

### Via Web UI

1. Click **Create VM** in the top-right corner.
2. Fill in General (Node, VMID, Name), OS type, System (BIOS/UEFI, machine type), Disk, CPU, Memory, Network.
3. Click **Finish**.

### Via CLI (`qm`)

```bash
qm create 100 \
  --name myvm \
  --memory 4096 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:32 \
  --cdrom local:iso/ubuntu-22.04.iso \
  --boot order=scsi0 \
  --ostype l26
```

---

## qm Command Reference

```bash
# Start / Stop / Reboot
qm start <vmid>
qm stop <vmid>
qm shutdown <vmid>
qm reboot <vmid>
qm reset <vmid>

# Status and info
qm status <vmid>
qm config <vmid>
qm list

# Modify configuration
qm set <vmid> --memory 8192
qm set <vmid> --cores 4
qm set <vmid> --net0 virtio,bridge=vmbr0

# Remove a VM (with all disks)
qm destroy <vmid> --purge

# Clone
qm clone <vmid> <newid> --name newvm --full

# Snapshot
qm snapshot <vmid> snap1 --description "Before update"
qm rollback <vmid> snap1
qm delsnapshot <vmid> snap1
qm listsnapshot <vmid>

# Console
qm terminal <vmid>
qm monitor <vmid>

# Disk operations
qm disk import <vmid> disk.qcow2 local-lvm
qm resize <vmid> scsi0 +10G

# Live migration
qm migrate <vmid> <target-node>
qm migrate <vmid> <target-node> --online
```

---

## VM Configuration File (`/etc/pve/qemu-server/<VMID>.conf`)

```ini
agent: 1
bios: ovmf
boot: order=scsi0;net0
cores: 4
cpu: host
memory: 8192
name: myvm
net0: virtio=DE:AD:BE:EF:CA:FE,bridge=vmbr0,firewall=1
numa: 0
ostype: l26
scsi0: local-lvm:vm-100-disk-0,size=32G
scsihw: virtio-scsi-pci
sockets: 1
```

### Key Configuration Options

| Option | Description |
|--------|-------------|
| `memory` | RAM in MiB |
| `cores` | CPU cores per socket |
| `sockets` | Number of CPU sockets |
| `cpu` | CPU type (e.g., `host`, `kvm64`, `x86-64-v2-AES`) |
| `bios` | `seabios` (default) or `ovmf` (UEFI) |
| `machine` | QEMU machine type (e.g., `q35`, `i440fx`) |
| `ostype` | OS optimization hint (`l26`, `win10`, `win11`, etc.) |
| `scsiX` | SCSI disk device |
| `netX` | Network interface |
| `agent` | Enable QEMU Guest Agent (`1` to enable) |
| `balloon` | Enable memory ballooning |
| `numa` | Enable NUMA topology |
| `args` | Pass raw QEMU arguments |

---

## CPU Configuration

```bash
# Use host CPU type for best performance
qm set <vmid> --cpu host

# Enable CPU flags
qm set <vmid> --cpu x86-64-v2-AES

# Limit CPU usage (percent of one core)
qm set <vmid> --cpulimit 2.0

# CPU pinning (pin to specific host cores)
qm set <vmid> --affinity 0-3
```

---

## Disk Configuration

### Disk Interfaces

- `scsi` with `virtio-scsi-pci` controller — recommended for Linux guests
- `virtio` (virtio-blk) — fast, for performance-critical workloads
- `ide` — for compatibility (legacy)
- `sata` — for Windows guests without VirtIO drivers

### Disk Options

```bash
# Add disk (32 GiB on local-lvm storage)
qm set <vmid> --scsi1 local-lvm:32

# Resize
qm resize <vmid> scsi0 +20G

# Enable SSD emulation and discard (TRIM)
qm set <vmid> --scsi0 local-lvm:vm-100-disk-0,ssd=1,discard=on

# Import external image
qm disk import <vmid> /tmp/disk.qcow2 local-lvm
```

---

## Network Configuration

```bash
# VirtIO NIC on vmbr0 (recommended for Linux)
qm set <vmid> --net0 virtio,bridge=vmbr0

# VLAN tag
qm set <vmid> --net0 virtio,bridge=vmbr0,tag=10

# Rate limit (MB/s)
qm set <vmid> --net0 virtio,bridge=vmbr0,rate=100

# Firewall enabled on NIC
qm set <vmid> --net0 virtio,bridge=vmbr0,firewall=1
```

---

## Cloud-Init

Cloud-init enables automated first-boot configuration for cloud images.

```bash
# Add a cloud-init drive
qm set <vmid> --ide2 local-lvm:cloudinit

# Set cloud-init options
qm set <vmid> --ciuser ubuntu --cipassword secret
qm set <vmid> --sshkeys /path/to/keys.pub
qm set <vmid> --ipconfig0 ip=dhcp
qm set <vmid> --ipconfig0 ip=192.168.1.50/24,gw=192.168.1.1

# Regenerate cloud-init image
qm cloudinit dump <vmid> user
```

---

## PCI / GPU Passthrough

```bash
# Find PCI device IDs
lspci -v

# Pass PCI device to VM (add to config)
# hostpci0: 01:00.0,pcie=1
qm set <vmid> --hostpci0 01:00.0,pcie=1

# Full GPU passthrough (disable vga)
qm set <vmid> --vga none
```

---

## QEMU Guest Agent

Install `qemu-guest-agent` inside the VM for graceful shutdown, IP reporting, and file-system freeze during snapshots:

```bash
# Inside Debian/Ubuntu guest
apt install qemu-guest-agent
systemctl enable --now qemu-guest-agent

# In Proxmox config
qm set <vmid> --agent 1
```

---

## Templates

```bash
# Convert VM to template (irreversible)
qm template <vmid>

# Clone from template
qm clone <template-vmid> <new-vmid> --name newvm --full
```

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-qm.html
- https://pve.proxmox.com/pve-docs/qm.1.html
