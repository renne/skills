---
name: https-tls
description: Traefik HTTPS and TLS configuration covering automatic certificate management with Let's Encrypt (ACME) using HTTP-01, DNS-01, and TLS-ALPN-01 challenges, manual TLS certificates, TLS options (versions, cipher suites), and wildcard certificates. Use when enabling HTTPS on Traefik, configuring Let's Encrypt for automatic certificates, managing TLS settings, or setting up wildcard certificate support.
---
# Traefik HTTPS & TLS

Source: https://doc.traefik.io/traefik/https/overview/

## Overview

Traefik handles TLS termination at the router level. There are two ways to provide TLS certificates:

1. **Automatic**: Let's Encrypt (ACME) — Traefik requests and renews certificates automatically.
2. **Manual**: Provide certificate files directly in the dynamic configuration.

## Enabling HTTPS on a Router

```yaml
http:
  routers:
    my-app:
      rule: "Host(`example.com`)"
      entryPoints:
        - websecure
      service: my-service
      tls: {}   # Enable TLS with the default certificate resolver
```

To specify a certificate resolver:
```yaml
      tls:
        certResolver: letsencrypt
        domains:
          - main: "example.com"
            sans:
              - "www.example.com"
```

---

## Automatic Certificates with Let's Encrypt (ACME)

Configure `certificatesResolvers` in **static configuration** (`traefik.yaml`):

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /data/acme.json   # Must be a persistent path
      # Choose ONE challenge type:
      httpChallenge:
        entryPoint: web           # HTTP-01 challenge
      # OR
      # tlsChallenge: {}          # TLS-ALPN-01 challenge
      # OR
      # dnsChallenge:
      #   provider: cloudflare    # DNS-01 challenge
```

> **Important**: Mount `/data/` as a persistent volume to retain certificates across restarts. Loss of `acme.json` may cause hitting Let's Encrypt rate limits.

### HTTP-01 Challenge

Let's Encrypt makes an HTTP request to `http://domain/.well-known/acme-challenge/...` to verify domain ownership.

- **Pros**: Simple, no DNS changes needed.
- **Cons**: Requires port 80 to be publicly accessible; cannot issue wildcard certificates.

```yaml
certificatesResolvers:
  http-resolver:
    acme:
      email: admin@example.com
      storage: /data/acme.json
      httpChallenge:
        entryPoint: web
```

### TLS-ALPN-01 Challenge

Let's Encrypt connects to port 443 and verifies a special TLS certificate served during the TLS handshake.

- **Pros**: Only port 443 needed; no port 80 required.
- **Cons**: Cannot issue wildcard certificates.

```yaml
certificatesResolvers:
  tls-resolver:
    acme:
      email: admin@example.com
      storage: /data/acme.json
      tlsChallenge: {}
```

### DNS-01 Challenge

Let's Encrypt verifies domain ownership by checking a DNS TXT record that Traefik creates via your DNS provider's API.

- **Pros**: Works for wildcard certificates (`*.example.com`); works for private/internal services.
- **Cons**: Requires API credentials for your DNS provider.

```yaml
certificatesResolvers:
  dns-resolver:
    acme:
      email: admin@example.com
      storage: /data/acme.json
      dnsChallenge:
        provider: cloudflare         # See list of providers below
        delayBeforeCheck: 30         # Seconds to wait for DNS propagation
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
        disablePropagationCheck: false
```

Set the required environment variables for your DNS provider, for example with Cloudflare:
```bash
CF_API_TOKEN=your-cloudflare-api-token
```

#### Supported DNS Providers (partial list)

| Provider | Code | Environment Variables |
|----------|------|-----------------------|
| Cloudflare | `cloudflare` | `CF_API_TOKEN` or `CF_API_KEY`+`CF_API_EMAIL` |
| Route53 (AWS) | `route53` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| Google Cloud DNS | `gcloud` | `GCE_PROJECT`, `GCE_SERVICE_ACCOUNT_FILE` |
| DigitalOcean | `digitalocean` | `DO_AUTH_TOKEN` |
| Namecheap | `namecheap` | `NAMECHEAP_API_USER`, `NAMECHEAP_API_KEY` |
| OVH | `ovh` | `OVH_ENDPOINT`, `OVH_APPLICATION_KEY`, etc. |
| Hetzner | `hetzner` | `HETZNER_API_KEY` |
| Gandi | `gandi` | `GANDIV5_PERSONAL_ACCESS_TOKEN` |

Full list: https://doc.traefik.io/traefik/https/acme/#providers

### Wildcard Certificates

Wildcard certificates (`*.example.com`) require DNS-01 challenge. Specify them at the router level:

```yaml
http:
  routers:
    wildcard-app:
      rule: "Host(`app.example.com`)"
      entryPoints:
        - websecure
      service: my-service
      tls:
        certResolver: dns-resolver
        domains:
          - main: "example.com"
            sans:
              - "*.example.com"
```

### Staging / Testing

Use Let's Encrypt's staging CA during development to avoid production rate limits:

```yaml
certificatesResolvers:
  staging:
    acme:
      email: admin@example.com
      storage: /data/acme-staging.json
      caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
      httpChallenge:
        entryPoint: web
```

### Let's Encrypt Rate Limits

- 50 certificates per registered domain per week
- 5 duplicate certificate failures per hour
- Use staging environment for testing
- Keep `acme.json` persisted to avoid redundant requests

---

## Manual TLS Certificates

Provide certificates directly in the dynamic configuration. Traefik will use the certificate that matches the SNI of the incoming request.

```yaml
# dynamic/tls.yaml
tls:
  certificates:
    - certFile: /etc/traefik/certs/example.com.crt
      keyFile: /etc/traefik/certs/example.com.key
    - certFile: /etc/traefik/certs/other.com.crt
      keyFile: /etc/traefik/certs/other.com.key
  # Optional: Default certificate for unmatched SNI
  stores:
    default:
      defaultCertificate:
        certFile: /etc/traefik/certs/default.crt
        keyFile: /etc/traefik/certs/default.key
```

---

## TLS Options

Define reusable TLS option profiles to control protocol versions and cipher suites.

```yaml
# dynamic/tls-options.yaml
tls:
  options:
    default:
      minVersion: VersionTLS12
      maxVersion: VersionTLS13
      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
      curvePreferences:
        - CurveP521
        - CurveP384
      sniStrict: true       # Reject requests with unknown SNI
    modern:
      minVersion: VersionTLS13
    legacy:
      minVersion: VersionTLS10
      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_RSA_WITH_AES_256_CBC_SHA
```

Apply a TLS option to a router:
```yaml
http:
  routers:
    my-app:
      tls:
        certResolver: letsencrypt
        options: modern   # Reference TLS option profile by name
```

### TLS Version Constants

| Constant | Description |
|----------|-------------|
| `VersionTLS10` | TLS 1.0 (insecure, avoid) |
| `VersionTLS11` | TLS 1.1 (insecure, avoid) |
| `VersionTLS12` | TLS 1.2 (minimum recommended) |
| `VersionTLS13` | TLS 1.3 (most secure) |

---

## Mutual TLS (mTLS)

Require clients to present a certificate (client authentication).

```yaml
tls:
  options:
    mtls:
      clientAuth:
        caFiles:
          - /etc/traefik/certs/client-ca.crt
        clientAuthType: RequireAndVerifyClientCert
        # Options: NoClientCert, RequestClientCert, RequireAnyClientCert,
        #          VerifyClientCertIfGiven, RequireAndVerifyClientCert
```

Apply to a router:
```yaml
http:
  routers:
    secure-api:
      tls:
        options: mtls
```

---

## Docker Compose Example: Full HTTPS Setup

```yaml
services:
  traefik:
    image: traefik:v3
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-data:/data

  myapp:
    image: myapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`app.example.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"

volumes:
  traefik-data:
```

---

## References

- [HTTPS Overview](https://doc.traefik.io/traefik/https/overview/)
- [Let's Encrypt (ACME)](https://doc.traefik.io/traefik/https/acme/)
- [ACME Certificate Resolver Reference](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/)
- [TLS Certificates](https://doc.traefik.io/traefik/reference/routing-configuration/http/tls/tls-certificates/)
- [TLS Options](https://doc.traefik.io/traefik/reference/routing-configuration/http/tls/tls-options/)
- [DNS Challenge Providers](https://doc.traefik.io/traefik/https/acme/#providers)
