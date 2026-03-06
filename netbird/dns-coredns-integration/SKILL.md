---
name: dns-coredns-integration
description: Practical guide for integrating NetBird DNS with CoreDNS as an authoritative nameserver for internal zones. Explains that NetBird DNS is a forwarder (not an authoritative server), how to configure NetBird to point peers to a CoreDNS instance for specific domains, and when you must keep CoreDNS for hostname/subdomain mappings. Use when deciding whether to replace CoreDNS hostname mappings with NetBird DNS, setting up internal domain resolution for NetBird peers, or integrating NetBird with an existing CoreDNS setup.
---
# NetBird DNS + CoreDNS Integration

## Key Concept: NetBird DNS is a Forwarder, Not an Authoritative Server

**NetBird DNS cannot host DNS zones or serve A/CNAME records directly.**

NetBird's DNS management feature does two things:
1. **Distributes nameserver configuration** to all peers (which DNS server to use for which domain)
2. **Adds search domains** so short hostnames resolve correctly

It does **not** run an authoritative DNS server. To serve hostname → IP mappings for a zone (e.g., `example.com`), you need an external authoritative DNS server such as CoreDNS, BIND, or PowerDNS.

---

## Integration Pattern

The recommended pattern for internal zones is:

```
Peer → NetBird DNS forwarder → CoreDNS (authoritative for internal zone)
                                        ↓
                             Returns A/CNAME records
```

1. **CoreDNS** remains the authoritative server for the internal zone (e.g., `example.com`)
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

## What NetBird DNS CAN Do (vs CoreDNS)

| Capability | NetBird DNS | CoreDNS |
|-----------|-------------|---------|
| Forward queries to specific servers for domains | ✅ | ✅ (forward plugin) |
| Serve A/CNAME records for a zone | ❌ | ✅ (file / hosts plugin) |
| Wildcard subdomain resolution | ❌ (native) | ✅ (zone file `*` record) |
| Push DNS config to all VPN peers automatically | ✅ | ❌ (manual config per host) |
| Search domain distribution | ✅ | ❌ (manual config per host) |
| Split DNS (different servers per domain) | ✅ | ✅ (multiple server blocks) |

---

## When to Keep CoreDNS

Keep CoreDNS as the authoritative server when you need:
- **Static A/CNAME records** for hostnames in a zone
- **Wildcard subdomain** resolution (e.g., `*.example.com → 192.168.1.1`)
- **PTR records** for reverse DNS
- **SRV/TXT records** for service discovery
- **Zone transfers** to secondary nameservers

Use NetBird DNS to **distribute** the CoreDNS address to all peers automatically instead of manually configuring each host.

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

## References

- https://docs.netbird.io/manage/dns
- https://coredns.io/plugins/hosts/
- https://coredns.io/plugins/file/
- https://coredns.io/plugins/forward/
