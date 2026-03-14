---
name: deployment
description: Deploy and configure Watchtower for automatic Docker container updates via Docker Compose. Covers label-enable mode, scheduling, Docker API version compatibility fix, and how to label existing stacks.
---
# Watchtower Deployment

Watchtower automatically updates running Docker containers when new images are available.

## Docker Compose Setup

Minimal `compose.yml`:

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: Europe/Berlin
      WATCHTOWER_LABEL_ENABLE: "true"
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_SCHEDULE: "0 0 4 * * *"
      DOCKER_API_VERSION: "1.40"
```

**No network is needed** — Watchtower only requires the Docker socket.

## Key Environment Variables

| Variable | Purpose |
|---|---|
| `WATCHTOWER_LABEL_ENABLE` | `"true"` — only update containers with `com.centurylinklabs.watchtower.enable=true` label |
| `WATCHTOWER_CLEANUP` | `"true"` — remove old images after updating |
| `WATCHTOWER_SCHEDULE` | 6-field cron (seconds first): `0 0 4 * * *` = daily at 04:00 |
| `DOCKER_API_VERSION` | Set to `"1.40"` to fix crash-loop on modern Docker hosts (see below) |
| `TZ` | Timezone for schedule evaluation (e.g., `Europe/Berlin`) |

## Docker API Version Mismatch (Critical Fix)

**Symptom:** Watchtower crash-loops immediately with:
```
Error response from daemon: client version 1.25 is too old. Minimum supported API version is 1.40
```

**Cause:** Modern Docker servers (API ≥ 1.54) dropped backward compatibility for client API < 1.40. Watchtower's internal Docker client negotiates API 1.25 by default.

**Fix:** Add to the `environment` section:
```yaml
DOCKER_API_VERSION: "1.40"
```

This overrides the negotiated version to one the server accepts.

## Label-Enable Mode

When `WATCHTOWER_LABEL_ENABLE=true`, only containers with this label are monitored:

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

Or in YAML dict form:
```yaml
labels:
  com.centurylinklabs.watchtower.enable: "true"
```

Add this label to **every service** in every stack you want updated.

### Labeling Existing Stacks

For each target stack, add the label to every service in its `compose.yml`:

```yaml
services:
  myapp:
    image: myapp:latest
    labels:
      com.centurylinklabs.watchtower.enable: "true"
```

Then redeploy the stack to recreate containers with the new label:
```bash
cd /srv/docker/mystack && docker compose up -d
```

### Important Notes

- **Locally-built images** (`build: .`): Watchtower cannot pull updates from a registry, but will skip/warn gracefully. Labeling is harmless.
- **YAML anchors**: If a service uses `<<: *anchor` merge and `labels: []` override, replace `labels: []` with the watchtower label.
- **AIO containers** (e.g., Nextcloud AIO): AIO has its own internal update mechanism. Adding our label to `nextcloud-aio-mastercontainer` is safe and won't conflict.

## Schedule Format

Watchtower uses a **6-field cron** (seconds-first, unlike standard 5-field cron):

```
┌─────────── second (0-59)
│ ┌───────── minute (0-59)
│ │ ┌─────── hour (0-23)
│ │ │ ┌───── day of month (1-31)
│ │ │ │ ┌─── month (1-12)
│ │ │ │ │ ┌─ day of week (0-6, 0=Sunday)
│ │ │ │ │ │
0 0 4 * * *   ← daily at 04:00
```

## Verification

After deploying, confirm healthy operation:
```bash
docker logs watchtower 2>&1 | tail -20
```

Expected output:
```
level=info msg="Watchtower 1.7.1"
level=info msg="Only checking containers using enable label"
level=info msg="Scheduling first run: 2026-03-14 04:00:00 +0100 CET"
```

The line `"Only checking containers using enable label"` confirms label-enable mode is active.

## References

- [Watchtower Documentation](https://containrrr.dev/watchtower/)
- [Watchtower Arguments Reference](https://containrrr.dev/watchtower/arguments/)
- [Watchtower Container Labels](https://containrrr.dev/watchtower/container-selection/)
- [GitHub: containrrr/watchtower](https://github.com/containrrr/watchtower)
