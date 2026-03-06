# Docker Compose File Reference

Source: https://docs.docker.com/reference/compose-file/

## Overview

The Compose Specification is the latest and recommended version of the Compose file format. The `compose.yaml` file configures your application's services, networks, volumes, configs, and secrets.

**Preferred file names** (in order of precedence):
1. `compose.yaml`
2. `compose.yml`
3. `docker-compose.yaml`
4. `docker-compose.yml`

## Top-Level Elements

| Element | Description |
|---------|-------------|
| `name` | Project name (overrides directory-based default) |
| `services` | Container definitions (required) |
| `networks` | Named networks shared across services |
| `volumes` | Named persistent volumes shared across services |
| `configs` | Non-sensitive runtime configuration data |
| `secrets` | Sensitive runtime configuration data |
| `include` | Include other Compose files as sub-projects |

### Project name

```yaml
name: myapp

services:
  web:
    image: nginx
```

---

## `services`

A service is an abstract definition of a container. Each key under `services` is the service name; the value is the service configuration.

### `image`

Specifies the image to start the container from.
```yaml
services:
  web:
    image: nginx:alpine
```

### `build`

Configuration for building the image from source.
```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.dev
      target: builder        # multi-stage build target
      args:
        NODE_ENV: development
```

Short form: `build: .` (uses `Dockerfile` in the current directory).

### `command`

Overrides the default command from the image.
```yaml
services:
  app:
    image: python:3.12
    command: flask run --debug
    # or as a list:
    command: ["flask", "run", "--debug"]
```

### `entrypoint`

Overrides the default entrypoint from the image.

### `ports`

Expose container ports to the host. Format: `HOST:CONTAINER`.
```yaml
services:
  web:
    ports:
      - "8080:80"        # host 8080 → container 80
      - "443:443"
      - "127.0.0.1:3000:3000"  # bind to localhost only
```

### `expose`

Expose ports to other services on the same network (not to the host).
```yaml
services:
  api:
    expose:
      - "3000"
```

### `environment`

Set environment variables inside the container.
```yaml
services:
  db:
    image: postgres
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    # or as a list:
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secret
      - DEBUG   # value taken from shell / .env file
```

> **Security**: Do not use `environment` for sensitive data. Use `secrets` instead.

### `env_file`

Load environment variables from a file.
```yaml
services:
  app:
    env_file:
      - .env
      - .env.local
    # or with optional flag (Compose >= 2.24.0):
    env_file:
      - path: .env.optional
        required: false
```

### `volumes`

Mount host paths or named volumes into the container.
```yaml
services:
  db:
    volumes:
      - db-data:/var/lib/postgresql/data   # named volume
      - ./config:/etc/config:ro            # bind mount (read-only)
      - /tmp/cache:/cache                  # absolute host path
```

Long form syntax:
```yaml
services:
  app:
    volumes:
      - type: volume
        source: app-data
        target: /data
      - type: bind
        source: ./src
        target: /app/src
        read_only: true
```

### `networks`

Connect a service to one or more named networks.
```yaml
services:
  app:
    networks:
      - front-tier
      - back-tier

networks:
  front-tier:
  back-tier:
```

### `depends_on`

Control startup and shutdown order.

Short form (starts dependencies first, does not wait for healthy):
```yaml
services:
  web:
    depends_on:
      - db
      - redis
```

Long form (wait for specific conditions):
```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy
        restart: true
      redis:
        condition: service_started
```

Conditions:
- `service_started` — dependency container has started (default)
- `service_healthy` — dependency passes its `healthcheck`
- `service_completed_successfully` — dependency exited with code 0

### `healthcheck`

Configure a check that runs to determine whether a container is healthy.
```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

### `restart`

Restart policy for the container.

| Value | Description |
|-------|-------------|
| `no` | Never restart (default) |
| `always` | Always restart |
| `on-failure` | Restart only on non-zero exit |
| `unless-stopped` | Restart unless explicitly stopped |

```yaml
services:
  web:
    restart: unless-stopped
```

### `container_name`

Assign a custom container name. Note: disables scaling beyond one instance.
```yaml
services:
  web:
    container_name: my-web-container
```

### `profiles`

Assign the service to one or more profiles. Services without `profiles` are always started. Services with `profiles` are only started when that profile is active.
```yaml
services:
  app:
    image: myapp
  debug-tools:
    image: busybox
    profiles:
      - debug
```

### `secrets`

Grant access to secrets (mounted at `/run/secrets/<name>`).
```yaml
services:
  app:
    secrets:
      - db_password
      # or long form:
      - source: db_password
        target: /run/secrets/db_password
        uid: "1000"
        mode: 0400
```

### `configs`

Mount non-sensitive runtime configuration as files.
```yaml
services:
  app:
    configs:
      - my_config
      # or long form:
      - source: my_config
        target: /etc/app/config.yaml
        mode: 0444
```

### `deploy`

Deployment configuration (replicas, resource limits, placement, etc.).
```yaml
services:
  web:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M
      restart_policy:
        condition: on-failure
        max_attempts: 3
```

### `develop`

Development configuration for Compose Watch (auto-sync/rebuild on file changes). Requires Compose >= 2.22.0.
```yaml
services:
  web:
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - node_modules/
        - action: rebuild
          path: package.json
```

Watch actions:
- `sync` — copy changed files into the container (hot reload)
- `rebuild` — rebuild the image and recreate the container
- `sync+restart` — sync files then restart the container

### `logging`

Logging configuration for the service.
```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### `labels`

Add metadata labels to containers.
```yaml
services:
  web:
    labels:
      com.example.version: "1.0"
      com.example.environment: "production"
```

### `extra_hosts`

Add hostname mappings to the container's `/etc/hosts`.
```yaml
services:
  app:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

### `user`

Override the user used to run the container process.
```yaml
services:
  app:
    user: "1000:1000"
```

### `working_dir`

Override the working directory in the container.

### `stdin_open` / `tty`

Keep STDIN open / allocate a pseudo-TTY (equivalent to `docker run -it`).
```yaml
services:
  app:
    stdin_open: true
    tty: true
```

### `platform`

Target platform for the container image.
```yaml
services:
  app:
    image: myapp
    platform: linux/amd64
```

---

## `networks`

Define named networks shared across services.

```yaml
networks:
  front-tier:
    driver: bridge
  back-tier:
    driver: bridge
    internal: true   # no external connectivity
```

| Attribute | Description |
|-----------|-------------|
| `driver` | Network driver (`bridge`, `overlay`, `host`, `none`) |
| `internal` | Prevent external access to the network |
| `external` | Network managed outside of Compose |
| `name` | Custom network name (not prefixed with project name) |
| `attachable` | Allow standalone containers to join |
| `ipam` | Custom IP address management config |
| `labels` | Metadata labels |
| `enable_ipv6` | Enable IPv6 |

### External network

```yaml
networks:
  my-net:
    external: true
    name: my-pre-existing-network
```

### Custom IPAM

```yaml
networks:
  custom:
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
```

---

## `volumes`

Define named volumes for persistent data.

```yaml
volumes:
  db-data:           # uses default driver config
  app-logs:
    driver: local
    driver_opts:
      type: none
      device: /host/path
      o: bind
```

| Attribute | Description |
|-----------|-------------|
| `driver` | Volume driver (default: `local`) |
| `driver_opts` | Options to pass to the driver |
| `external` | Volume managed outside of Compose |
| `name` | Custom volume name |
| `labels` | Metadata labels |

### External volume

```yaml
volumes:
  data:
    external: true
    name: existing-volume-name
```

---

## `configs`

Define non-sensitive configuration data available to services.

```yaml
configs:
  nginx_config:
    file: ./nginx.conf     # from a local file
  app_config:
    external: true         # already exists on the platform
```

---

## `secrets`

Define sensitive configuration data.

```yaml
secrets:
  db_password:
    file: ./db_password.txt    # value read from file
  api_key:
    environment: API_KEY       # value from environment variable
  tls_cert:
    external: true             # managed outside of Compose
```

---

## Variable Interpolation

Use `${VARIABLE}` or `$VARIABLE` in Compose files to substitute values from environment variables or `.env` files.

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-16}
    environment:
      POSTGRES_USER: ${DB_USER}
```

Default values: `${VARIABLE:-default}` — uses `default` if unset or empty.
Required values: `${VARIABLE:?error message}` — fails if unset or empty.

---

## YAML Anchors and Extensions

Use YAML anchors (`&`) and aliases (`*`) to avoid repetition:

```yaml
x-common-env: &common-env
  LOG_LEVEL: info
  TZ: UTC

services:
  api:
    environment:
      <<: *common-env
      API_PORT: 8080
  worker:
    environment:
      <<: *common-env
```

---

## Complete Example

```yaml
name: myapp

services:
  proxy:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    configs:
      - source: nginx_config
        target: /etc/nginx/conf.d/app.conf
    secrets:
      - source: tls_cert
        target: /etc/nginx/ssl/cert.pem
    networks:
      - front-tier
      - back-tier
    depends_on:
      backend:
        condition: service_healthy

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgres://db:5432/mydb
    secrets:
      - db_password
    networks:
      - back-tier
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    networks:
      - back-tier
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

networks:
  front-tier:
  back-tier:
    internal: true

volumes:
  db-data:

configs:
  nginx_config:
    file: ./nginx.conf

secrets:
  db_password:
    file: ./secrets/db_password.txt
  tls_cert:
    file: ./secrets/tls.pem
```

## References

- [Compose File Reference](https://docs.docker.com/reference/compose-file/)
- [Services Reference](https://docs.docker.com/reference/compose-file/services/)
- [Networks Reference](https://docs.docker.com/reference/compose-file/networks/)
- [Volumes Reference](https://docs.docker.com/reference/compose-file/volumes/)
- [Configs Reference](https://docs.docker.com/reference/compose-file/configs/)
- [Secrets Reference](https://docs.docker.com/reference/compose-file/secrets/)
- [Build Specification](https://docs.docker.com/reference/compose-file/build/)
- [Deploy Specification](https://docs.docker.com/reference/compose-file/deploy/)
