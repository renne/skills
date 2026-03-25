---
name: dns-coredns-integration
description: Practical guide for integrating NetBird DNS with CoreDNS as an authoritative nameserver for internal zones. Explains NetBird's two DNS approaches (Custom Zones for simple records vs. CoreDNS for complex zone management), how to configure NetBird to point peers to a CoreDNS instance for specific domains, and when CoreDNS is still required for advanced hostname/zone management. Use when deciding whether to use NetBird Custom Zones vs. CoreDNS, setting up internal domain resolution for NetBird peers, or integrating NetBird with an existing CoreDNS setup.
---
# NetBird DNS + CoreDNS Integration

## Two Approaches to Internal DNS in NetBird

NetBird provides two ways to serve internal DNS records:

### Option A: NetBird Custom Zones (Recommended for simple use cases)

NetBird's **Custom Zones** feature (available in recent versions) lets you host A, AAAA, and CNAME records directly within NetBird — distributed to peers, no external DNS server needed.

- Create a zone (e.g., `corp.example.com`) in **DNS → Zones**
- Add A/AAAA/CNAME records for internal hosts
- Assign distribution groups — only those peers receive the records
- Custom Zones take precedence over nameserver forwarding for the same domain

Use Custom Zones when you need simple hostname → IP mappings and don't want to run a separate DNS server. See [Custom Zones documentation](https://docs.netbird.io/manage/dns/custom-zones) for full details.

### Option B: Docker Embedded DNS (for Docker container name resolution)

If the Netbird agent runs **inside a Docker container** and only needs to resolve other containers on the same Docker Compose network, Docker's built-in DNS resolver (`127.0.0.11`) is the simplest solution — no CoreDNS required.

Docker natively resolves `<container-name>.<network-name>` (e.g., `traefik.proxy` → `172.0.4.2`). Configure Netbird with:

- A **non-primary** nameserver for domain `<network-name>` pointing to `127.0.0.11:53` with `search_domains_enabled: true`
- A **primary** nameserver (e.g., your central DNS) as catch-all

See the `netbird/dns` skill for the full configuration pattern.

> Use this approach when the goal is Docker container name resolution only. Choose CoreDNS when you need full zone management, SRV/TXT/PTR records, or dynamic DNS.

### Option C: CoreDNS (or other external DNS) via Nameserver Forwarding

For **complex zone management** — zone transfers, SRV/TXT/PTR records, dynamic DNS, DNSSEC, secondary nameservers, or existing infrastructure — you need an external authoritative DNS server.

NetBird's **nameserver forwarder** then distributes the CoreDNS address to all peers automatically.

NetBird's forwarding feature does two things:
1. **Distributes nameserver configuration** to all peers (which DNS server to use for which domain)
2. **Adds search domains** so short hostnames resolve correctly

---

## CoreDNS Integration Pattern

The recommended pattern for CoreDNS-backed zones is:

```
Peer → NetBird DNS forwarder → CoreDNS (authoritative for internal zone)
                                        ↓
                             Returns A/CNAME/SRV/PTR records
```

1. **CoreDNS** is the authoritative server for the internal zone (e.g., `example.com`)
2. **NetBird** is configured with a nameserver entry pointing to CoreDNS for that zone
3. All NetBird peers automatically use CoreDNS for the matching domain

---

## Step-by-Step Setup

### 1. CoreDNS: Authoritative for the Internal Zone

CoreDNS serves the zone using the `file` plugin (zone file) or `hosts` plugin (flat mapping):

**Using `hosts` plugin (simple mappings):**
```corefile
example.com {
    hosts {
        192.168.1.10 server.example.com
        192.168.1.20 nas.example.com
        192.168.1.30 proxy.example.com
        fallthrough
    }
    log
}
```

**Using `file` plugin (full zone file):**
```corefile
example.com {
    file /etc/coredns/zones/example.com.zone
    log
}
```

Zone file (`example.com.zone`):
```zone
$ORIGIN example.com.
$TTL 3600
@   IN SOA ns1 hostmaster (
            2024010101 ; serial
            3600       ; refresh
            900        ; retry
            604800     ; expire
            300 )      ; minimum TTL
    IN NS  ns1
ns1 IN A   192.168.1.1
server IN A 192.168.1.10
nas    IN A 192.168.1.20
proxy  IN A 192.168.1.30
*.internal IN A 192.168.1.1  ; wildcard subdomain
```

**Typical CoreDNS Docker setup** (`/srv/docker/coredns/`):
```yaml
# docker-compose.yml
services:
  coredns:
    image: coredns/coredns
    restart: unless-stopped
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    volumes:
      - ./Corefile:/etc/coredns/Corefile
      - ./zones:/etc/coredns/zones
    command: -conf /etc/coredns/Corefile
```

### 2. NetBird: Configure Nameserver Entry

In the NetBird dashboard → **DNS → Add Nameserver**:

| Field | Value |
|-------|-------|
| Name | `coredns-internal` |
| Nameserver IP | IP of the CoreDNS host on the NetBird mesh (e.g., `100.x.y.z`) or LAN IP |
| Port | `53` |
| Match Domains | `example.com` |
| Primary | No (unless it should handle all queries) |
| Search Domains | Optional: enable so `server` resolves as `server.example.com` |
| Distribution Groups | Group of peers that should use this resolver (e.g., `All`) |

Via MCP (`netbird-create_netbird_nameserver`):
```json
{
  "name": "coredns-internal",
  "nameservers": [{"ip": "100.x.y.z", "ns_type": "udp", "port": 53}],
  "domains": ["example.com"],
  "primary": false,
  "search_domains_enabled": true,
  "enabled": true,
  "groups": ["<group-id-for-all-peers>"]
}
```

---

## What NetBird Custom Zones CAN Do (vs CoreDNS)

| Capability | NetBird Custom Zones | NetBird Nameserver Forwarding | CoreDNS |
|-----------|---------------------|-------------------------------|---------|
| Serve A/AAAA/CNAME records for a zone | ✅ | ❌ | ✅ (file / hosts plugin) |
| Wildcard subdomain resolution | ❌ | ❌ (native) | ✅ (zone file `*` record) |
| PTR records for reverse DNS | ❌ | ❌ | ✅ |
| SRV/TXT records | ❌ | ❌ | ✅ |
| Zone transfers to secondary nameservers | ❌ | ❌ | ✅ |
| DNSSEC | ❌ | ❌ | ✅ |
| DNS name rewrites (`rewrite name`) | ❌ | ❌ | ✅ (rewrite plugin) |
| DNS over TLS (DoT) upstream forwarding | ❌ | ❌ | ✅ (`forward . tls://…`) |
| Query-type view filtering (A vs AAAA) | ❌ | ❌ | ✅ (view plugin) |
| Chained forwarding with fallthrough | ❌ | ❌ | ✅ (`forward { next NXDOMAIN }`) |
| Forward queries to specific servers per domain | ❌ | ✅ | ✅ (forward plugin) |
| Push DNS config to all VPN peers automatically | ✅ | ✅ | ❌ (manual config per host) |
| Search domain distribution | ✅ | ✅ | ❌ (manual config per host) |
| Split DNS (different servers per domain) | ✅ (zones) | ✅ (match domains) | ✅ (multiple server blocks) |
| Group-based access control for DNS records | ✅ (distribution groups) | ❌ | ❌ |

---

## CoreDNS Features with No NetBird Equivalent (Gap Analysis)

This section documents CoreDNS Corefile features that NetBird DNS cannot currently replace. These are the blockers that prevent retiring CoreDNS in favour of NetBird-only DNS.

### 1. DNS Name Rewrites (`rewrite` plugin) — **Critical Blocker**

CoreDNS can rewrite DNS query names before resolution:

```corefile
rewrite name suffix .bartschnet.de .fritz.box  # all *.bartschnet.de → *.fritz.box
rewrite name exact pbs.bartschnet.de pbs.fritz.box
rewrite name regex (.*)-srv\.bartschnet\.de {1}.fritz.box
```

This allows DHCP-assigned hostnames on a router (`fritz.box`) to be accessed via the public domain (`bartschnet.de`) without any static records.

**NetBird has no equivalent.** Custom Zones require a static record per hostname; dynamic DHCP names cannot be rewritten.

**Open issues:**
- **[netbirdio/netbird#5499](https://github.com/netbirdio/netbird/issues/5499)** — "Support rewrite rules in Netbird DNS" (open, filed by `renne`, refs #2660 and #817)
- **[netbirdio/netbird#817](https://github.com/netbirdio/netbird/issues/817)** — "Embedded DNS peer subdomain resolution customization" (open, 20 👍)
- **[netbirdio/netbird#2660](https://github.com/netbirdio/netbird/issues/2660)** — "Custom DNS records" (open, 13 👍)

### 2. DNS over TLS (DoT) Upstream Forwarding

CoreDNS can forward upstream queries over TLS for privacy and ad-blocking:

```corefile
forward . tls://146.255.56.98 tls://146.255.56.99 {
    tls_servername joindns4.eu
    health_check 5s
}
```

NetBird nameservers only support plain UDP/TCP (`ns_type: "udp"`). There is no `tls` nameserver type.

**No known upstream NetBird issue as of 2026-06.** Workaround: keep CoreDNS as the primary resolver and forward DoT from CoreDNS.

### 3. Query-Type View Filtering (`view` plugin)

CoreDNS `view` plugin serves different answers based on the query type or client subnet:

```corefile
public.bartschnet.de {
    view ipv6only { type AAAA }   # return AAAA only to IPv6 clients
    view ipv4only { type A }      # return A only to IPv4 clients
    hosts { … }
}
```

NetBird Custom Zones serve the same records to all peers — no per-query-type or per-client filtering.

**No known upstream NetBird issue as of 2026-06.**

### 4. Chained Forwarding with Fallthrough

CoreDNS can chain multiple upstreams with conditional fallthrough:

```corefile
fritz.box {
    forward . 10.0.0.1 {
        next NXDOMAIN
    }
    forward . 127.0.0.1:5353   # local bridge zone fallback
}
```

NetBird nameservers have a single server list per entry; there is no ordered fallthrough between multiple upstreams.

**No known upstream NetBird issue as of 2026-06.**

### 5. Local-Only DNS Listeners (`bind`)

CoreDNS can bind to specific interfaces/addresses:

```corefile
fritz.box:5353 {
    bind 127.0.0.1
    …
}
```

This allows a local bridge zone only accessible on the same host. NetBird distributes DNS config to peers — it has no concept of a locally-bound-only DNS listener.

### Summary: What CAN Be Migrated to NetBird Today

| CoreDNS usage pattern | NetBird replacement |
|----------------------|-------------------|
| Static A/AAAA records for fixed services (e.g., `pbs.bartschnet.de → 10.0.0.x`) | ✅ Custom Zone record |
| Static CNAME aliases | ✅ Custom Zone CNAME record |
| Simple split-DNS forwarding | ✅ Nameserver with match domains |
| **Dynamic suffix rewrites for DHCP hostnames** | ❌ Not possible yet (#5499) |
| **DoT upstream for ad-blocking/privacy** | ❌ Not supported |
| **View/type-based answer filtering** | ❌ Not supported |
| **Chained forwarding with fallthrough** | ❌ Not supported |

---

## When to Use CoreDNS vs NetBird Custom Zones

Use **NetBird Custom Zones** when you need:
- Simple A/AAAA/CNAME records for internal services
- Group-based access control (different peers see different records)
- No separate DNS infrastructure to manage

Use **CoreDNS** (with NetBird nameserver forwarding) when you need:
- **PTR records** for reverse DNS
- **SRV/TXT records** for service discovery (e.g., Active Directory, Kubernetes)
- **Zone transfers** to secondary nameservers
- **DNSSEC** signing
- **Wildcard subdomain** resolution (e.g., `*.example.com → 192.168.1.1`)
- **Dynamic DNS** updates
- **Existing CoreDNS infrastructure** you want to keep

---

## Netbird Peer DNS Resolution Flow

```
Peer requests: server.example.com
       ↓
NetBird local DNS resolver on peer
       ↓ (checks match-domain rules)
"example.com" → forward to 100.x.y.z:53 (CoreDNS)
       ↓
CoreDNS answers: 192.168.1.10
       ↓
Peer connects to 192.168.1.10
```

---

## Corefile Structure Reference

A typical production Corefile combining forwarding and internal zone:

```corefile
# Internal zone - authoritative
example.com {
    hosts {
        192.168.1.10  server.example.com
        192.168.1.20  nas.example.com
        fallthrough
    }
    forward . 8.8.8.8  # fallback for unresolved names
    cache 300
    log
    errors
}

# All other queries - forward to upstream
. {
    forward . 1.1.1.1 8.8.8.8
    cache 3600
    log
    errors
}
```

---

## Troubleshooting

**Peers not resolving internal names:**
1. Check NetBird nameserver entry is enabled and assigned to correct groups
2. Verify CoreDNS is reachable from peers: `ping <coredns-ip>` then `dig @<coredns-ip> server.example.com`
3. Check `netbird status` on peer to see which DNS config was received

**CoreDNS not responding:**
```bash
# Test directly
dig @localhost server.example.com
# Check container logs
docker logs coredns
```

**Match domain not triggering:**
- Ensure the domain in NetBird exactly matches (e.g., `example.com` not `example.com.`)
- NetBird match domains are suffix-matched: `example.com` also matches `sub.example.com`

---

## Running CoreDNS and NetBird Agent in the Same Container

Both CoreDNS and NetBird's built-in DNS proxy want to bind port 53. To resolve the conflict when the container acts as both a DNS server and a NetBird routing peer:

**Disable NetBird's DNS proxy:**
```bash
netbird up --disable-dns
```

**Also set the group to DNS Unmanaged Mode** in the NetBird dashboard (the group the container's peer belongs to, e.g. "Routing Peers" → DNS settings → Unmanaged). This prevents the management server from pushing DNS config down to the container, which would otherwise re-enable the DNS proxy.

Without both steps, the containers may fight over port 53 on restart.

---

## Catch-All Nameserver for All NetBird Peers

To push a CoreDNS instance as the primary resolver to **all** NetBird peers (covering every domain, not just specific match domains):

```json
POST /api/dns/nameservers
{
  "name": "CoreDNS HA",
  "nameservers": [
    {"ip": "100.x.x.x", "ns_type": "udp", "port": 53},
    {"ip": "100.x.x.y", "ns_type": "udp", "port": 53}
  ],
  "domains": [],
  "primary": true,
  "search_domains_enabled": false,
  "enabled": true,
  "groups": ["<all-peers-group-id>"]
}
```

- `"domains": []` with `"primary": true` = catch-all (all queries routed through these nameservers)
- Use the NetBird overlay IPs (`100.x.x.x`) of the CoreDNS containers so it works for peers regardless of LAN topology
- LAN clients (not NetBird peers) should be configured via the upstream router's DNS settings instead

---

## References

- https://docs.netbird.io/manage/dns
- https://coredns.io/plugins/hosts/
- https://coredns.io/plugins/file/
- https://coredns.io/plugins/forward/
