---
name: unattended-upgrades
description: Configure and enable unattended-upgrades (automatic security/package updates) on Debian/Ubuntu hosts, including third-party repositories such as NetBird.
---

# Unattended-Upgrades Skill

## Overview

`unattended-upgrades` is a package that automatically installs security (and optionally other) updates on Debian/Ubuntu systems. This skill covers:

- Installing unattended-upgrades
- Enabling periodic runs via `20auto-upgrades`
- Adding third-party repositories to allowed origins using drop-in config files
- Verifying configuration with a dry run

---

## Installation

On Debian hosts where `unattended-upgrades` is not pre-installed:

```bash
apt-get install -y unattended-upgrades
```

Ubuntu 24.04 (noble) typically ships with it pre-installed.

---

## Configuration Approach

**Always use drop-in files** — never modify `/etc/apt/apt.conf.d/50unattended-upgrades` directly, as it is distribution-managed and may be overwritten by package upgrades.

Drop-in files go in `/etc/apt/apt.conf.d/` with a numeric prefix higher than 50, e.g. `51unattended-upgrades-<service>`.

---

## Adding a Third-Party Repository (e.g. NetBird)

### 1. Identify origin metadata

Inspect the repository's InRelease file to find the correct origin fields:

```bash
cat /var/lib/apt/lists/<encoded-repo-url>_dists_<suite>_InRelease | grep -E "^(Origin|Label|Suite|Codename):"
```

For NetBird (`pkgs.netbird.io`):
```
Origin: Artifactory
Label: Artifactory
Suite: stable
```

> **Important:** Do NOT use `origin=Artifactory` — it is too generic and would match any Artifactory-hosted repo. Use `site=pkgs.netbird.io` to target the specific host.

### 2. Create the drop-in file

```bash
cat > /etc/apt/apt.conf.d/51unattended-upgrades-netbird << 'EOF'
Unattended-Upgrade::Origins-Pattern {
    "site=pkgs.netbird.io";
};
EOF
```

This works on both Ubuntu (which uses `Allowed-Origins`) and Debian (which uses `Origins-Pattern`) — `Origins-Pattern` is the newer, more flexible format and coexists correctly with legacy `Allowed-Origins` blocks on Ubuntu.

---

## Enable Periodic Runs

Create `/etc/apt/apt.conf.d/20auto-upgrades` if not present:

```bash
cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF
```

Then enable and start the apt timers:

```bash
systemctl enable --now apt-daily.timer apt-daily-upgrade.timer
```

Check timer status:
```bash
systemctl status apt-daily.timer apt-daily-upgrade.timer
```

---

## Verification

Run a dry-run to confirm allowed origins and which packages would be upgraded:

```bash
unattended-upgrade --dry-run -d 2>&1 | grep -i "allowed origins\|<package-name>"
```

Expected output for NetBird:
```
Allowed origins are: ..., site=pkgs.netbird.io
Checking: netbird (...)
Checking: netbird-ui (...)
netbird
netbird-ui
```

---

## Gotchas and Known Issues

### Stale apt cache causes missed upgrades

`unattended-upgrade` only upgrades to the **apt candidate at runtime**. If `apt-get update`
hasn't been run since a new package version was published to the repository, the package will
not be upgraded even if a newer version is available in the repo.

**Symptom:** `unattended-upgrade -v` runs successfully but skips a package you expected to upgrade.

**Fix:** Run `apt-get update` explicitly before `unattended-upgrade`:

```bash
apt-get update && unattended-upgrade -v
```

The `APT::Periodic::Update-Package-Lists "1"` setting in `20auto-upgrades` schedules daily
updates, but the first time or if the timer hasn't fired yet, the cache may be stale.

---

### DNS circular dependency on NetBird routing peers (LXC containers)

**Problem:** If an LXC container resolves DNS exclusively through NetBird routing peers (e.g.,
`resolv.conf` points only to `10.0.0.7` / `10.0.0.8` reachable via NetBird routing), and
NetBird disconnects, DNS resolution fails — which prevents resolving the NetBird management
server — which prevents reconnecting → **deadlock**.

**Symptom:** After rebooting or NetBird restarting in the container, `apt-get update` fails
with "Temporary failure resolving..." and `netbird status` shows Disconnected.

**Fix:** Always add a **local DNS fallback** that is reachable without NetBird (e.g., the
LAN gateway). For an LXC container on Proxmox, use `pct set` to persist the nameservers:

```bash
# On the Proxmox host:
pct set <CTID> --nameserver "10.0.0.7 10.0.0.8 192.168.178.1"
```

Then immediately update the live container's resolv.conf:

```bash
# Append the fallback directly to the running container's resolv.conf
echo "nameserver 192.168.178.1" >> /proc/$(pct exec <CTID> -- sh -c 'echo $$')/root/1/etc/resolv.conf
# Or simpler:
pct exec <CTID> -- sh -c 'echo "nameserver 192.168.178.1" >> /etc/resolv.conf'
```

**Key rule:** Any NetBird routing peer that uses DNS via the overlay must have a local fallback
DNS that works without NetBird being connected.

---

### Proxmox LXC resolv.conf is managed by PVE

Proxmox regenerates `/etc/resolv.conf` inside LXC containers on restart from the PVE
container config. **Direct edits to `/etc/resolv.conf` inside the container are overwritten.**

To persist DNS changes on an LXC container:

```bash
# On the Proxmox host (not inside the container):
pct set <CTID> --nameserver "dns1 dns2 dns3"
```

The PVE block in `/etc/resolv.conf` is delimited by:
```
# --- BEGIN PVE ---
nameserver 10.0.0.7
nameserver 10.0.0.8
# --- END PVE ---
```

Lines appended below the PVE block survive restarts **only** if persisted via `pct set`.

---

### SSH host key changes after NetBird restart in LXC containers

After restarting the NetBird service in an LXC container, the SSH host key may change
(container overlay filesystem issue). Additionally, `/root/.ssh/authorized_keys` may
disappear.

**Symptom:** SSH connection fails with "REMOTE HOST IDENTIFICATION HAS CHANGED" or
"Permission denied (publickey)".

**Fix:**
1. Clear the old host key locally: `ssh-keygen -f ~/.ssh/known_hosts -R '<host-ip>'`
2. Re-add authorized_keys via `pct exec` from the Proxmox host:
   ```bash
   ssh root@<proxmox-host> "pct exec <CTID> -- bash -c 'mkdir -p /root/.ssh && echo \"<pubkey>\" >> /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys'"
   ```

---

### Stale apt cache — missed upgrades

`unattended-upgrade` only upgrades to the **apt candidate** at runtime — i.e. the version last
fetched by `apt-get update`. If the apt package lists are stale, the latest version is
never seen, and unattended-upgrades will report "package already at latest version" even
when a newer release is available.

**Symptom:** `apt-get install <pkg>` gets a newer version than `unattended-upgrade -d` offered.

**Fix:** Run `apt-get update` before running unattended-upgrades, or ensure that the
`APT::Periodic::Update-Package-Lists "1"` setting fires correctly (requires `apt-daily.timer`
to be active):

```bash
systemctl status apt-daily.timer
journalctl -u apt-daily.service --since "1 day ago"
```

---

### DNS circular dependency on Netbird routing peers

Routing peers that use DNS-only addresses (e.g. CoreDNS at `10.0.0.7`/`10.0.0.8` reachable
only via the Fritz!OS WireGuard tunnel) can deadlock: if Netbird disconnects and needs to
reconnect, it must resolve the management server FQDN — but DNS is unreachable without the
tunnel.

**Affected host:** nb1 (CT111 on pve2), which uses CoreDNS at `10.0.0.7`/`10.0.0.8` on the
Weißdornweg LAN, reachable via Fritz!Box WireGuard VPN.

**Fix:** Always add a fallback DNS server that is directly reachable without any tunnel.
For nb1: add `192.168.178.1` (Fritz!Box 7690) as third nameserver.

Persist in Proxmox (regenerated on restart — see Proxmox LXC DNS management below):
```bash
pct set 111 --nameserver "10.0.0.7 10.0.0.8 192.168.178.1"
```

---

### Proxmox LXC resolv.conf management

Proxmox VE **regenerates `/etc/resolv.conf`** inside LXC containers on every restart from
the container's node config (`/etc/pve/lxc/<CTID>.conf`). Edits made directly inside the
container will be lost on next `pct restart`.

**Persist DNS via pct set on the Proxmox host:**

```bash
pct set <CTID> --nameserver "ns1 ns2 ns3"
# Example for nb1:
pct set 111 --nameserver "10.0.0.7 10.0.0.8 192.168.178.1"
```

PVE writes a `# --- BEGIN PVE ---` block into `/etc/resolv.conf`. Extra nameservers can be
appended **below** that block inside the container for immediate effect without restart, but
the `pct set` command is authoritative for persistence.

---

## Notes

- **Debian 13 (trixie)**: `unattended-upgrades` is not pre-installed. Default allowed origins are:
  - `origin=Debian,codename=trixie,label=Debian`
  - `origin=Debian,codename=trixie,label=Debian-Security`
  - `origin=Debian,codename=trixie-security,label=Debian-Security`
- **Ubuntu 24.04 (noble)**: Pre-installed with ESM origins already configured.
- **Remote hosts via SSH**: Use `ssh root@<ip>` — Netbird IPs are used for connectivity.
- **aspire (local)**: Requires `sudo` for writes to `/etc/apt/apt.conf.d/`.
