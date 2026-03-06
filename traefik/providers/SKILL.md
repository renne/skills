---
name: providers
description: Traefik provider configuration covering Docker labels, Kubernetes Ingress and CRDs (IngressRoute, Middleware, TraefikService), file provider (YAML/TOML), and other providers (Consul, etcd). Use when configuring Traefik to discover routes from Docker containers, Kubernetes clusters, configuration files, or other infrastructure components.
---
# Traefik Providers

Source: https://doc.traefik.io/traefik/providers/overview/

## Overview

Providers are infrastructure components from which Traefik discovers routing configuration dynamically. When a provider changes (new container, updated Kubernetes resource, modified file), Traefik automatically applies the new configuration without restarting.

Enable providers in **static configuration** (`traefik.yaml`):

```yaml
providers:
  docker:
    exposedByDefault: false
  kubernetesIngress: {}
  kubernetesCRD: {}
  file:
    directory: /etc/traefik/dynamic/
    watch: true
```

---

## Docker Provider

Traefik watches the Docker socket and reads container labels to create routers, services, and middlewares.

### Static Configuration

```yaml
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false   # Only expose containers with traefik.enable=true
    network: traefik          # Default network for routing
    defaultRule: "Host(`{{ normalize .Name }}.example.com`)"
    watch: true
    swarmMode: false          # Set to true for Docker Swarm
```

### Container Labels

```yaml
# docker-compose.yml
services:
  myapp:
    image: myapp:latest
    labels:
      # Enable this container
      - "traefik.enable=true"

      # Router
      - "traefik.http.routers.myapp.rule=Host(`app.example.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      - "traefik.http.routers.myapp.middlewares=auth@file,rate-limit@file"

      # Service (optional: auto-detected from container port)
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"
      - "traefik.http.services.myapp.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.myapp.loadbalancer.sticky.cookie.name=lb_session"

      # Middleware defined inline (can also be defined in file provider)
      - "traefik.http.middlewares.myapp-auth.basicauth.users=admin:$$apr1$$..."
```

### Docker Label Naming Convention

Labels follow the pattern:
```
traefik.<protocol>.<resource-type>.<name>.<option>[.<sub-option>]=<value>
```

Examples:
```
traefik.http.routers.myrouter.rule=Host(`example.com`)
traefik.http.routers.myrouter.entrypoints=websecure
traefik.http.services.mysvc.loadbalancer.server.port=3000
traefik.http.middlewares.mymw.basicauth.users=admin:hash
traefik.tcp.routers.mydb.rule=HostSNI(`db.example.com`)
traefik.tcp.services.mydb.loadbalancer.server.port=5432
```

> **Note**: Dollar signs (`$`) in password hashes must be escaped as `$$` in Docker Compose labels.

### Docker Swarm

```yaml
providers:
  docker:
    swarmMode: true
    endpoint: "tcp://leader-node:2377"
    swarmModeRefreshSeconds: 15
```

---

## Kubernetes Providers

### Kubernetes Ingress Provider

Use standard Kubernetes `Ingress` resources. Traefik acts as an ingress controller.

```yaml
providers:
  kubernetesIngress:
    ingressClass: traefik       # Only process Ingress with this class
    allowEmptyServices: false
    namespaces:
      - default
      - production
```

```yaml
# Standard Ingress resource
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: "default-auth@kubernetescrd"
    traefik.ingress.kubernetes.io/router.tls.certresolver: "letsencrypt"
spec:
  ingressClassName: traefik
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret
```

### Kubernetes CRD Provider (IngressRoute)

The recommended approach for full Traefik feature support. Provides native Traefik resources as Kubernetes CRDs.

```yaml
providers:
  kubernetesCRD:
    allowCrossNamespace: false   # Allow referencing resources in other namespaces
    allowExternalNameServices: false
    namespaces:
      - default
```

#### IngressRoute (HTTP)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: my-service
          port: 80
          weight: 1
          responseForwarding:
            flushInterval: 1ms
      middlewares:
        - name: auth-middleware
        - name: rate-limit
          namespace: default   # explicit namespace (optional if same ns)
  tls:
    certResolver: letsencrypt
    domains:
      - main: app.example.com
```

#### Middleware CRD

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: auth-middleware
  namespace: default
spec:
  basicAuth:
    secret: my-auth-secret    # K8s Secret with users key

---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 100
    burst: 50

---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

#### TraefikService (Advanced Load Balancing)

```yaml
apiVersion: traefik.io/v1alpha1
kind: TraefikService
metadata:
  name: weighted-service
spec:
  weighted:
    services:
      - name: app-v1
        port: 80
        weight: 80
      - name: app-v2
        port: 80
        weight: 20
```

#### IngressRouteTCP (TCP)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: postgres
spec:
  entryPoints:
    - tcp-9000
  routes:
    - match: HostSNI(`db.example.com`)
      services:
        - name: postgres-service
          port: 5432
  tls:
    passthrough: true
```

#### IngressRouteUDP (UDP)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteUDP
metadata:
  name: dns-udp
spec:
  entryPoints:
    - udp-53
  routes:
    - services:
        - name: coredns
          port: 53
```

#### TLSOption CRD

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: modern-tls
spec:
  minVersion: VersionTLS13
  sniStrict: true
```

#### TLSStore CRD

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default      # "default" is the default TLS store
spec:
  defaultCertificate:
    secretName: wildcard-tls-secret
```

### Cross-Namespace References

Reference a middleware from a different namespace using the `namespace@kubernetescrd` suffix:

```yaml
spec:
  routes:
    - middlewares:
        - name: auth-middleware
          namespace: traefik-system
```

Or in file/Docker label format: `auth-middleware@kubernetescrd`

---

## File Provider

Load dynamic configuration from YAML or TOML files. Supports watching for file changes.

```yaml
providers:
  file:
    directory: /etc/traefik/dynamic/   # Watch all files in directory
    watch: true                         # Hot-reload on change
    # OR single file:
    # filename: /etc/traefik/dynamic.yaml
```

File provider directory structure:
```
/etc/traefik/
├── traefik.yaml           # Static config
└── dynamic/
    ├── routes.yaml        # HTTP routers/services
    ├── middlewares.yaml   # Middleware definitions
    └── tls.yaml           # TLS certificates and options
```

### Templating in File Provider

File provider supports Go templating for dynamic values:

```yaml
http:
  routers:
    my-app:
      rule: "Host(`{{ env \"APP_HOST\" }}`)"
      service: my-app
```

---

## Consul / etcd / ZooKeeper (KV Providers)

Store dynamic configuration in a key-value store:

```yaml
providers:
  consul:
    endpoints:
      - "consul-server:8500"
    rootKey: "traefik"
    token: "my-consul-token"

  etcd:
    endpoints:
      - "etcd-server:2379"
    rootKey: "traefik"
    tls:
      ca: /etc/traefik/certs/ca.crt
      cert: /etc/traefik/certs/client.crt
      key: /etc/traefik/certs/client.key
```

---

## Provider Reference Suffixes

When referencing resources from a specific provider, use the `@provider` suffix:

| Provider | Suffix | Example |
|----------|--------|---------|
| Docker | `@docker` | `my-middleware@docker` |
| Kubernetes CRD | `@kubernetescrd` | `auth@kubernetescrd` |
| Kubernetes Ingress | `@kubernetesingress` | `svc@kubernetesingress` |
| File | `@file` | `rate-limit@file` |
| Internal | `@internal` | `api@internal` |

---

## References

- [Providers Overview](https://doc.traefik.io/traefik/reference/install-configuration/providers/overview/)
- [Docker Provider](https://doc.traefik.io/traefik/providers/docker/)
- [Kubernetes Ingress Provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [Kubernetes CRD Provider](https://doc.traefik.io/traefik/providers/kubernetes-crd/)
- [File Provider](https://doc.traefik.io/traefik/providers/file/)
- [IngressRoute CRD Reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/http/ingressroute/)
- [Middleware CRD Reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/http/middleware/)
- [TraefikService CRD Reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/http/traefikservice/)
