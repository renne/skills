---
name: https-and-reverse-proxy
description: Vaultwarden HTTPS and reverse proxy setup guide covering enabling HTTPS via reverse proxy (Caddy, nginx) or Rocket TLS, obtaining Let's Encrypt certificates, Cloudflare TLS, OCSP stapling, and validating SSL certificates. Use when setting up HTTPS for Vaultwarden, configuring a reverse proxy, obtaining TLS certificates, or troubleshooting SSL issues with Bitwarden mobile clients.
---
# Vaultwarden HTTPS & Reverse Proxy

Sources: https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS and https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples and https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose

## Overview

HTTPS is required for Vaultwarden because the Bitwarden web vault uses Web Crypto APIs that browsers only expose in HTTPS contexts. The **recommended approach** is to place Vaultwarden behind a reverse proxy that terminates TLS.

---

## Option 1: Reverse Proxy (Recommended)

### Caddy 2.x

Caddy automatically obtains and renews Let's Encrypt certificates.

**Caddyfile:**

```
{$DOMAIN} {
  log {
    level INFO
    output file {$LOG_FILE} {
      roll_size 10MB
      roll_keep 10
    }
  }

  # Get a cert via ACME (Let's Encrypt or ZeroSSL):
  tls {$EMAIL}

  # Or provide your own cert (also use for Cloudflare-proxied setups):
  # tls {$SSL_CERT_PATH} {$SSL_KEY_PATH}

  encode zstd gzip

  reverse_proxy 127.0.0.1:8000 {
    # Pass real client IP to Vaultwarden (needed for fail2ban)
    header_up X-Real-IP {remote_host}
  }
}
```

With Docker Compose, replace `127.0.0.1:8000` with `vaultwarden:80`.

### nginx

```nginx
upstream vaultwarden-default {
  zone vaultwarden-default 64k;
  server 127.0.0.1:8000;
  keepalive 2;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      "";
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name vaultwarden.example.tld;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name vaultwarden.example.tld;

    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    client_max_body_size 525M;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location / {
        proxy_pass http://vaultwarden-default;
    }
}
```

If you get 504 Gateway Timeout errors, add to the `server {}` block:

```nginx
proxy_connect_timeout  777;
proxy_send_timeout     777;
proxy_read_timeout     777;
send_timeout           777;
```

### nginx with Sub-Path

To host Vaultwarden at `https://shared.example.tld/vault/`:

```nginx
# Set DOMAIN env var:
DOMAIN=https://shared.example.tld/vault/

location /vault/ {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://vaultwarden-default;
}
```

---

## Option 2: Rocket TLS (Not Recommended)

Built-in TLS via Rocket is available but immature. Use a reverse proxy instead.

```bash
docker run -d --name vaultwarden \
  -e ROCKET_TLS='{certs="/ssl/certs.pem",key="/ssl/key.pem"}' \
  -v /ssl/keys/:/ssl/ \
  -v /vw-data/:/data/ \
  -p 443:80 \
  vaultwarden/server:latest
```

**Caveats:**
- Must use an **RSA** cert/key (ECC/ECDSA not supported)
- Use `fullchain.pem` (not `cert.pem`) to include the full certificate chain
- Required for Android clients which don't include Let's Encrypt intermediates by default

---

## Getting TLS Certificates

### Let's Encrypt (Free, Recommended)

Requires a domain name pointing to your server.

- **Certbot**: `certbot certonly --standalone -d vaultwarden.example.com`
- **acme.sh**: `acme.sh --issue -d vaultwarden.example.com --webroot /var/www/html`
- **Caddy**: Automatic via ACME (no extra steps)
- **DNS challenge** (for private/LAN instances): Use Duck DNS or another dynamic DNS provider

### Cloudflare

1. Add your domain to Cloudflare
2. Go to **SSL/TLS → Origin Server** and generate an origin certificate (up to 15-year validity)
3. Configure Vaultwarden to use the origin certificate
4. Cloudflare handles client-facing TLS automatically

> For Rocket TLS with Cloudflare origin certs, select **RSA** as the private key type.

---

## Validating Your SSL Certificate

### Check certificate chain (required for Android clients)

```bash
openssl s_client -showcerts -connect vault.example.com:443 -servername vault.example.com
```

Expected output shows depth=2 (root CA), depth=1 (intermediate), depth=0 (your cert).

### Check OCSP stapling (required for mobile app)

```bash
openssl s_client -showcerts -connect vault.example.com:443 -servername vault.example.com -status
```

Look for: `OCSP Response Status: successful (0x0)`

Connecting a mobile app will fail with `Chain validation failed` if OCSP stapling is not configured correctly.

### Online Tools

- [Qualys SSL Labs](https://www.ssllabs.com/ssltest/) — full SSL/TLS analysis
- [Digicert SSL Checker](https://www.digicert.com/help/) — includes OCSP staple status

---

## Security Headers (Optional)

Add to your reverse proxy for improved security:

```
Strict-Transport-Security: max-age=31536000;
X-Content-Type-Options: nosniff
X-Robots-Tag: noindex, nofollow
X-Frame-Options: SAMEORIGIN   # Use SAMEORIGIN (not DENY) if you use FIDO2 WebAuthn
```

---

## References

- [Enabling HTTPS](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS)
- [Proxy examples](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples)
- [Using Docker Compose](https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose)
- [Running a private instance with Let's Encrypt certs](https://github.com/dani-garcia/vaultwarden/wiki/Running-a-private-vaultwarden-instance-with-Let%27s-Encrypt-certs)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
