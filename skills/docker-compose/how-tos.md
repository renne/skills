# Docker Compose How-Tos

Sources:
- https://docs.docker.com/compose/how-tos/networking/
- https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/
- https://docs.docker.com/compose/how-tos/environment-variables/envvars-precedence/
- https://docs.docker.com/compose/how-tos/use-secrets/
- https://docs.docker.com/compose/how-tos/profiles/
- https://docs.docker.com/compose/how-tos/file-watch/
- https://docs.docker.com/compose/how-tos/multiple-compose-files/
- https://docs.docker.com/compose/how-tos/production/

---

## Networking

### Default network

When you run `docker compose up`, Compose automatically creates a network named `<project>_default`. All services join this network and can reach each other by service name.

```yaml
# myapp/compose.yaml
services:
  web:
    image: myapp
  db:
    image: postgres
    ports:
      - "8001:5432"
```

- `web` can connect to `db` at `postgres://db:5432` (container port, service name as host).
- The host machine connects at `postgres://localhost:8001` (host port).

> Always reference containers by **service name**, not IP address. IPs can change when containers are recreated.

### Custom networks

Use the top-level `networks` element to create custom networks and assign services selectively:

```yaml
services:
  proxy:
    image: nginx
    networks:
      - front-tier
      - back-tier
  app:
    image: myapp
    networks:
      - back-tier
  db:
    image: postgres
    networks:
      - back-tier

networks:
  front-tier:
  back-tier:
    internal: true  # no external Internet access
```

`proxy` can reach both `app` and `db`. `db` and `app` are on `back-tier` only — they are isolated from the front tier.

### Network aliases (links)

Add additional hostnames for a service within a network:

```yaml
services:
  web:
    links:
      - "db:database"   # db is also reachable as "database"
```

### External (pre-existing) network

Connect to a network created outside of Compose:

```yaml
networks:
  my-net:
    external: true
    name: my-pre-existing-network
```

### Custom network name

Prevent Compose from prepending the project name:

```yaml
networks:
  shared:
    name: shared-network
```

### Static IP addresses

```yaml
services:
  app:
    networks:
      custom:
        ipv4_address: 172.28.1.5

networks:
  custom:
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

---

## Environment Variables

### Setting environment variables

**`environment` attribute** (inline):
```yaml
services:
  app:
    environment:
      NODE_ENV: production
      PORT: 3000
      DEBUG:            # value passed through from shell
```

**`env_file` attribute** (from file):
```yaml
services:
  app:
    env_file:
      - .env
      - .env.local
      - path: .env.optional
        required: false   # silently ignored if missing (Compose >= 2.24.0)
```

**`docker compose run -e` (one-off override)**:
```bash
docker compose run -e DEBUG=true app npm test
```

### The `.env` file

Place a `.env` file in the project directory (next to `compose.yaml`) to define default values for variable interpolation in the Compose file itself (not for container environment unless also referenced via `environment` or `env_file`):

```dotenv
POSTGRES_VERSION=16
APP_PORT=8080
```

Use `--env-file` to specify a different file:
```bash
docker compose --env-file .env.staging up -d
```

### Variable interpolation in compose.yaml

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-16}
  web:
    ports:
      - "${APP_PORT:-3000}:3000"
```

Syntax:
- `${VAR}` — substitute value
- `${VAR:-default}` — use `default` if unset or empty
- `${VAR:?error}` — fail with error if unset or empty

### Environment variable precedence (highest to lowest)

1. `docker compose run -e VAR=value` (CLI flag with explicit value)
2. `docker compose run -e VAR` (CLI flag — value from shell)
3. `environment` attribute with explicit value in Compose file
4. `env_file` attribute with explicit value in file
5. `environment` attribute without value (pass-through from shell or `.env`)
6. `env_file` attribute without value (pass-through from shell or `.env`)
7. Image `ENV` directive in Dockerfile

> Shell environment takes precedence over the `.env` file when both are present.

---

## Secrets

Use secrets to pass sensitive data (passwords, API keys, certificates) to containers without using environment variables.

Secrets are mounted at `/run/secrets/<name>` inside the container.

### Define and use a secret

```yaml
services:
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Secret from environment variable

```yaml
secrets:
  api_key:
    environment: API_KEY   # value taken from host environment variable API_KEY
```

### Multi-service secret sharing

```yaml
services:
  frontend:
    image: myapp
    secrets:
      - db_password
  backend:
    image: myapi
    secrets:
      - db_password
      - db_root_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  db_root_password:
    file: ./secrets/db_root_password.txt
```

### Build-time secrets

```yaml
services:
  app:
    build:
      context: .
      secrets:
        - npm_token

secrets:
  npm_token:
    environment: NPM_TOKEN
```

### Long-form secret syntax (custom mount options)

```yaml
services:
  app:
    secrets:
      - source: db_password
        target: /run/secrets/db_password
        uid: "1000"
        gid: "1000"
        mode: 0400
```

> Many Docker Official Images (like `mysql`, `postgres`) support `_FILE` environment variables that read values from secret files mounted at `/run/secrets/`.

---

## Profiles

Profiles let you selectively start services. Services without a `profiles` attribute always start; services with `profiles` only start when their profile is active.

### Assign profiles to services

```yaml
services:
  app:
    image: myapp              # always started
  db:
    image: postgres           # always started
  phpmyadmin:
    image: phpmyadmin
    profiles:
      - debug                 # only started with --profile debug
  seed:
    image: myapp
    command: seed-database
    profiles:
      - tools                 # only started with --profile tools
```

### Activate profiles

```bash
docker compose --profile debug up          # start app, db, phpmyadmin
docker compose --profile tools run seed    # run the seed service
COMPOSE_PROFILES=debug docker compose up   # via environment variable
```

### Multiple profiles

```bash
docker compose --profile debug --profile tools up
# or
COMPOSE_PROFILES=debug,tools docker compose up
```

### All profiles

```bash
docker compose --profile "*" up
```

### One-off service targeting

If you explicitly target a profiled service by name, Compose starts it regardless of whether its profile is active (along with its `depends_on` dependencies):

```bash
docker compose run seed     # starts seed even though --profile tools is not set
```

---

## Compose Watch (Live File Sync)

Requires Docker Compose >= 2.22.0.

Compose Watch automatically syncs, rebuilds, or restarts services when local files change, supporting a hands-off development workflow.

### Configuration in `compose.yaml`

```yaml
services:
  web:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - node_modules/
        - action: rebuild
          path: package.json
        - action: sync+restart
          path: ./config
          target: /app/config
```

### Watch actions

| Action | Behavior | Ideal for |
|--------|----------|-----------|
| `sync` | Copy changed files into the container | Interpreted languages, hot-reload frameworks |
| `rebuild` | Rebuild the image and recreate the container | Compiled languages, `package.json` changes |
| `sync+restart` | Sync files then restart the container | Config file changes that don't need a rebuild |

### Running watch mode

```bash
docker compose up --watch
# or separately:
docker compose watch
```

### Path rules

- Paths are relative to the project directory
- Directories are watched recursively
- Glob patterns are not supported
- `.dockerignore` rules apply
- `.git` directories are ignored automatically
- `ignore` paths are relative to the `path` of the watch action

### Prerequisites

The service container must have `stat`, `mkdir`, and `rmdir` available, and the container `USER` must be able to write to the `target` path.

---

## Multiple Compose Files

### Merge with `-f` flag

Combine Compose files; later files override earlier ones:

```bash
docker compose -f compose.yaml -f compose.override.yaml up
```

- Simple attributes and maps: later file wins
- Lists: merged by appending
- Relative paths: resolved relative to the **first** file's directory

### Extend a service (`extends`)

Reuse a service definition from another file, overriding specific fields:

```yaml
# compose.yaml
services:
  web:
    extends:
      file: common-services.yaml
      service: webapp
    environment:
      ENV: production
```

### Include sub-Compose files (`include`)

Include another Compose file as a sub-project (resources are namespaced):

```yaml
# compose.yaml
include:
  - path: ./infra/infra.yaml
  - path: ./monitoring/monitoring.yaml

services:
  app:
    image: myapp
```

### Environment-specific overrides pattern

```
compose.yaml              # base configuration
compose.override.yaml     # local development overrides (loaded automatically)
compose.production.yaml   # production-specific settings
compose.ci.yaml           # CI environment settings
```

The `compose.override.yaml` file is loaded automatically alongside `compose.yaml`.

---

## Production Deployment

### Using override files for production

```bash
docker compose -f compose.yaml -f compose.production.yaml up -d
```

`compose.production.yaml` contains only the differences from the base file:
```yaml
services:
  web:
    restart: always
    environment:
      LOG_LEVEL: warn
    ports:
      - "80:80"
      - "443:443"
```

### Deploying changes to a single service

Rebuild and recreate only the changed service without restarting dependencies:
```bash
docker compose up -d --no-deps --build web
```

### Deploying to a remote Docker host

Set these environment variables to target a remote host:
```bash
export DOCKER_HOST=ssh://user@remote-host
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/path/to/certs

docker compose up -d   # runs against the remote host
```

---

## CLI Global Options

```bash
docker compose [OPTIONS] COMMAND

Options:
  -f, --file FILE              Specify alternate Compose file(s)
  -p, --project-name NAME      Set project name
  --profile PROFILE            Enable profile
  --env-file FILE              Specify alternate environment file
  --project-directory PATH     Specify alternate working directory
  --parallel N                 Max parallelism (-1 = unlimited)
  --dry-run                    Show what would happen without doing it
  --ansi [auto|always|never]   ANSI output control
```

### Project name resolution order

1. `-p` / `--project-name` CLI flag
2. `COMPOSE_PROJECT_NAME` environment variable
3. `name:` top-level attribute in Compose file
4. Basename of the directory containing the Compose file
5. Basename of the current directory

### Useful environment variables

| Variable | Equivalent flag | Description |
|----------|----------------|-------------|
| `COMPOSE_FILE` | `-f` | Path(s) to Compose file(s) |
| `COMPOSE_PROJECT_NAME` | `-p` | Project name |
| `COMPOSE_PROFILES` | `--profile` | Active profiles |
| `COMPOSE_PARALLEL_LIMIT` | `--parallel` | Max parallelism |
| `COMPOSE_IGNORE_ORPHANS` | — | Don't warn about orphaned containers |
| `COMPOSE_MENU` | — | Show/hide helper menu in `up` mode |
| `DOCKER_HOST` | — | Remote Docker daemon address |

## References

- [Networking in Compose](https://docs.docker.com/compose/how-tos/networking/)
- [Set environment variables](https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/)
- [Environment variable precedence](https://docs.docker.com/compose/how-tos/environment-variables/envvars-precedence/)
- [Variable interpolation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [Use secrets](https://docs.docker.com/compose/how-tos/use-secrets/)
- [Use service profiles](https://docs.docker.com/compose/how-tos/profiles/)
- [Use Compose Watch](https://docs.docker.com/compose/how-tos/file-watch/)
- [Multiple Compose files](https://docs.docker.com/compose/how-tos/multiple-compose-files/)
- [Compose in production](https://docs.docker.com/compose/how-tos/production/)
- [CLI reference](https://docs.docker.com/reference/cli/docker/compose/)
