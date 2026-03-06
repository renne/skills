# Docker Compose Overview

Source: https://docs.docker.com/compose/

## What is Docker Compose?

Docker Compose is a tool for defining and running multi-container applications. With Compose, you use a YAML configuration file (`compose.yaml`) to configure your application's services, networks, and volumes, and then create and start all services with a single command.

## Key Concepts

### Services

Computing components of an application are defined as **services**. A service is an abstract concept implemented by running the same container image and configuration one or more times. Services communicate with each other through networks and store persistent data in volumes.

### Networks

A **network** is a platform capability abstraction that establishes IP routes between containers within services. By default, Compose creates a single network for your app; each container joins this network and is reachable by other containers using the service name as the hostname.

### Volumes

**Volumes** are persistent data stores. Services store and share persistent data into volumes. Volumes survive container restarts and can be shared across services.

### Configs

**Configs** allow services to adapt their behavior without rebuilding a Docker image. Configs behave like volumes inside the container—they are mounted as files—but are defined differently at the platform level.

### Secrets

**Secrets** are a specific flavor of configuration data for sensitive data (passwords, certificates, API keys) that should not be exposed without security considerations. Secrets are made available to services as files mounted at `/run/secrets/<secret_name>`.

### Projects

A **project** is an individual deployment of an application specification. The project name is used to group and isolate resources. By default it is derived from the directory name, but can be overridden with `--project-name` or the `COMPOSE_PROJECT_NAME` environment variable.

## The Compose File

The default Compose file is named `compose.yaml` (preferred), `compose.yml`, `docker-compose.yaml`, or `docker-compose.yml` in the working directory. When both `compose.yaml` and `docker-compose.yaml` exist, Compose prefers `compose.yaml`.

### Minimal example

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

## Key Features and Use Cases

### Simplified control

Define and manage multi-container apps in a single YAML file, making orchestration straightforward.

### Development environments

Run an application in an isolated environment with all dependencies (databases, caches, queues) started with `docker compose up`. Compose can replace a multi-page developer getting-started guide with a single Compose file and a few commands.

### Automated testing environments

Create and destroy isolated testing environments for CI/CD with:
```bash
docker compose up -d
# run tests
docker compose down
```

### Rapid development with Compose Watch

Compose Watch automatically updates running services when source files change—no manual rebuild needed:
```bash
docker compose up --watch
# or
docker compose watch
```

### Portability

Compose supports variables in the Compose file, allowing the same file to be used across different environments (development, staging, production) by adjusting variable values.

## Getting Started

### Prerequisites

- Docker Desktop (includes Compose) or Docker Engine + Compose plugin
- A basic understanding of Docker concepts

### Quickstart: Python/Flask + Redis

1. Create a project directory and add `app.py`, `requirements.txt`, and a `Dockerfile`.

2. Create `compose.yaml`:
   ```yaml
   services:
     web:
       build: .
       ports:
         - "8000:5000"
     redis:
       image: redis:alpine
   ```

3. Start the application:
   ```bash
   docker compose up
   ```

4. Open `http://localhost:8000/` in your browser.

5. Stop the application:
   ```bash
   docker compose down
   ```

### Detached mode

Run services in the background:
```bash
docker compose up -d
docker compose ps      # list running services
docker compose logs    # view logs
docker compose down    # stop and remove
```

## CLI Overview

| Command | Description |
|---------|-------------|
| `docker compose up` | Create and start all services |
| `docker compose up -d` | Start in detached (background) mode |
| `docker compose up --watch` | Start with file watch mode |
| `docker compose down` | Stop and remove containers and networks |
| `docker compose down -v` | Also remove volumes |
| `docker compose ps` | List containers |
| `docker compose logs` | View output from containers |
| `docker compose exec <svc> <cmd>` | Execute a command in a running container |
| `docker compose run <svc> <cmd>` | Run a one-off command on a service |
| `docker compose build` | Build or rebuild service images |
| `docker compose pull` | Pull service images |
| `docker compose push` | Push service images |
| `docker compose restart` | Restart service containers |
| `docker compose stop` | Stop services without removing |
| `docker compose start` | Start previously stopped services |
| `docker compose config` | Validate and view the resolved Compose file |
| `docker compose ls` | List running Compose projects |

## Multiple Compose Files

You can combine multiple Compose files using:

- **`-f` flag**: Merge files in order; later files override earlier ones.
  ```bash
  docker compose -f compose.yaml -f compose.production.yaml up -d
  ```
- **`extends`**: Extend a service definition from another file.
- **`include`**: Include an entire Compose file as a sub-project.

## Production Considerations

When deploying to production:

- Remove volume bindings for application code (use `COPY` in Dockerfile instead).
- Bind to appropriate host ports.
- Set `restart: always` (or `unless-stopped`) on services to avoid downtime.
- Use secrets instead of environment variables for sensitive data.
- Use a `compose.production.yaml` override file for production-specific settings.
- Rebuild and recreate only changed services:
  ```bash
  docker compose up -d --no-deps --build <service>
  ```

## References

- [Docker Compose Overview](https://docs.docker.com/compose/)
- [Quickstart Guide](https://docs.docker.com/compose/gettingstarted/)
- [How Compose Works](https://docs.docker.com/compose/intro/compose-application-model/)
- [Features and Use Cases](https://docs.docker.com/compose/intro/features-uses/)
- [Compose File Reference](https://docs.docker.com/reference/compose-file/)
- [CLI Reference](https://docs.docker.com/reference/cli/docker/compose/)
- [Awesome Compose samples](https://github.com/docker/awesome-compose)
