---
name: smtp-configuration
description: Vaultwarden SMTP email configuration guide covering environment variables, security modes (STARTTLS, force_tls, off), provider-specific settings for Gmail, Outlook, Sendgrid, OAuth2 proxy workaround, sendmail integration, special character escaping in passwords, and connectivity troubleshooting. Use when configuring email delivery for Vaultwarden invitations, account recovery, 2FA notifications, or troubleshooting SMTP connection issues.
---
# Vaultwarden SMTP Configuration

Sources: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration

## Overview

Vaultwarden can send emails for:
- User invitations
- Account recovery
- Two-step login notifications
- Emergency access alerts

Configure SMTP via environment variables.

---

## Basic Configuration

```bash
docker run -d --name vaultwarden \
  -e SMTP_HOST=smtp.domain.tld \
  -e SMTP_FROM=vaultwarden@domain.tld \
  -e SMTP_PORT=587 \
  -e SMTP_SECURITY=starttls \
  -e SMTP_USERNAME=myusername \
  -e SMTP_PASSWORD=MyPassw0rd \
  -e DOMAIN=https://vault.example.com \
  -v /vw-data/:/data/ \
  -p 80:80 \
  vaultwarden/server:latest
```

> `DOMAIN` is required for invitation links to be generated correctly.

---

## SMTP Security Modes

| `SMTP_SECURITY` | Port default | Description |
|-----------------|-------------|-------------|
| `starttls` | 587 | Starts unencrypted, upgrades to TLS (TLSv1.1+ only). **Default.** |
| `force_tls` | 465 | Implicit TLS from the start (SSL) |
| `off` | 25 | No encryption (opportunistic). **Insecure — only if you know what you're doing.** |

### Port selection guidance

- Port **465** → use `SMTP_SECURITY=force_tls`
- Port **587** (or sometimes 25) → use `SMTP_SECURITY=starttls`
- Port **25** (no auth) → use `SMTP_SECURITY=off`

---

## Provider-Specific Settings

### Gmail

Generate an **App Password** at https://support.google.com/accounts/answer/185833

```env
# Full SSL (port 465)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_SECURITY=force_tls
SMTP_FROM=user@gmail.com
SMTP_USERNAME=user@gmail.com
SMTP_PASSWORD=your-app-password

# StartTLS (port 587)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_FROM=user@gmail.com
SMTP_USERNAME=user@gmail.com
SMTP_PASSWORD=your-app-password
```

> If App Passwords are unavailable (organizational restrictions), use the [OAuth2 proxy](#oauth2-support) workaround.

### Hotmail / Outlook / Office 365

> **Warning:** Microsoft has switched to OAuth2-only authentication. Direct SMTP password login no longer works. Use the [OAuth2 proxy](#oauth2-support) workaround.

```env
SMTP_HOST=smtp-mail.outlook.com
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_FROM=user@outlook.com
SMTP_USERNAME=user@outlook.com
SMTP_PASSWORD=MyPassw0rd
SMTP_AUTH_MECHANISM=Login
```

### SendGrid

Replace `<full-api-key>` with your API key (starts with `SG.`). The API key requires full **Mail Send** permission.

```env
# StartTLS
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_USERNAME=apikey
SMTP_PASSWORD=<full-api-key>
SMTP_AUTH_MECHANISM=Login
SMTP_FROM=user@domain.tld

# Full SSL
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=465
SMTP_SECURITY=force_tls
SMTP_USERNAME=apikey
SMTP_PASSWORD=<full-api-key>
SMTP_AUTH_MECHANISM=Login
SMTP_FROM=user@domain.tld
```

### Free SMTP Services

| Service | Free Tier |
|---------|-----------|
| [MailJet](https://www.mailjet.com/) | 200 emails/day |
| [Brevo (SendinBlue)](https://www.brevo.com) | 300 emails/day |
| [SMTP2GO](https://www.smtp2go.com/) | 1000 emails/month |

---

## HELO Hostname

By default, the machine hostname is used in the SMTP HELO command. Override it:

```env
HELO_NAME=mail.example.com
```

---

## Passwords with Special Characters

Wrap passwords in single quotes and escape `\`, `'`, `"` characters.

Example: password `~^",a.%\,'}b&@|/c!1(#}`
becomes: `'~^",a.%\\,\'}b&@|/c!1(#}'`

Rules:
- Surround with single quotes `'...'`
- Escape `\` as `\\`
- Escape `'` as `\'`

---

## OAuth2 Support

If you see: `No compatible authentication mechanism was found`

Microsoft and Google (in some cases) require OAuth2, which Vaultwarden doesn't natively support. The recommended workaround is [email-oauth2-proxy](https://github.com/simonrob/email-oauth2-proxy):

1. Set up `email-oauth2-proxy` as a local SMTP relay
2. Point Vaultwarden's `SMTP_HOST` to the proxy (e.g., `localhost`)
3. The proxy handles OAuth2 token exchange with the mail provider

---

## Using `sendmail` (Without Docker)

For a non-Docker installation with an existing Postfix/Sendmail setup:

1. In `/etc/vaultwarden.env`, set `USE_SENDMAIL=true` and `SMTP_FROM=user@example.com`
2. Add `vaultwarden` user to the `postdrop` group: `gpasswd -a vaultwarden postdrop`
3. Edit the systemd service (`systemctl edit vaultwarden`) and add to `[Service]`:

```ini
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_LOCAL AF_NETLINK
ReadWritePaths=/var/lib/vaultwarden /var/log/vaultwarden.log /var/spool/postfix/maildrop
```

4. Restart: `systemctl restart vaultwarden`

---

## Troubleshooting

Test SMTP connectivity from the host or container:

```bash
# Check HTTPS connectivity (baseline test)
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/www.google.com/443'; echo $?

# Test port 587
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/smtp.gmail.com/587'; echo $?

# Test port 465
timeout 5 bash -c 'cat < /dev/null > /dev/tcp/smtp.gmail.com/465'; echo $?

# Using nc (if available)
nc -vz smtp.gmail.com 587
```

Test from inside the container:

```bash
docker exec -it vaultwarden sh
# Then run the commands above
```

**Return value of 0** = connectivity OK; anything else = connection blocked (check firewall, ISP blocking).

---

## References

- [SMTP Configuration](https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration)
- [email-oauth2-proxy](https://github.com/simonrob/email-oauth2-proxy)
- [Enabling mobile client push notifications](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-Mobile-Client-push-notification)
