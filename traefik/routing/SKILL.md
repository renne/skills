---
name: routing
description: Traefik routing configuration covering HTTP/TCP/UDP routers, router rules, services (load balancers, weighted round-robin, mirroring), entrypoint configuration, and priority. Use when configuring Traefik routers and services, writing routing rules, setting up load balancing, or working with TCP/UDP traffic routing.
---
# Traefik Routing

Source: https://doc.traefik.io/traefik/routing/overview/

## HTTP Routing

### Routers

HTTP routers connect incoming requests to services based on rules. Each router must have:
- A **rule** (request matching expression)
- At least one **entrypoint**
- A **service** to forward to

#### Router Configuration (file provider)

```yaml
http:
  routers:
    my-router:
      rule: "Host(`example.com`) && PathPrefix(`/api`)"
      entryPoints:
        - websecure
      service: my-service
      middlewares:
        - auth-middleware
      tls: {}
      priority: 10
```

#### Router Configuration (Docker labels)

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-router.rule=Host(`example.com`)"
  - "traefik.http.routers.my-router.entrypoints=websecure"
  - "traefik.http.routers.my-router.tls=true"
  - "traefik.http.routers.my-router.middlewares=auth-middleware"
```

### Router Rules

Rules are expressions that match incoming requests. Combine with `&&` (AND), `||` (OR), and `!` (NOT).

| Matcher | Example | Description |
|---------|---------|-------------|
| `Host` | `Host("example.com")` | Match hostname (exact) |
| `HostRegexp` | `HostRegexp("^(www\\.)?example\\.com$")` | Match hostname by regex |
| `Path` | `Path("/exact/path")` | Match exact URL path |
| `PathPrefix` | `PathPrefix("/api")` | Match URL path prefix |
| `PathRegexp` | `PathRegexp("^/api/v[0-9]+")` | Match path by regex |
| `Method` | `Method("GET", "POST")` | Match HTTP methods |
| `Headers` | `Headers("X-Env", "prod")` | Match header exact value |
| `HeadersRegexp` | `HeadersRegexp("X-Env", "prod.*")` | Match header by regex |
| `Query` | `Query("debug", "true")` | Match query param |
| `QueryRegexp` | `QueryRegexp("id", "[0-9]+")` | Match query param by regex |
| `ClientIP` | `ClientIP("192.168.1.0/24")` | Match client IP or CIDR |

#### Rule Examples

```
# Match all traffic to example.com
Host(`example.com`)

# Match example.com with /api prefix
Host(`example.com`) && PathPrefix(`/api`)

# Match either of two hosts
Host(`app.example.com`) || Host(`api.example.com`)

# Match path but exclude a sub-path
PathPrefix(`/app`) && !Path(`/app/admin`)
```

### Router Priority

When multiple routers could match the same request, priority determines which one wins. Higher numbers take precedence. By default, priority equals the length of the router's rule string (longer rules have higher priority).

```yaml
http:
  routers:
    specific-router:
      rule: "Host(`example.com`) && Path(`/specific`)"
      priority: 100
      service: specific-service
    catch-all:
      rule: "Host(`example.com`)"
      priority: 1
      service: default-service
```

## HTTP Services

### Load Balancer

The most common service type. Distributes requests across multiple backend servers using a round-robin strategy by default.

```yaml
http:
  services:
    my-service:
      loadBalancer:
        servers:
          - url: "http://192.168.1.10:8080"
          - url: "http://192.168.1.11:8080"
        healthCheck:
          path: /health
          interval: 10s
          timeout: 3s
        sticky:
          cookie:
            name: lb_cookie
            secure: true
            httpOnly: true
```

#### Load Balancer Options

| Option | Description |
|--------|-------------|
| `servers[].url` | Backend server URL |
| `servers[].weight` | Relative weight for weighted round-robin |
| `healthCheck.path` | Path for health check requests |
| `healthCheck.interval` | How often to check (default: 30s) |
| `healthCheck.timeout` | Timeout for health check (default: 5s) |
| `passHostHeader` | Pass the original `Host` header (default: true) |
| `sticky.cookie.name` | Enable sticky sessions via cookie |
| `responseForwarding.flushInterval` | Streaming response flush interval |

### Weighted Round Robin (WRR)

Distribute traffic across multiple services with weighted percentages. Useful for canary deployments or A/B testing.

```yaml
http:
  services:
    weighted-service:
      weighted:
        services:
          - name: v1-service
            weight: 90
          - name: v2-canary
            weight: 10
```

### Mirroring

Mirror a fraction of traffic to a secondary service for testing.

```yaml
http:
  services:
    mirroring-service:
      mirroring:
        service: main-service
        mirrors:
          - name: test-service
            percent: 10
```

## EntryPoints

EntryPoints are the network entry points (ports/protocols) where Traefik accepts incoming traffic.

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt
  custom-port:
    address: ":8443"
  tcp-port:
    address: ":9000/tcp"
  udp-port:
    address: ":9001/udp"
```

### HTTP/S Redirect via EntryPoint

Redirect all HTTP traffic to HTTPS automatically:

```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
```

## TCP Routing

TCP routers route raw TCP connections based on SNI (Server Name Indication) from the TLS ClientHello.

### TCP Router (file provider)

```yaml
tcp:
  routers:
    my-tcp-router:
      rule: "HostSNI(`db.example.com`)"
      entryPoints:
        - tcp-port
      service: my-db-service
      tls:
        passthrough: true   # Don't terminate TLS; pass to backend

  services:
    my-db-service:
      loadBalancer:
        servers:
          - address: "192.168.1.20:5432"
```

### TCP Router (Kubernetes CRD)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: postgres-route
spec:
  entryPoints:
    - tcp-port
  routes:
    - match: HostSNI(`db.example.com`)
      services:
        - name: postgres-service
          port: 5432
  tls:
    passthrough: true
```

### TCP SNI Rules

| Rule | Description |
|------|-------------|
| `HostSNI("example.com")` | Match by TLS SNI hostname |
| `HostSNI("*")` | Match all (for plain TCP without SNI) |
| `ClientIP("10.0.0.0/8")` | Match by client IP |

### TLS Passthrough

With `passthrough: true`, Traefik does not terminate TLS. It reads the SNI to choose a backend and then forwards the full encrypted stream. The backend must present the TLS certificate.

## UDP Routing

UDP routers route UDP datagrams to backend services. No SNI or hostname matching is possible (UDP has no TLS); routing is purely entrypoint-based.

```yaml
udp:
  routers:
    my-udp-router:
      entryPoints:
        - udp-port
      service: my-udp-service

  services:
    my-udp-service:
      loadBalancer:
        servers:
          - address: "192.168.1.30:5060"
```

### UDP Router (Kubernetes CRD)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteUDP
metadata:
  name: udp-route
spec:
  entryPoints:
    - udp-port
  routes:
    - services:
        - name: my-udp-service
          port: 5060
```

## Internal Services

Traefik provides built-in internal services referenced as `name@internal`:

| Service | Description |
|---------|-------------|
| `api@internal` | Traefik API (used for dashboard) |
| `dashboard@internal` | Dashboard UI (alias for api@internal) |
| `noop@internal` | Silently drops all requests |
| `ping@internal` | Health check endpoint |

```yaml
http:
  routers:
    dashboard:
      rule: "Host(`traefik.example.com`)"
      service: api@internal
      middlewares:
        - auth
      tls: {}
```

## References

- [HTTP Routing Overview](https://doc.traefik.io/traefik/reference/routing-configuration/dynamic-configuration-methods/)
- [HTTP Routers](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/router/)
- [HTTP Services](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/services/)
- [EntryPoints](https://doc.traefik.io/traefik/routing/entrypoints/)
- [TCP Routing](https://doc.traefik.io/traefik/reference/routing-configuration/tcp/)
- [UDP Routing](https://doc.traefik.io/traefik/reference/routing-configuration/udp/)
- [IngressRouteTCP CRD](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/tcp/ingressroutetcp/)
- [IngressRouteUDP CRD](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/udp/ingressrouteudp/)
