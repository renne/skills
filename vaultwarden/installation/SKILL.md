---
name: installation
description: Vaultwarden installation and deployment guide covering Docker container images (tags, registries, multi-arch), starting a container with Docker or Podman, using Docker Compose with Caddy or nginx, and customizing container startup. Use when deploying Vaultwarden for the first time, choosing a container image, setting up persistent storage, or configuring Docker Compose for Vaultwarden.
---
# Vaultwarden Installation & Deployment

Sources: https://github.com/dani-garcia/vaultwarden/wiki/Which-container-image-to-use and https://github.com/dani-garcia/vaultwarden/wiki/Starting-a-Container and https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose

## Overview

Vaultwarden is an unofficial Bitwarden server implementation written in Rust, ideal for self-hosted deployments. The recommended deployment method is via Docker or Podman using the official `vaultwarden/server` container image.

---

## Container Image

### Registries

The official image is published on three registries:

| Registry | Image |
|----------|-------|
| Docker Hub | `docker.io/vaultwarden/server` |
| GitHub Container Registry | `ghcr.io/dani-garcia/vaultwarden` |
| Quay | `quay.io/vaultwarden/server` |

### Tags

| Tag | Description |
|-----|-------------|
| `latest` | Latest stable release (recommended for most users) |
| `testing` | Tracks the latest commits; newest features but less stable |
| `x.y.z` (e.g. `1.30.0`) | Specific released version |
| `latest-alpine` | Same as `latest` but Alpine-based (slimmer image) |
| `x.y.z-alpine` | Specific version, Alpine-based |

The image is **multi-arch**: pulling `vaultwarden/server` automatically selects the correct image for your architecture (x86_64, arm64, armv7, armv6). For ARMv6 (Raspberry Pi 1/Zero), Docker 20.10.0+ is required.

---

## Starting a Container

### Docker

```bash
mkdir /vw-data

docker run -d --name vaultwarden \
  -v /vw-data/:/data/ \
  -p 80:80 \
  vaultwarden/server:latest
```

To bind to a specific IP (e.g., on a LAN):

```bash
docker run -d --name vaultwarden \
  -v /vw-data/:/data/ \
  -p 192.168.0.2:80:80 \
  vaultwarden/server:latest
```

### Podman (non-root)

Non-root containers cannot use privileged ports (<1024), so use port 8080:

```bash
podman run -d --name vaultwarden \
  -v /vw-data/:/data/:Z \
  -e ROCKET_PORT=8080 \
  -p 8080:8080 \
  vaultwarden/server:latest
```

### SELinux (RHEL/Fedora)

Set the correct context before mounting:

```bash
semanage fcontext -a -t svirt_sandbox_file_t '/vw-data(/.*)?'
restorecon -Rv /vw-data
```

### Starting / Stopping

```bash
# Stop
docker stop vaultwarden

# Start again (don't use docker run a second time)
docker start vaultwarden
```

---

## Docker Compose

### Minimal Setup (no reverse proxy)

```yaml
# compose.yml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      SIGNUPS_ALLOWED: "true"   # Set to "false" after creating your account
    volumes:
      - ./vw-data:/data
    ports:
      - 11001:80
```

```bash
docker compose up -d && docker compose logs -f
# Update:
docker compose pull && docker compose up -d && docker compose logs -f
```

### With Caddy (automatic HTTPS via HTTP challenge)

```yaml
# compose.yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      DOMAIN: "https://vaultwarden.example.com"
      SIGNUPS_ALLOWED: "true"
    volumes:
      - ./vw-data:/data

  caddy:
    image: caddy:2
    container_name: caddy
    restart: always
    ports:
      - 80:80
      - 443:443
      - 443:443/udp   # HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy-config:/config
      - ./caddy-data:/data
    environment:
      DOMAIN: "https://vaultwarden.example.com"
      EMAIL: "admin@example.com"
      LOG_FILE: "/data/access.log"
```

`Caddyfile` (no modifications needed):

```
{$DOMAIN} {
  log {
    level INFO
    output file {$LOG_FILE} {
      roll_size 10MB
      roll_keep 10
    }
  }
  tls {$EMAIL}
  encode zstd gzip
  reverse_proxy vaultwarden:80 {
    header_up X-Real-IP {remote_host}
  }
}
```

---

## Customizing Container Startup

Mount a script as `/etc/vaultwarden.sh` (single script) or a directory of `.sh` files as `/etc/vaultwarden.d/`:

```bash
docker run -d --name vaultwarden \
  -v $(pwd)/init.sh:/etc/vaultwarden.sh \
  <other args...> \
  vaultwarden/server:latest
```

Scripts run on **every** container start, so make them idempotent:

```bash
if [ ! -e /.init ]; then
  touch /.init
  # run one-time init steps here
fi
```

---

## Keeping Vaultwarden Up to Date

Vaultwarden must be kept up to date because the official Bitwarden mobile and browser clients auto-update and can introduce breaking changes. Incompatible client/server versions may cause sudden breakage.

```bash
docker compose pull
docker compose up -d
```

---

## References

- [Which container image to use](https://github.com/dani-garcia/vaultwarden/wiki/Which-container-image-to-use)
- [Starting a Container](https://github.com/dani-garcia/vaultwarden/wiki/Starting-a-Container)
- [Using Docker Compose](https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose)
- [Proxy examples](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples)
