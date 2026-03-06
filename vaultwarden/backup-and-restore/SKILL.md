---
name: backup-and-restore
description: Vaultwarden backup and restore guide covering the data folder structure, SQLite database backup using sqlite3 or the built-in backup command, backing up attachments and sends, config.json and RSA key considerations, restore procedure, and automated backup examples. Use when setting up automated backups for Vaultwarden, restoring from a backup, understanding what data to back up, or integrating with cloud storage tools like rclone or restic.
---
# Vaultwarden Backup & Restore

Sources: https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault

## Overview

Back up Vaultwarden data regularly via an automated process (e.g., cron job). Store at least one copy remotely (cloud storage or a separate machine). Consider encrypting backups, especially if they include `config.json` (which may contain the admin token and SMTP credentials).

---

## Data Folder Structure (SQLite backend)

By default, all data lives under the `data/` directory (configurable via `DATA_FOLDER`):

```
data/
├── attachments/            # File attachments — BACKUP REQUIRED
│   └── <uuid>/
│       └── <random_id>
├── config.json             # Admin page config — BACKUP RECOMMENDED
├── db.sqlite3              # Main SQLite database — BACKUP REQUIRED
├── db.sqlite3-shm          # SQLite shared memory (not always present, skip)
├── db.sqlite3-wal          # SQLite write-ahead log (include if present)
├── icon_cache/             # Cached website icons — BACKUP OPTIONAL
├── rsa_key.der             # RSA keys for JWT signing — BACKUP RECOMMENDED
├── rsa_key.pem
├── rsa_key.pub.der
└── sends/                  # Send file attachments — BACKUP OPTIONAL
    └── <uuid>/
        └── <random_id>
```

For **MySQL/PostgreSQL backends**: same directory structure (no SQLite files), but you must also dump the database separately.

---

## Backing Up the SQLite Database

### Using the built-in backup command (recommended, since v1.32.1)

```bash
# Standalone binary
/vaultwarden backup

# Docker
docker exec -it vaultwarden /vaultwarden backup
```

### Using `sqlite3`

The `.backup` command uses the [SQLite Online Backup API](https://www.sqlite.org/backup.html) — safe to use while Vaultwarden is running:

```bash
sqlite3 data/db.sqlite3 ".backup '/path/to/backups/db-$(date '+%Y%m%d-%H%M').sqlite3'"
```

Or with `VACUUM INTO` (compacts empty space, takes longer):

```bash
sqlite3 data/db.sqlite3 "VACUUM INTO '/path/to/backups/db-$(date '+%Y%m%d-%H%M').sqlite3'"
```

Example: running on January 1, 2021 at 12:34pm creates `db-20210101-1234.sqlite3`.

---

## What to Back Up

| Item | Priority | Notes |
|------|----------|-------|
| `db.sqlite3` | **Required** | Main database with all vault data |
| `db.sqlite3-wal` | **Required** (if present) | Must be backed up together with `db.sqlite3` |
| `attachments/` | **Required** | File attachments not stored in DB |
| `config.json` | Recommended | Admin panel config; may contain sensitive data |
| `rsa_key.*` | Recommended | Deleting these logs out all users and invalidates invitations |
| `sends/` | Optional | Ephemeral Send attachments |
| `icon_cache/` | Optional | Re-fetched automatically if missing |

---

## Restore Procedure

1. **Stop Vaultwarden:**
   ```bash
   docker stop vaultwarden
   # or: docker compose down
   ```

2. **Replace files** in the `data/` directory with backup versions.

3. **For `.backup`/`VACUUM INTO` backups:** Delete any existing `db.sqlite3-wal` file before restoring, to avoid database corruption.

4. **For copy-based backups:** Restore `db.sqlite3` and `db.sqlite3-wal` as a matching pair.

5. **Restart Vaultwarden:**
   ```bash
   docker start vaultwarden
   # or: docker compose up -d
   ```

> **Test restores regularly** to verify your backup process works before you actually need it.

---

## Automated Backup (Cron + Docker Example)

> Docker images do not include `sqlite3` or `cron`. Install these on the Docker host and run backup jobs outside the container.

**Backup script (`/root/backup-vaultwarden.sh`):**

```bash
#!/bin/bash
docker compose -f /opt/vaultwarden/compose.yaml down
datestamp=$(date +%Y%m%d-%H%M)
backup_dir="/home/user/vw-backups"
zip -9 -r "${backup_dir}/${datestamp}.zip" /opt/vw-data/
# Optional: copy to remote
# scp -i ~/.ssh/id_rsa "${backup_dir}/${datestamp}.zip" user@remote:~/vw-backups/
docker compose -f /opt/vaultwarden/compose.yaml up -d
```

**Cron job (daily at midnight):**

```cron
0 0 * * * /root/backup-vaultwarden.sh
```

**Cleanup script (keep only last 7 days):**

```bash
#!/bin/bash
find /home/user/vw-backups/ -type f -name '*.zip' -mtime +7 -delete
```

---

## Cloud Storage / Incremental Backups

- [**rclone**](https://rclone.org/) — sync backup files to S3, Google Drive, Backblaze B2, and many others
- [**restic**](https://restic.net/) — encrypted, deduplicated backups; efficient for large attachment collections
- [**rustic**](https://rustic.cli.rs/) — Rust-based restic alternative

---

## Third-Party Backup Tools

| Tool | Description |
|------|-------------|
| [vaultwarden-backup](https://github.com/ttionya/vaultwarden-backup) | Docker-based backup with rclone |
| [bitwarden_rs-local-backup](https://github.com/shivpatel/bitwarden_rs-local-backup) | Local backup script |
| [vaultwarden-backup (GitLab)](https://gitlab.com/1O/vaultwarden-backup) | Comprehensive backup solution |

---

## References

- [Backing up your vault](https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault)
- [Changing persistent data location](https://github.com/dani-garcia/vaultwarden/wiki/Changing-persistent-data-location)
- [SQLite Online Backup API](https://www.sqlite.org/backup.html)
