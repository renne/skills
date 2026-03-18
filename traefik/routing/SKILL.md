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

### gRPC Routing

gRPC runs over HTTP/2. Traefik can route gRPC traffic by detecting the `Content-Type: application/grpc` header on HTTP/2 connections.

**Backend scheme: `h2c` (HTTP/2 cleartext)**

When the backend server speaks HTTP/2 without TLS (gRPC cleartext), use `h2c://` in the service URL:

```yaml
http:
  routers:
    grpc-router:
      rule: "Host(`api.example.com`) && PathPrefix(`/mypackage.MyService/`)"
      entryPoints: [websecure]
      service: grpc-backend
      tls: {}
      priority: 100

  services:
    grpc-backend:
      loadBalancer:
        servers:
          - url: "h2c://backend-service:80"
```

With Docker Compose labels:
```yaml
labels:
  - "traefik.http.routers.grpc.rule=Host(`api.example.com`) && PathPrefix(`/mypackage.MyService/`)"
  - "traefik.http.routers.grpc.entrypoints=websecure"
  - "traefik.http.routers.grpc.tls=true"
  - "traefik.http.services.grpc.loadbalancer.servers.url=h2c://backend-service:80"
  - "traefik.http.routers.grpc.priority=100"
```

#### Mixed gRPC + REST on the same port (cmux)

Some servers (e.g., Netbird management) use `cmux` to multiplex gRPC, HTTP/1.1, and HTTP/2 on the same port. Use separate Traefik routers with different priorities:

```yaml
# High-priority gRPC router (PathPrefix matches gRPC service paths)
- "traefik.http.routers.grpc.rule=Host(`api.example.com`) && PathPrefix(`/mypackage.MyService/`)"
- "traefik.http.routers.grpc.priority=100"
- "traefik.http.services.grpc-svc.loadbalancer.servers.url=h2c://backend:80"

# Lower-priority REST router (catches everything else)
- "traefik.http.routers.rest.rule=Host(`api.example.com`)"
- "traefik.http.routers.rest.priority=10"
- "traefik.http.services.rest-svc.loadbalancer.servers.url=http://backend:80"
```

#### Testing gRPC endpoints

> ⚠️ **Plain `curl` (HTTP/1.1) always returns 404 for gRPC endpoints** — gRPC requires HTTP/2.

```bash
# ❌ Wrong: HTTP/1.1 curl returns 404 even if routing is correct
curl https://api.example.com/mypackage.MyService/Method

# ✅ Correct: force HTTP/2
curl --http2 -k https://api.example.com/mypackage.MyService/Method

# ✅ Or use grpcurl
grpcurl -plaintext api.example.com:80 list
grpcurl api.example.com:443 list  # TLS
```

Check Traefik access logs for `HTTP/2.0 200` to confirm gRPC is routing correctly.



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

## Practical Routing Notes

### TCP SNI Passthrough Conflicts with HTTP Routers

When TCP SNI passthrough routers coexist with HTTP routers on the same Traefik instance, the **TCP router matches first** at the transport layer — before any HTTP router sees the request. This is because TLS passthrough happens at a lower level (TCP), not after TLS termination.

**Symptom:** An HTTPS request for a hostname that has *both* a TCP SNI passthrough rule *and* an HTTP router is routed to the TCP backend (e.g., a reverse proxy), even if the intended target is the HTTP service.

**Common scenario:** NetBird reverse-proxy containers expose services via TCP SNI passthrough labels. If a service is later reverted to Traefik HTTP routing, the old TCP SNI label must be explicitly removed from the proxy container. Otherwise the TCP router intercepts all HTTPS traffic for that hostname, and the user sees the proxy's error page.

**Fix:** Remove the TCP SNI passthrough labels for hostnames that should be handled by HTTP routers. Only TCP labels for the proxy's own domains (e.g., `proxy.example.com`, `*.proxy.example.com`) should remain.

```yaml
# Keep ONLY proxy-domain TCP labels:
- "traefik.tcp.routers.netbird-proxy-base.rule=HostSNI(`proxy.example.com`)"
- "traefik.tcp.routers.netbird-proxy-wild.rule=HostSNIRegexp(`^.+\\.proxy\\.example\\.com$`)"

# REMOVE service-specific TCP labels when reverting to HTTP routing:
# - "traefik.tcp.routers.netbird-vault.rule=HostSNI(`vault.example.com`)"
```

After removing TCP labels, redeploy the container — Traefik picks up the change dynamically.



When a device stores a stable target URL (for example, a WebDAV or API path) but its apparent source IP may vary because of dual-stack selection, proxy hops, or network changes, prefer matching the exception route on the destination request shape instead of `ClientIP(...)`.

Example: route a scanner's saved WebDAV destination through a special auth-normalizing proxy:

```yaml
http:
  routers:
    scanner-webdav:
      rule: "Host(`files.example.com`) && PathPrefix(`/remote.php/dav/files/alice/inbox`)"
      entryPoints:
        - websecure
      service: scanner-proxy
      priority: 100
      tls: {}
```

This is usually more durable than:

```yaml
rule: "Host(`files.example.com`) && ClientIP(`192.168.1.50`)"
```

Use `ClientIP` only when the source address is the actual invariant you care about. If the real requirement is "requests for this exact stored destination must pass through a special service", encode that with `Host`, `Path`, or `PathPrefix`.

### Use explicit priority when a special-case router overlaps a generic router

If both of these exist:

- a generic router such as `Host(`files.example.com`)`
- a special-case router such as `Host(`files.example.com`) && PathPrefix(`/remote.php/dav/files/alice/inbox`)`

set an explicit higher `priority` on the special-case router. This prevents later edits or rule-length changes from accidentally sending the request back to the generic service.

## References

- [HTTP Routing Overview](https://doc.traefik.io/traefik/reference/routing-configuration/dynamic-configuration-methods/)
- [HTTP Routers](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/router/)
- [HTTP Services](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/services/)
- [EntryPoints](https://doc.traefik.io/traefik/routing/entrypoints/)
- [TCP Routing](https://doc.traefik.io/traefik/reference/routing-configuration/tcp/)
- [UDP Routing](https://doc.traefik.io/traefik/reference/routing-configuration/udp/)
- [IngressRouteTCP CRD](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/tcp/ingressroutetcp/)
- [IngressRouteUDP CRD](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/udp/ingressrouteudp/)
