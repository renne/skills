---
name: middlewares
description: Traefik HTTP middleware reference covering all built-in middlewares including BasicAuth, DigestAuth, ForwardAuth, Headers, IPAllowList, RateLimit, Compress, StripPrefix, AddPrefix, RedirectScheme, RedirectRegex, Retry, CircuitBreaker, Chain, and more. Use when adding authentication, rate limiting, path rewriting, response compression, security headers, redirects, or any request/response transformation to Traefik routers.
---
# Traefik Middlewares

Source: https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/

## Overview

Middlewares are processing steps attached to routers that transform requests before reaching the backend service (and responses on the return path). They are defined centrally and referenced by name in routers.

Middlewares can be configured via:
- **File provider** (YAML/TOML)
- **Docker labels**
- **Kubernetes CRDs** (`Middleware` resource)

### Attaching Middlewares to a Router

```yaml
# File provider
http:
  routers:
    my-router:
      rule: "Host(`example.com`)"
      service: my-service
      middlewares:
        - strip-prefix
        - rate-limit
        - compress
```

```yaml
# Docker label
- "traefik.http.routers.my-router.middlewares=strip-prefix@file,rate-limit@file"
```

```yaml
# Kubernetes CRD - reference a Middleware resource by name (same namespace)
# or namespace@kubernetescrd for cross-namespace
spec:
  routes:
    - middlewares:
        - name: strip-prefix
```

---

## Authentication

### BasicAuth

Restrict access using HTTP Basic Authentication.

```yaml
http:
  middlewares:
    basic-auth:
      basicAuth:
        users:
          - "admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"  # htpasswd format
        usersFile: /etc/traefik/.htpasswd   # optional: load from file
        realm: "My App"                     # optional: realm string
        removeHeader: true                  # strip Authorization header before forwarding
```

Generate a password hash: `htpasswd -nb admin mypassword` or `echo $(htpasswd -nB admin)`

```yaml
# Docker label
- "traefik.http.middlewares.basic-auth.basicauth.users=admin:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"
# Note: $ must be escaped as $$ in Docker labels
```

```yaml
# Kubernetes CRD
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
spec:
  basicAuth:
    secret: my-auth-secret  # Kubernetes Secret with users key in htpasswd format
```

### DigestAuth

Similar to BasicAuth but uses digest authentication (MD5-based challenge-response).

```yaml
http:
  middlewares:
    digest-auth:
      digestAuth:
        users:
          - "admin:traefik:a2688e031edb4be6fe15f1d5f37f56f9"
        usersFile: /etc/traefik/.htdigest
        realm: "My App"
        removeHeader: false
```

### ForwardAuth

Delegate authentication to an external service. Traefik forwards the request to `address`; if the response is 2xx, the original request proceeds. Any other status code is returned to the client.

```yaml
http:
  middlewares:
    forward-auth:
      forwardAuth:
        address: "https://auth.example.com/verify"
        trustForwardHeader: true
        authResponseHeaders:
          - "X-Auth-User"
          - "X-Auth-Role"
        authRequestHeaders:
          - "Authorization"
          - "Cookie"
        tls:
          insecureSkipVerify: false
```

| Option | Description |
|--------|-------------|
| `address` | URL of the authentication service (required) |
| `trustForwardHeader` | Trust existing `X-Forwarded-*` headers |
| `authResponseHeaders` | Headers from auth response to copy to forwarded request |
| `authRequestHeaders` | Headers from original request to copy to auth request |
| `authResponseHeadersRegex` | Regex to select response headers to copy |
| `addAuthCookiesToResponse` | Copy auth cookies back to client |
| `preserveRequestMethod` | Use original HTTP method when calling auth service |
| `maxBodySize` | Max request body size forwarded (-1 = unlimited) |

---

## Headers

### Headers Middleware

Add, remove, or modify HTTP request and response headers. Used for security headers, CORS, HSTS, and custom headers.

```yaml
http:
  middlewares:
    security-headers:
      headers:
        # Request headers
        customRequestHeaders:
          X-Custom-Header: "my-value"
          Authorization: ""           # Remove a header by setting empty string
        # Response headers
        customResponseHeaders:
          X-Frame-Options: "DENY"
          X-Content-Type-Options: "nosniff"
        # Security shortcuts
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
        contentTypeNosniff: true
        browserXssFilter: true
        frameDeny: true
        # CORS
        accessControlAllowOriginList:
          - "https://app.example.com"
        accessControlAllowMethods:
          - "GET"
          - "POST"
          - "OPTIONS"
        accessControlAllowHeaders:
          - "Authorization"
          - "Content-Type"
        accessControlMaxAge: 3600
        addVaryHeader: true
```

---

## Path Manipulation

### StripPrefix

Remove path prefixes before forwarding to the backend.

```yaml
http:
  middlewares:
    strip-prefix:
      stripPrefix:
        prefixes:
          - "/api/v1"
          - "/api/v2"
        forceSlash: false
```

A request to `/api/v1/users` becomes `/users` at the backend.

### StripPrefixRegex

Remove path prefixes matched by a regular expression.

```yaml
http:
  middlewares:
    strip-prefix-regex:
      stripPrefixRegex:
        regex:
          - "/api/v[0-9]+"
```

### AddPrefix

Prepend a path prefix before forwarding.

```yaml
http:
  middlewares:
    add-prefix:
      addPrefix:
        prefix: "/app"
```

### ReplacePath

Replace the request path entirely.

```yaml
http:
  middlewares:
    replace-path:
      replacePath:
        path: "/new-path"
```

### ReplacePathRegex

Replace path using a regular expression and template.

```yaml
http:
  middlewares:
    replace-path-regex:
      replacePathRegex:
        regex: "^/old/(.+)"
        replacement: "/new/$1"
```

---

## Redirects

### RedirectScheme

Redirect requests to a different scheme (e.g., HTTP → HTTPS).

```yaml
http:
  middlewares:
    https-redirect:
      redirectScheme:
        scheme: https
        permanent: true  # 301 redirect; false = 302
```

### RedirectRegex

Redirect requests using a regular expression to match and replace the URL.

```yaml
http:
  middlewares:
    redirect-regex:
      redirectRegex:
        regex: "^http://example.com/(.*)"
        replacement: "https://www.example.com/$1"
        permanent: true
```

---

## Rate Limiting

### RateLimit

Limit the number of requests per time window. Uses a token-bucket algorithm.

```yaml
http:
  middlewares:
    rate-limit:
      rateLimit:
        average: 100     # Requests per second (sustained rate)
        burst: 50        # Max burst above average
        period: 1s       # Time window (default: 1s)
        sourceCriterion:
          ipStrategy:
            depth: 1           # Number of IPs from X-Forwarded-For to skip
            excludedIPs:
              - "127.0.0.1"
```

---

## Traffic Control

### InFlightReq

Limit the number of concurrent in-flight requests.

```yaml
http:
  middlewares:
    in-flight:
      inFlightReq:
        amount: 10
        sourceCriterion:
          requestHeaderName: "X-User-ID"
```

### CircuitBreaker

Open the circuit (stop forwarding) when the backend has too many errors.

```yaml
http:
  middlewares:
    circuit-breaker:
      circuitBreaker:
        expression: "ResponseCodeRatio(500, 600, 0, 600) > 0.30"
        checkPeriod: 10s
        fallbackDuration: 10s
        recoveryDuration: 10s
```

### Retry

Automatically retry failed requests.

```yaml
http:
  middlewares:
    retry:
      retry:
        attempts: 4
        initialInterval: 100ms
```

Use `Retry` narrowly. It works best when attached only to the route that needs protection from short upstream restart windows or connection races, instead of globally across unrelated application traffic.

Example:

```yaml
http:
  routers:
    scanner-webdav:
      rule: "Host(`files.example.com`) && PathPrefix(`/remote.php/dav/files/alice/inbox`)"
      service: scanner-proxy
      middlewares:
        - scanner-retry

  middlewares:
    scanner-retry:
      retry:
        attempts: 4
        initialInterval: 2s
```

If the backend application can also distinguish transient connect failures from real authentication failures, combine Traefik `Retry` with a small application-level retry for connection-refused or connect-timeout errors only.

### Buffering

Buffer requests/responses for better error handling and size limiting.

```yaml
http:
  middlewares:
    buffering:
      buffering:
        maxRequestBodyBytes: 10485760   # 10 MB
        memRequestBodyBytes: 2097152    # 2 MB in memory, rest on disk
        maxResponseBodyBytes: 10485760
        memResponseBodyBytes: 2097152
        retryExpression: "IsNetworkError() && Attempts() < 2"
```

---

## Compression

### Compress

Enable gzip or Brotli response compression (when client supports it).

```yaml
http:
  middlewares:
    compress:
      compress:
        excludedContentTypes:
          - "image/jpeg"
          - "image/png"
        defaultEncoding: "gzip"   # default when client accepts both gzip and br
        minResponseBodyBytes: 1024
```

---

## Security

### IPAllowList

Restrict access to a list of allowed IP addresses or CIDR ranges.

```yaml
http:
  middlewares:
    ip-allowlist:
      ipAllowList:
        sourceRange:
          - "127.0.0.1"
          - "192.168.1.0/24"
          - "10.0.0.0/8"
        ipStrategy:
          depth: 2    # Trust X-Forwarded-For depth (useful behind load balancers)
```

### PassTLSClientCert

Pass TLS client certificate information as headers to the backend.

```yaml
http:
  middlewares:
    pass-tls-cert:
      passTLSClientCert:
        pem: true
        info:
          notAfter: true
          notBefore: true
          sans: true
          subject:
            country: true
            commonName: true
            organization: true
```

---

## Error Handling

### Errors

Return a custom error page for specific HTTP status codes.

```yaml
http:
  middlewares:
    error-pages:
      errors:
        status:
          - "404"
          - "500-599"
        service: error-service
        query: "/{status}.html"
```

---

## Middleware Composition

### Chain

Group multiple middlewares into a single named middleware chain for reuse.

```yaml
http:
  middlewares:
    default-chain:
      chain:
        middlewares:
          - https-redirect
          - security-headers
          - rate-limit
          - compress
```

Reference the chain in a router:
```yaml
http:
  routers:
    my-app:
      middlewares:
        - default-chain
```

---

## Middleware Reference Table

| Middleware | Category | Description |
|------------|----------|-------------|
| `addPrefix` | Path | Prepend a prefix to the path |
| `basicAuth` | Auth | HTTP Basic Authentication |
| `buffering` | Traffic | Buffer and limit request/response sizes |
| `chain` | Composition | Group middlewares into a named chain |
| `circuitBreaker` | Traffic | Open circuit on backend errors |
| `compress` | Compression | gzip/Brotli response compression |
| `contentType` | Headers | Auto-detect and set Content-Type |
| `digestAuth` | Auth | HTTP Digest Authentication |
| `errors` | Error | Custom error pages |
| `forwardAuth` | Auth | External authentication service |
| `grpcWeb` | Protocol | gRPC-Web to gRPC translation |
| `headers` | Headers | Add/modify/remove HTTP headers |
| `ipAllowList` | Security | IP-based access control |
| `inFlightReq` | Traffic | Limit concurrent requests |
| `passTLSClientCert` | Security | Pass TLS client cert to backend |
| `rateLimit` | Traffic | Token-bucket rate limiting |
| `redirectRegex` | Redirect | Regex-based URL redirect |
| `redirectScheme` | Redirect | HTTP → HTTPS redirect |
| `replacePath` | Path | Replace the request path |
| `replacePathRegex` | Path | Regex-based path replacement |
| `retry` | Traffic | Auto-retry failed requests |
| `stripPrefix` | Path | Remove path prefixes |
| `stripPrefixRegex` | Path | Regex-based prefix removal |

## References

- [Middleware Overview](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/)
- [BasicAuth](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/)
- [ForwardAuth](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/forwardauth/)
- [Headers](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/headers/)
- [RateLimit](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/ratelimit/)
- [Compress](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/compress/)
- [StripPrefix](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/stripprefix/)
- [RedirectScheme](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/redirectscheme/)
- [CircuitBreaker](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/circuitbreaker/)
- [IPAllowList](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/ipallowlist/)
