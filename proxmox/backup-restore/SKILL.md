---
name: backup-restore
description: Proxmox VE backup and restore guide covering vzdump backup tool, backup modes (snapshot, suspend, stop), scheduled backup jobs, Proxmox Backup Server (PBS) integration, compression options, retention policies, and restore procedures for VMs and containers. Use when setting up VM or container backups in Proxmox VE, configuring backup schedules and retention, integrating Proxmox Backup Server, or restoring from backup archives.
---
# Proxmox VE Backup and Restore

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE uses **vzdump** to create backups of VMs and containers. Backups include both the disk data and the configuration file. Backup archives can be stored on any configured storage that supports the `backup` content type.

For advanced deduplicated incremental backups, **Proxmox Backup Server (PBS)** is the recommended target.

---

## Backup Modes

| Mode | Description | Downtime | Consistency |
|------|-------------|----------|-------------|
| `snapshot` | Live backup using storage snapshots | Minimal (freeze only) | Good (with QEMU Guest Agent) |
| `suspend` | Pause VM/CT during backup | Short | Better |
| `stop` | Shut down completely, backup, restart | Full restart time | Best |

**Recommendation:** Use `snapshot` mode for production VMs. Enable the QEMU Guest Agent for filesystem-consistent snapshots.

---

## vzdump CLI

### Backup a VM or Container

```bash
# Snapshot mode backup to local storage
vzdump <vmid> --mode snapshot --storage local --compress zstd

# Backup a container with stop mode
vzdump <ctid> --mode stop --storage local --compress gzip

# Backup all VMs and CTs on the node
vzdump --all --mode snapshot --storage local --compress zstd

# Backup to a specific directory
vzdump <vmid> --dumpdir /mnt/backup --compress zstd

# Bandwidth limit (KiB/s)
vzdump <vmid> --storage local --bwlimit 51200

# Exclude specific VMs
vzdump --all --exclude 101,102 --storage local

# Send email notification on completion
vzdump <vmid> --storage local --mailto admin@example.com
```

### Compression Options

| Value | Format | Speed vs. Ratio |
|-------|--------|-----------------|
| `0` or `none` | No compression | Fastest, largest |
| `lzo` | LZO | Fast, moderate compression |
| `gzip` | gzip | Moderate speed and ratio |
| `zstd` | Zstandard | Best balance (recommended) |

---

## vzdump Configuration File

Default settings: `/etc/vzdump.conf`

```ini
# /etc/vzdump.conf
bwlimit: 0
ionice: 7
lockwait: 180
stopwait: 10
storage: local
tmpdir: /var/tmp
compress: zstd
mailto: admin@example.com
mailnotification: always
mode: snapshot
```

---

## Scheduled Backup Jobs (Web UI)

1. Go to **Datacenter** → **Backup** → **Add**.
2. Configure:
   - **Node** (or all nodes)
   - **Storage** target
   - **Schedule** (cron-like schedule, e.g., `Mon..Fri 03:00`)
   - **Selection**: All VMs, Pool, or specific VMIDs
   - **Mode**: snapshot / suspend / stop
   - **Compression**: zstd recommended
   - **Retention** settings (max backups per VM, or PBS-managed)
3. Click **Create**.

Scheduled backup jobs are stored in `/etc/pve/jobs.cfg`.

---

## Proxmox Backup Server (PBS) Integration

PBS provides deduplication, incremental backups, and fine-grained retention management.

### Add PBS as Storage

In the web UI: **Datacenter** → **Storage** → **Add** → **Proxmox Backup Server**.

Or via CLI:
```bash
pvesm add pbs mypbs \
  --server 192.168.1.50 \
  --datastore mystore \
  --username backup@pbs \
  --password mypassword \
  --fingerprint <PBS-server-fingerprint>
```

Retrieve PBS fingerprint:
```bash
# On the PBS host
proxmox-backup-manager cert info | grep Fingerprint
```

### Backup to PBS

```bash
vzdump <vmid> --storage mypbs --mode snapshot --compress zstd
```

### PBS Retention Policies

Configured in the PBS datastore or in the Proxmox VE backup job:
- Keep last N backups
- Keep daily/weekly/monthly/yearly backups

---

## Restore

### Restore via Web UI

1. Navigate to the storage containing the backup: **Node** → **Storage** → **Backups**.
2. Select the backup archive.
3. Click **Restore**.
4. Choose target node, storage, and optionally a new VMID.

### Restore via CLI

```bash
# Restore a VM from a vzdump archive
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2024_01_01-03_00_00.vma.zst 100

# Restore to a different VMID
qmrestore /path/to/backup.vma.zst 200 --storage local-lvm

# Restore a container
pct restore 101 /var/lib/vz/dump/vzdump-lxc-101-2024_01_01-03_00_00.tar.zst \
  --storage local-lvm

# Restore from PBS storage
qmrestore pbs:backup/vm/100/2024-01-01T03:00:00Z 100 --storage local-lvm
```

### Force Restore (overwrite existing VM)

```bash
qmrestore /path/to/backup.vma.zst 100 --force
```

---

## Backup Archive Naming Convention

```
vzdump-<type>-<vmid>-<YYYY_MM_DD>-<HH_MM_SS>.<ext>
```

Examples:
- `vzdump-qemu-100-2024_01_01-03_00_00.vma.zst` — VM 100, zstd compressed
- `vzdump-lxc-200-2024_01_01-03_00_00.tar.zst` — CT 200, zstd compressed
- `vzdump-qemu-100-2024_01_01-03_00_00.vma.lzo` — VM 100, LZO compressed

---

## Host Configuration Backup

VM/CT data backups do not include host configuration. Back up `/etc/pve` separately:

```bash
# Backup entire Proxmox configuration
tar czf /mnt/backup/pve-config-$(date +%Y%m%d).tar.gz /etc/pve

# Backup crontab and other host-specific files
tar czf /mnt/backup/host-config-$(date +%Y%m%d).tar.gz \
  /etc/pve /etc/network/interfaces /etc/hostname /etc/hosts
```

---

## Verify Backups

```bash
# Verify a VMA backup
vma verify /var/lib/vz/dump/vzdump-qemu-100-2024_01_01-03_00_00.vma.zst

# Extract config from backup (without restoring)
vma config /path/to/backup.vma
```

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-vzdump.html
- https://pve.proxmox.com/pve-docs/vzdump.1.html
