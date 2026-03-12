---
name: containers
description: Proxmox VE LXC container management using the pct CLI and web UI. Covers creating, configuring, starting, stopping, cloning, snapshotting, and managing Linux containers, as well as rootfs, network, mount points, resource limits, and template management. Use when creating or managing LXC containers on Proxmox VE, configuring container resources, or scripting container lifecycle with the pct command.
---
# Proxmox VE LXC Containers

Source: https://pve.proxmox.com/pve-docs/

## Overview

Proxmox VE uses LXC (Linux Containers) for lightweight OS-level virtualization. Containers share the host kernel but have isolated filesystems, processes, and networking. They start faster and consume fewer resources than full VMs.

Container configurations are stored in `/etc/pve/lxc/<CTID>.conf`.

**Limitations:**
- Linux guests only (no Windows containers)
- No custom kernel modules by default
- Nested virtualization (Docker-in-LXC) requires the `nesting` feature

---

## Container Templates

Download templates using `pveam`:

```bash
# Update the template list
pveam update

# List available templates
pveam available --section system

# Download a template
pveam download local ubuntu-22.04-standard_22.04-1_amd64.tar.zst

# List downloaded templates
pveam list local
```

---

## Creating a Container

### Via Web UI

1. Click **Create CT** in the top-right corner.
2. Fill in CT ID, hostname, password/SSH key.
3. Select template, storage, network, and resource limits.
4. Click **Finish**.

### Via CLI (`pct`)

```bash
pct create 200 \
  local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname mycontainer \
  --storage local-lvm \
  --rootfs 8 \
  --memory 1024 \
  --swap 512 \
  --cores 2 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --password secretpass \
  --unprivileged 1
```

---

## pct Command Reference

```bash
# Start / Stop / Reboot / Destroy
pct start <ctid>
pct stop <ctid>
pct shutdown <ctid>
pct reboot <ctid>
pct destroy <ctid>

# Status and info
pct status <ctid>
pct config <ctid>
pct list

# Modify configuration
pct set <ctid> --memory 2048
pct set <ctid> --cores 4
pct set <ctid> --hostname newname

# Enter container shell
pct enter <ctid>

# Execute command inside container
pct exec <ctid> -- apt update

# Push/pull files
pct push <ctid> /local/path /container/path
pct pull <ctid> /container/path /local/path

# Resize rootfs
pct resize <ctid> rootfs 20G

# Clone
pct clone <ctid> <newid> --hostname newct

# Snapshot
pct snapshot <ctid> snap1 --description "Before update"
pct rollback <ctid> snap1
pct delsnapshot <ctid> snap1
pct listsnapshot <ctid>

# Mount container filesystem (for recovery)
pct mount <ctid>
pct unmount <ctid>
```

---

## Container Configuration File (`/etc/pve/lxc/<CTID>.conf`)

```ini
arch: amd64
cores: 2
hostname: mycontainer
memory: 1024
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:AA:BB:CC,ip=192.168.1.200/24,gw=192.168.1.1,type=veth
ostype: ubuntu
rootfs: local-lvm:vm-200-disk-0,size=8G
swap: 512
unprivileged: 1
features: nesting=1
```

### Key Configuration Options

| Option | Description |
|--------|-------------|
| `memory` | RAM in MiB |
| `swap` | Swap in MiB |
| `cores` | CPU core limit |
| `cpulimit` | Max CPU usage (e.g., `2.0` = 2 cores max) |
| `rootfs` | Root filesystem device and size |
| `netX` | Network interface |
| `mpX` | Additional mount points |
| `unprivileged` | `1` for unprivileged (safer, recommended) |
| `features` | Comma-separated feature flags |
| `ostype` | OS hint for template-specific behavior |

---

## Networking

```bash
# Static IP
pct set <ctid> --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1

# DHCP
pct set <ctid> --net0 name=eth0,bridge=vmbr0,ip=dhcp

# Multiple interfaces
pct set <ctid> --net1 name=eth1,bridge=vmbr1,ip=10.0.0.5/24

# VLAN tag
pct set <ctid> --net0 name=eth0,bridge=vmbr0,tag=100,ip=dhcp

# Rate limit (MB/s)
pct set <ctid> --net0 name=eth0,bridge=vmbr0,ip=dhcp,rate=100
```

---

## Mount Points

Bind-mount host directories or add extra volumes:

```bash
# Add extra volume
pct set <ctid> --mp0 local-lvm:10,mp=/data

# Bind-mount host directory (privileged containers only)
pct set <ctid> --mp0 /host/path,mp=/container/path

# Bind-mount read-only (e.g. shared config)
pct set <ctid> --mp0 /host/path,mp=/container/path,ro=1
```

In config file:
```
mp0: local-lvm:vm-200-disk-1,mp=/data,size=10G

# Bind-mount: host path directly, no storage volume
mp0: /opt/coredns,mp=/etc/coredns,ro=1
```

**Shared config across multiple containers:** Bind-mount a host directory read-only into several containers. Apps with a `reload` directive (e.g. CoreDNS `reload 2s`) will pick up changes automatically via inotify — no restart or sync mechanism needed.

**Important:** Bind mounts require the container to be **stopped** before editing `/etc/pve/lxc/<ctid>.conf`. Adding the `mp` line while the container is running takes effect only after a restart.

---

## Features

Enable special capabilities in the container config:

```
features: nesting=1,keyctl=1,fuse=1
```

| Feature | Use Case |
|---------|----------|
| `nesting=1` | Docker-in-LXC, nested containers |
| `keyctl=1` | Required for some apps (e.g., Keybase) |
| `fuse=1` | FUSE filesystem support |
| `mknod=1` | Allow mknod in unprivileged containers |

### TUN Device Passthrough (WireGuard / NetBird)

To run **NetBird** or any WireGuard-based VPN inside an LXC container, the container needs access to `/dev/net/tun` and the `NET_ADMIN` capability. Add these lines to `/etc/pve/lxc/<ctid>.conf`:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Without these, `netbird up` fails with `Failed to create WireGuard interface` or similar errors.

---

## Storage Notes

**`local` storage on Proxmox does NOT support container rootfs directories** — it only supports ISO images and templates. Use `local-lvm` (LVM-thin) or `local-zfs` (ZFS) for container rootfs:

```bash
# ZFS-backed rootfs (use on ZFS hosts)
pct create 107 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --rootfs local-zfs:2 \
  --hostname dns1 ...

# Templates are still referenced from local: even when rootfs is on local-zfs
```

**Template storage vs rootfs storage are independent** — you can download templates to `local:vztmpl/` and create the rootfs on `local-zfs` or `local-lvm`.

---

## Resource Limits

```bash
# CPU limit (cores used at most)
pct set <ctid> --cpulimit 2

# CPU units (relative weight)
pct set <ctid> --cpuunits 512

# Memory and swap
pct set <ctid> --memory 2048 --swap 1024
```

---

## Privileged vs. Unprivileged Containers

| | Privileged | Unprivileged |
|-|------------|--------------|
| UID mapping | Root = host root | UIDs mapped (root = UID 100000 on host) |
| Security | Lower | Higher (recommended) |
| Feature support | Full | Most features |

Set at creation time:
```bash
pct create <ctid> <template> --unprivileged 1
```

---

## DNS Management in LXC Containers

Proxmox VE **regenerates `/etc/resolv.conf`** inside every LXC container on restart from the
container's node configuration. Edits made directly inside the container are **lost on next
`pct restart`**.

### Persisting DNS settings

Use `pct set` on the Proxmox host to persist nameservers and search domains:

```bash
# Set nameservers (space-separated, applied in order):
pct set <CTID> --nameserver "10.0.0.7 10.0.0.8 192.168.178.1"

# Set search domain (optional):
pct set <CTID> --searchdomain "local"

# View current config:
grep -E "^nameserver|^searchdomain" /etc/pve/lxc/<CTID>.conf
```

PVE writes a `# --- BEGIN PVE ---` / `# --- END PVE ---` block into `/etc/resolv.conf`
inside the container. Additional lines outside that block can be edited directly for
immediate effect, but the PVE block is authoritative and will be re-written on restart.

### Always include a fallback DNS

For containers whose primary DNS (e.g. `10.0.0.7`/`10.0.0.8` via a WireGuard tunnel) may be
unreachable during startup or tunnel reconnect, always add a locally-reachable fallback:

```bash
# For nb1 on pve2 (Friedensstraße):
pct set 111 --nameserver "10.0.0.7 10.0.0.8 192.168.178.1"
```

Without a fallback, the container may fail to resolve hostnames on startup, breaking
services like Netbird that need DNS to reconnect to the management server.

---

## References

- https://pve.proxmox.com/pve-docs/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- https://pve.proxmox.com/pve-docs-8/chapter-pct.html
- https://pve.proxmox.com/pve-docs/pct.1.html
