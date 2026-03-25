---
name: dns
description: NetBird DNS management covering nameserver configuration, match domains, primary nameservers, search domains, Custom Zones (hosting A/AAAA/CNAME records directly in NetBird), DNS settings and management modes, DNS aliases for routed networks, split-horizon DNS, and DNS troubleshooting. Use when setting up DNS resolution for internal domains, creating private DNS records without external DNS servers, configuring DNS management modes per peer group, resolving DNS issues in a NetBird network, or using friendly DNS names for private network resources.
---
# NetBird DNS Management

## Key Concepts

- **Nameserver**: A DNS server that NetBird distributes to peers. Configured in the dashboard under **DNS → Nameservers**.
- **Match Domains**: Domain suffixes resolved by a specific nameserver (e.g., `company.internal`). Adding a domain automatically matches all its subdomains.
- **Primary Nameserver**: The default nameserver that handles all domains not matched by specific rules. Configured by leaving match domains empty.
- **Search Domains**: Domain suffixes appended to unqualified hostnames. Requires NetBird v0.24.0+.
- **Custom Zones**: Private DNS zones hosted within NetBird itself (A/AAAA/CNAME records), distributed to peers — no external DNS server needed.
- **DNS Management Mode**: Whether NetBird controls DNS settings on a peer (Managed) or leaves them unchanged (Unmanaged).

### Automatic Peer Domain Names

Every peer is automatically assigned a DNS name in the form `<peer-name>.netbird.cloud` (cloud) or `<peer-name>.<your-domain>` (self-hosted). Available since NetBird v0.11.0.

---

## Architecture

NetBird deploys a **local DNS resolver** on each peer. The management service acts as a DHCP-like authority only for the NetBird overlay (`100.64.0.0/10` CGNAT range). Your LAN DHCP and DNS remain unchanged.

The local resolver runs on `100.x.255.254:53` (userspace mode), where `x` is the second octet of your account's /16 block.

### DNS Query Routing

1. Check if query matches a specific nameserver's match domains → forward to that nameserver
2. If no match → forward to primary nameserver
3. Return the result to the application

**Example** (primary: Cloudflare 1.1.1.1; match domain: `company.internal` → 10.0.0.1):

```
"google.com"          → no match → 1.1.1.1 (public IP)
"web.company.internal" → matches  → 10.0.0.1 (private IP)
"server"              → expanded to server.company.internal → 10.0.0.1
```

### Default Behavior (No Nameservers Configured)

| Platform | Behavior |
|----------|----------|
| Linux | Always sets up DNS; peer domains (`my-server.netbird.cloud`) work automatically; original nameservers preserved as upstream |
| macOS / Windows / Mobile | DNS not modified; peer domains won't resolve without custom nameservers configured |

---

## Internal DNS Servers (Nameserver Configuration)

### When You Don't Need a Nameserver

To expose a few internal resources by domain name, create a **Network resource** with the domain name (e.g., `fileserver.corp.local`) and an access control policy. The routing peer resolves DNS on behalf of clients. No nameserver configuration needed.

### Configuring a Nameserver

Dashboard → **DNS → Nameservers → Add Nameserver → Custom DNS**:

1. Add one or more nameserver IPs (e.g., `10.0.0.1`)
2. Select **Distribution Groups** — peer groups that should use this nameserver
3. Under **Domains**, add a **Match Domain** (e.g., `corp.local`, `company.internal`)
4. Enable **Search Domains** toggle to allow short names (`server` → `server.corp.local`)
5. Save

> Configure 2–3 nameserver IPs per entry for automatic failover.

### Primary vs Match Domain

| Type | Configuration | Handles |
|------|--------------|---------|
| **Match domain** | Domain(s) listed under Domains | Queries for those domains (and all subdomains) |
| **Primary** | Leave Domains empty | All queries not matched by other nameservers |

Each peer should have exactly **one** primary nameserver. Multiple nameservers matching a query: the **most specific match domain wins**.

### Split-Horizon DNS Example

**Primary (internet)**: Cloudflare `1.1.1.1`, `1.0.0.1` — no match domains — Distribution: "All Peers"

**Internal match**: Custom DNS `10.0.0.1`, `10.0.0.2` — match domain `company.internal` — Distribution: "All Peers" — search domains enabled

Result: `google.com` → Cloudflare; `app.company.internal` → internal DNS; `server` → `server.company.internal` → internal DNS

### Docker Embedded DNS for Container Name Resolution

Docker provides a built-in DNS resolver at `127.0.0.11` inside every container. It natively resolves `<container-name>.<network-name>` to the container's IP on that network — for example, `traefik.proxy` → `172.0.4.2`.

To make a Netbird agent container resolve Docker container names without CoreDNS or a hosts file:

1. Create a **non-primary** nameserver for the Docker network name as match domain, pointing to `127.0.0.11:53`
2. Enable **Search Domains** so bare names like `traefik` expand to `traefik.proxy`
3. Scope both nameservers (Docker embedded + central catch-all) to the agent peer group

```
# Nameserver: "Docker Embedded DNS"
# IPs: 127.0.0.11 port 53
# primary: false
# domains: ["proxy"]      ← name of the Docker network
# search_domains_enabled: true
# distribution_groups: ["docker-agent-peers"]

# Nameserver: "Central DNS"
# IPs: 10.0.0.7, 10.0.0.8 port 53
# primary: true            ← catch-all
# domains: []
# distribution_groups: ["docker-agent-peers"]
```

> ⚠️ **`127.0.0.11` cannot be the primary catch-all.** Netbird captures the original nameserver before takeover and assigns it priority -100. It will only work as a domain-specific non-primary nameserver.

> ℹ️ This eliminates the need for CoreDNS, hosts files, or cron jobs when the only requirement is resolving peer containers on a specific Docker network.

### Private DNS Behind a Routing Peer

If the internal DNS server is on a private network:

1. Create a **Network** with the DNS server's IP as a resource (e.g., `10.0.0.53/32`)
2. Add the routing peer that can reach the private network
3. Create an **Access Control Policy**: source = user groups, destination = DNS resource group, UDP port 53
4. Add the routing peer's group to the nameserver's **Distribution Groups** so the routing peer itself can resolve domain resources

### Active Directory / Domain Controllers

Do **not** use Domain Controllers as routing peers. If required:

1. Create a dedicated DC group (e.g., "Domain Controllers")
2. Go to **DNS → DNS Settings → Disable DNS management for these groups**
3. Add the DC group and save

This prevents NetBird from interfering with AD DNS services while still allowing the DC to participate as a routing peer.

---

## Custom Zones

NetBird's **Custom Zones** let you host private DNS records directly within NetBird — no external DNS server required. Records are distributed to selected peer groups.

> Custom Zones **take precedence** over nameservers. If both a Custom Zone and a nameserver match the same domain, the Custom Zone answers first.

### Supported Record Types

| Type | Description |
|------|-------------|
| `A` | IPv4 address |
| `AAAA` | IPv6 address |
| `CNAME` | Alias to another domain |

### Creating a Custom Zone

Dashboard → **DNS → Zones → Add Zone**:

| Field | Description |
|-------|-------------|
| **Domain** | FQDN for the zone, e.g. `services.company.internal`. Cannot be changed after creation. Must not conflict with the peer DNS domain. |
| **Distribution Groups** | Peer groups that receive this zone |
| **Enable Search Domain** | When on, allows `api` to resolve as `api.services.company.internal` |
| **Enable DNS Zone** | Toggle to activate/deactivate distribution without losing records |

### Adding Records

Click the zone → **Add Record**:

| Field | Description |
|-------|-------------|
| **Hostname** | Short name within the zone (e.g., `server` → `server.dev.local`) |
| **Record Type** | `A`, `AAAA`, or `CNAME` |
| **Value** | IPv4/IPv6 address or target domain |
| **TTL** | Cache lifetime in seconds (default: 300) |

### Limitations

- Zone domain cannot be changed after creation (delete and recreate if needed)
- CNAME records cannot coexist with A/AAAA records for the same hostname
- Empty zones (no records) are not distributed to peers
- **Nameserver upstream protocol**: only plain UDP/TCP is supported — DNS over TLS (DoT) or DNS over HTTPS (DoH) upstreams are not available. No known upstream issue as of 2026-06; use an intermediate resolver (e.g. CoreDNS with `forward . tls://…`) if you need encrypted upstream forwarding.
- **No DNS name rewrites**: dynamic suffix/regex rewrites (like CoreDNS `rewrite name suffix`) are not supported. See [#5499](https://github.com/netbirdio/netbird/issues/5499) / [#817](https://github.com/netbirdio/netbird/issues/817) / [#2660](https://github.com/netbirdio/netbird/issues/2660).
- **No wildcard records in Custom Zones**: `*.example.com` entries are not resolved; each subdomain requires its own record. See [#5170](https://github.com/netbirdio/netbird/issues/5170).

---

## DNS Aliases for Routed Networks

Combine **Custom Zones** with **Networks** to provide friendly DNS names for resources behind routing peers.

**Example**: PostgreSQL at `192.168.0.68` and internal Wiki at `192.168.0.78` in a private data center.

### Step 1 – Create Custom Zone

DNS → Zones → Add Zone:
- Domain: `netbird.internal`
- Distribution Groups: `dev`
- Enable Search Domain: on

### Step 2 – Add DNS Records

| Hostname | Type | Value |
|----------|------|-------|
| `postgres` | A | `192.168.0.68` |
| `wiki` | A | `192.168.0.78` |

### Step 3 – Configure Network Routing

Networks → Add Network → Add Resources:
- Resource `postgres.netbird.internal` with routing peer in data center
- Resource `wiki.netbird.internal` with routing peer in data center

Create Access Control Policy: source = `dev` group, destination = resource groups, TCP all ports.

### Step 4 – Verify

```bash
netbird networks ls
# Lists domain-based resources and their status

psql -h postgres.netbird.internal -U admin
curl http://wiki
```

---

## DNS Settings (Management Modes)

Dashboard → **DNS → DNS Settings**

### Managed Mode (Default)

NetBird configures system DNS. All queries route through NetBird's local resolver. Configured nameservers apply.

### Unmanaged Mode

NetBird does **not** modify system DNS. Peer uses its original DNS configuration. All configured nameservers are ignored. Peer connectivity still works.

Use unmanaged mode when a peer has conflicting VPN/DNS requirements or corporate policy requires specific DNS settings.

### Disabling DNS Management for a Group

1. DNS → DNS Settings
2. Add the group to the disabled list
3. Save Changes (takes effect in 10–30 seconds)

### Client-Side Override

```bash
# Disable DNS management on this peer (overrides server settings)
netbird up --disable-dns

# Re-enable
netbird up --disable-dns=false
```

The `--disable-dns` flag takes precedence over server-side DNS settings.

### API

```bash
# Get current settings
curl -X GET https://api.netbird.io/api/dns/settings \
  -H "Authorization: Token <TOKEN>"

# Update: disable DNS management for specific groups
curl -X PUT https://api.netbird.io/api/dns/settings \
  -H "Authorization: Token <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"disabled_management_groups": ["<group-id>"]}'
```

---

## Platform Notes

| Platform | DNS Setup | Notes |
|----------|-----------|-------|
| Linux | `/etc/resolv.conf` or `resolvectl` | Always configures DNS; original servers preserved as upstream |
| macOS | System APIs | Does **not** modify `/etc/resolv.conf` directly |
| Windows | Network adapter DNS | Sets adapter DNS to the local NetBird resolver |
| Android | VPN DNS config | Disable **Private DNS** in Settings → Network & Internet for custom DNS to work |

Match domain nameservers require macOS, Windows 10+, or Linux with `systemd-resolved`. Always configure at least one primary nameserver assigned to all peers as a fallback.

### Linux Bootstrap Caveat

On Linux, NetBird may manage `/etc/resolv.conf` directly instead of going through `systemd-resolved`. In that mode:

- `/etc/resolv.conf` can contain only the local NetBird resolver, for example `100.115.218.142`
- NetBird typically keeps a backup at `/etc/resolv.conf.original.netbird`
- host DNS can fail during startup if the local NetBird resolver path is not ready yet

For infrastructure hosts, keep the NetBird resolver first for overlay/private records, but add public fallback resolvers after it:

```conf
search nb.example.internal example.internal
nameserver 100.115.218.142
nameserver 86.54.11.100
nameserver 86.54.11.200
```

If NetBird keeps rewriting `/etc/resolv.conf`, persist the fallback via a systemd hook such as `ExecStartPost` on `netbird.service`.

---

## DNS Troubleshooting

### Quick Diagnostics

```bash
# 1. NetBird connected?
netbird status -d

# 2. What nameservers are configured on this peer?
netbird status | grep -A 5 Nameserver

# 3. Check system resolver
# Linux:
cat /etc/resolv.conf
resolvectl status
# macOS:
scutil --dns
# Windows:
Get-DnsClientServerAddress

# 4. Nameserver reachable?
ping 10.0.0.53   # replace with your nameserver IP

# 5. Direct query test (bypasses system resolver)
dig @10.0.0.53 app.internal.company.com

# 6. System resolver test
resolvectl query app.internal.company.com   # Linux
nslookup app.internal.company.com            # all platforms

# 7. Public DNS still works?
nslookup google.com
```

**Quick isolation test**: disable NetBird DNS to check if the issue is NetBird-related:

```bash
netbird down && netbird up --disable-dns
# If the problem goes away → NetBird DNS config issue
# Re-enable: netbird down && netbird up --disable-dns=false
```

### Bootstrap Failure Pattern on Self-Hosted Nodes

If a host both runs NetBird and starts critical services that need public DNS, watch for this chain:

1. service starts
2. service needs DNS immediately
3. resolver path depends on the local NetBird resolver
4. local NetBird DNS path is not ready
5. service fails before NetBird-dependent DNS recovers

Mitigations:

- keep the NetBird resolver first for private overlay records
- add public fallback resolvers on the host
- if a Docker container still fails through `127.0.0.11`, use an explicit per-service `dns:` override

### ⚠️ DNS Hijacking: Primary Nameserver Pitfall

**Symptom**: After WireGuard session keys expire (ENOKEY) or VPN peers disconnect, ALL internet DNS resolution fails — not just internal domains.

**Cause**: A nameserver group configured with `primary: true` and **empty domains** captures ALL DNS queries and routes them through VPN-only servers (e.g., `10.0.0.7:53`, `10.0.0.8:53`). When the WireGuard tunnel loses its session key (`ENOKEY`), those VPN-only servers become unreachable — taking ALL DNS with them.

**Diagnosis**:
```bash
# On the peer: check which interfaces/domains get which nameservers
resolvectl status

# Look for wt0 with no domain restrictions (matches everything):
# Link 5 (wt0)
#   DNS Domain: ~.   ← "~." means catch-all for ALL queries  ← PROBLEM

# Via Netbird MCP or API: list nameserver groups
# Look for any group where primary=true and domains=[]
```

**Fix**: In the Netbird dashboard (or via API/MCP), change the nameserver that should only handle internal domains:
- Set `primary: false`
- Set `domains: ["yourdomain.com"]` — only resolve your internal domain(s) through this nameserver
- Leave public DNS to the peer's own system resolver

**Via MCP**:
```
netbird-update_netbird_nameserver(
  nameserver_id="<id>",
  primary=false,
  domains=["yourdomain.com"],
  nameservers=[...],  # must include even if unchanged
  groups=[...],       # must include even if unchanged
)
```

**Note**: `nameservers` array is required in every update call even when not changing IPs — omitting it causes a 422 error.

**Verify fix**:
```bash
resolvectl status wt0
# Should show specific domains only (e.g., bartschnet.de nb.bartschnet.de ~115.100.in-addr.arpa)
# Should NOT show ~. (catch-all)
```

### Common Issues

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| Can't resolve internal domains | Wrong/missing match domain nameserver, or peer not in distribution group | Check nameserver config and distribution groups in dashboard |
| Public DNS broken | No primary nameserver configured or primary unreachable | Add primary nameserver (empty match domains) assigned to all peers |
| **All DNS fails when WireGuard loses key** | Nameserver with `primary=true` and `domains=[]` routes ALL DNS through VPN-only servers | Set `primary=false` and specify exact match domains — see DNS Hijacking pitfall above |
| Android ignores custom DNS | Private DNS is enabled | Settings → Network & Internet → Private DNS → Off |
| Match domains not working | Using macOS/Windows without a primary nameserver | Configure at least one primary nameserver |
| Routing peer can't resolve domain resources | Routing peer's group not in nameserver distribution | Add routing peer's group to the nameserver's distribution groups |
| Containerized service fails DNS while host DNS works | Docker embedded resolver `127.0.0.11` is unhealthy or forwarding incorrectly | Add explicit `dns:` entries on the affected service and test container namespace resolution separately |

### Wildcard Syntax Note

```
# ❌ Wrong: wildcard syntax not supported in match domains
"domains": ["*.company.internal"]

# ✅ Correct: single domain automatically matches all subdomains
"domains": ["company.internal"]
```

---

## DNS Forwarder on Routing Peers

When a Network resource uses a domain name, the routing peer runs a **DNS forwarder** that resolves the domain on behalf of clients.

- Port: `22054` (changed from `5353` in v0.59.0 to avoid mDNS conflicts; `5353` used if any peer runs below v0.59.0)
- Test: `nslookup -port=22054 <domain> <routing-peer-ip>`

NetBird attempts to open this port automatically on routing peer firewalls, but manual configuration may be needed on some systems.

---

## References

- [DNS in NetBird](https://docs.netbird.io/manage/dns)
- [Internal DNS Servers](https://docs.netbird.io/manage/dns/internal-dns-servers)
- [DNS Settings](https://docs.netbird.io/manage/dns/dns-settings)
- [Custom Zones](https://docs.netbird.io/manage/dns/custom-zones)
- [DNS Aliases for Routed Networks](https://docs.netbird.io/manage/dns/dns-aliases-for-routed-networks)
- [DNS Troubleshooting](https://docs.netbird.io/manage/dns/troubleshooting)
- [API Reference](https://docs.netbird.io/ipa/resources/dns)
