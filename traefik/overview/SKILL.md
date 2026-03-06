---
name: overview
description: Traefik proxy overview covering core concepts (entrypoints, routers, middlewares, services, providers), installation via Docker and binary, static vs. dynamic configuration, and the Traefik request lifecycle. Use when getting started with Traefik, understanding how traffic flows through the proxy, or setting up Traefik for the first time.
---
# Traefik Overview

Source: https://doc.traefik.io/traefik/

## What is Traefik?

Traefik is a modern, cloud-native **application proxy** and the core of the Traefik Hub Runtime Platform. It is designed to integrate with your existing infrastructure components and configure itself automatically and dynamically. Unlike traditional reverse proxies that require manual configuration file updates and restarts, Traefik reads its routing rules directly from your infrastructure (Docker labels, Kubernetes annotations/CRDs, configuration files, etc.) and updates in real time without downtime.

Traefik has 3.3 billion+ downloads and 55,000+ GitHub stars. Beyond reverse proxying, the broader Traefik platform includes:
- **API management** — manage, secure, and publish APIs
- **API gateway** — centralized entry point for microservices
- **AI gateway** — manage AI model endpoints
- **API mocking** — stub backends for development and testing

## Core Concepts

### Entrypoints

Entrypoints define the network ports and protocols on which Traefik listens for incoming traffic. They are declared in the **static configuration** and rarely change at runtime.

```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
```

### Routers

Routers analyze each incoming request and determine which service should handle it. A router has:
- **Rule**: An expression (e.g., `Host("example.com") && PathPrefix("/api")`) that matches requests.
- **EntryPoints**: Which entrypoints to listen on.
- **Middlewares** (optional): A chain of processing steps before the request reaches the service.
- **Service**: The backend that will handle the request.
- **TLS** (optional): Whether to terminate TLS on this router.

### Middlewares

Middlewares are processing steps that intercept requests (and responses) between the router and the service. They can authenticate, redirect, rewrite, rate-limit, compress, and more. Multiple middlewares can be chained.

### Services

Services define how to reach the actual backend applications. Traefik supports:
- **Load Balancers**: Distribute traffic across multiple servers with configurable strategies.
- **Weighted Round Robin**: Send traffic to multiple services with configured weights.
- **Mirroring**: Mirror traffic to a secondary service.

### Providers

Providers are infrastructure components from which Traefik reads its dynamic routing configuration. Supported providers include:
- Docker (labels on containers)
- Kubernetes (Ingress, IngressRoute CRDs)
- File (YAML/TOML files)
- Consul, etcd, ZooKeeper (KV stores)
- Marathon, Nomad, ECS, and others

## Configuration Model

Traefik has two distinct configuration types:

### Static Configuration

Defined at **startup** via one of:
- A configuration file (`traefik.yaml` or `traefik.toml`)
- CLI flags
- Environment variables

Static configuration declares entrypoints, providers, certificate resolvers, and global options. **Traefik must be restarted to pick up changes to static configuration.**

```yaml
# traefik.yaml (static config)
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
  file:
    directory: /etc/traefik/dynamic/
    watch: true

api:
  dashboard: true

log:
  level: INFO
```

### Dynamic Configuration

Defined at **runtime** and hot-reloaded by providers. Declares routers, middlewares, services, and TLS options. No restart required when changing dynamic configuration.

```yaml
# dynamic/routes.yaml
http:
  routers:
    my-app:
      rule: "Host(`app.example.com`)"
      entryPoints:
        - websecure
      service: my-app
      middlewares:
        - my-auth
      tls: {}

  middlewares:
    my-auth:
      basicAuth:
        users:
          - "admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"

  services:
    my-app:
      loadBalancer:
        servers:
          - url: "http://192.168.1.10:8080"
```

## Request Lifecycle

```
Client Request
      │
      ▼
  EntryPoint (e.g., :443)
      │
      ▼
  Router (matches rule, selects service)
      │
      ▼
  Middleware Chain (auth, rate-limit, headers, etc.)
      │
      ▼
  Service (load balancer → backend server)
      │
      ▼
Client Response (passes back through middleware chain)
```

## Installation

### Docker (Quickstart)

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"   # Traefik dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"
```

Start with `docker compose up -d` and visit http://whoami.localhost.

### Binary

```bash
# Download from https://github.com/traefik/traefik/releases
wget https://github.com/traefik/traefik/releases/download/v3.3.4/traefik_v3.3.4_linux_amd64.tar.gz
tar -xzf traefik_v3.3.4_linux_amd64.tar.gz
./traefik --configFile=traefik.yaml
```

### Kubernetes (Helm)

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik
```

## Dashboard

The Traefik dashboard provides a visual overview of routers, services, middlewares, and providers. Enable it in static config:

```yaml
api:
  dashboard: true
  insecure: false   # Set to true only for testing; secure with middleware in production
```

Access at `https://<traefik-host>/dashboard/` (note the trailing slash).

To expose the dashboard securely:

```yaml
http:
  routers:
    dashboard:
      rule: "Host(`traefik.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      entryPoints:
        - websecure
      service: api@internal
      middlewares:
        - dashboard-auth
      tls: {}
  middlewares:
    dashboard-auth:
      basicAuth:
        users:
          - "admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"
```

## Key Router Rule Syntax

| Rule | Description |
|------|-------------|
| `Host("example.com")` | Match by hostname |
| `PathPrefix("/api")` | Match by URL path prefix |
| `Path("/exact")` | Match exact path |
| `Method("GET", "POST")` | Match HTTP methods |
| `Headers("X-Custom", "value")` | Match by header value |
| `HeadersRegexp("X-Custom", "val.*")` | Match header by regex |
| `ClientIP("192.168.1.0/24")` | Match by client IP range |
| `Query("key", "value")` | Match by query parameter |
| `&&` | AND combinator |
| `\|\|` | OR combinator |
| `!` | NOT combinator |

## References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Getting Started](https://doc.traefik.io/traefik/getting-started/quick-start/)
- [Configuration Overview](https://doc.traefik.io/traefik/getting-started/configuration-overview/)
- [Routing Overview](https://doc.traefik.io/traefik/routing/overview/)
- [Install Traefik](https://doc.traefik.io/traefik/getting-started/install-traefik/)
