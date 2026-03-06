---
name: storage
description: Proxmox VE storage configuration guide covering all supported storage backends including ZFS, Ceph RBD/CephFS, LVM, LVM-Thin, NFS, iSCSI, CIFS/SMB, and directory storage. Explains when to use each type, how to add storage via web UI and pvesm CLI, storage content types, and key best practices. Use when adding or managing storage in Proxmox VE, choosing between storage backends, or configuring shared or local storage for VMs and containers.
---
# Proxmox VE Storage

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE abstracts different storage backends through a unified storage framework. Storage pools are defined cluster-wide in `/etc/pve/storage.cfg`. Each storage pool can hold different content types:

| Content Type | Description |
|---|---|
| `images` | VM disk images |
| `rootdir` | Container rootfs |
| `iso` | ISO images |
| `backup` | vzdump backup archives |
| `snippets` | Cloud-init and hook scripts |
| `vztmpl` | Container templates |

---

## Storage Backends

### 1. Directory (local filesystem)

- Simplest storage type; uses a directory on any mounted filesystem
- Supports all content types
- Best for: homelabs, small setups, ISO/backup storage

```bash
pvesm add dir mydir --path /mnt/data --content iso,backup,vztmpl
```

Config example:
```
dir: local
    path /var/lib/vz
    content iso,vztmpl,backup
```

---

### 2. LVM (Logical Volume Manager)

- Block-level storage for VM disks
- Thick provisioning only (no snapshots without LVM-Thin)
- Best for: high-performance workloads, database VMs

```bash
# Create VG
pvcreate /dev/sdb
vgcreate myvg /dev/sdb

# Add to Proxmox
pvesm add lvm myvg --vgname myvg --content images
```

---

### 3. LVM-Thin

- Thin provisioning on top of LVM
- Supports snapshots and clones efficiently
- Best for: development VMs, desktops, snapshot-heavy workloads

```bash
# Create thin pool on existing VG
lvcreate -L 200G --thinpool data myvg

# Add to Proxmox
pvesm add lvmthin mythin --vgname myvg --thinpool data --content images
```

---

### 4. ZFS (local)

- Copy-on-write filesystem with built-in RAID, snapshots, compression, deduplication
- Best for: local reliable storage with strong data integrity
- Requires adequate RAM (≥1 GB per TB of storage; ECC recommended)
- Do **not** layer hardware RAID under ZFS

```bash
# Create ZFS pool (RAID-1 mirror)
zpool create mypool mirror /dev/sdc /dev/sdd

# Add to Proxmox
pvesm add zfspool myzfs --pool mypool --content images,rootdir

# ZFS management
zpool status
zfs list
zfs get compression mypool
zpool scrub mypool
```

Key ZFS properties to enable:
```bash
zfs set compression=lz4 mypool
zfs set atime=off mypool
```

---

### 5. NFS (Network File System)

- Shared storage accessible over the network
- Supports all content types (ISO, backup, images with qcow2)
- Best for: shared ISO/template repositories, VM image storage in small clusters

```bash
pvesm add nfs mynfs \
  --server 192.168.1.10 \
  --export /srv/nfs/proxmox \
  --content iso,backup,images
```

Config example:
```
nfs: mynfs
    path /mnt/pve/mynfs
    server 192.168.1.10
    export /srv/nfs/proxmox
    content iso,backup,images
```

---

### 6. iSCSI

- Block-level access to remote storage arrays (SAN)
- Often combined with LVM on top for VM disk management
- Best for: enterprise SAN connectivity

```bash
pvesm add iscsi myiscsi \
  --portal 192.168.1.20 \
  --target iqn.2023-01.com.example:storage \
  --content images
```

---

### 7. Ceph RBD (RADOS Block Device)

- Distributed block storage across a Ceph cluster
- Best for: highly available shared storage in multi-node clusters
- Requires at least 3 Ceph OSD nodes

```bash
# Ceph is configured via the web UI or pveceph
pveceph init --network 10.0.0.0/24
pveceph mon create
pveceph osd create /dev/sdb
pveceph pool create vmdata

# Add RBD storage
pvesm add rbd myrbd \
  --monhost 10.0.0.1,10.0.0.2,10.0.0.3 \
  --pool vmdata \
  --content images,rootdir
```

---

### 8. CephFS

- Distributed POSIX-compliant filesystem on top of Ceph
- Supports all content types
- Best for: shared file storage on Ceph clusters

```bash
pveceph fs create --name myfs --add-storage
# or manually:
pvesm add cephfs mycephfs \
  --monhost 10.0.0.1 \
  --path /mnt/pve/mycephfs \
  --content backup,iso,snippets
```

---

### 9. CIFS/SMB

- Access Windows shares or Samba servers
- Suitable for backup and ISO storage

```bash
pvesm add cifs mysmb \
  --server 192.168.1.30 \
  --share backups \
  --username user \
  --password pass \
  --content backup
```

---

## pvesm CLI Reference

```bash
# List all storage
pvesm status
pvesm list <storage>

# Add storage (see type-specific examples above)
pvesm add <type> <name> [options]

# Remove storage (does not delete data)
pvesm remove <name>

# Enable/disable storage
pvesm set <name> --disable 1
pvesm set <name> --disable 0

# Scan NFS exports
pvesm scan nfs <server>

# Scan iSCSI targets
pvesm scan iscsi <portal>
```

---

## Storage Comparison

| Backend | Shared | Snapshots | Thin Provisioning | Best Use Case |
|---------|--------|-----------|-------------------|---------------|
| Directory | No | qcow2 only | qcow2 only | ISOs, backups, small setups |
| LVM | No | No | No | High-performance local disks |
| LVM-Thin | No | Yes | Yes | Local snapshots, dev VMs |
| ZFS | No | Yes | Yes | Reliable local storage |
| NFS | Yes | qcow2 only | qcow2 only | Shared files, ISOs |
| iSCSI | Yes | No | No | SAN block storage |
| Ceph RBD | Yes | Yes | Yes | HA cluster storage |
| CephFS | Yes | Yes | No | Shared file storage |
| CIFS | Yes | No | No | SMB/Windows shares |

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-pvesm.html
- https://pve.proxmox.com/pve-docs/pvesm.1.html
