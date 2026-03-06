---
name: networks-and-dns
description: NetBird network routes and DNS management covering site-to-site routing, network resources, routing peers, high availability, masquerading, DNS nameservers, match domains, and search domains. Use when setting up access to subnets or LANs via NetBird, configuring site-to-site VPN between offices or cloud VPCs, or managing internal DNS resolution for peers.
---
# NetBird Networks and DNS

Sources: https://docs.netbird.io/manage/networks and https://docs.netbird.io/manage/dns

## Overview

NetBird enables connectivity beyond peer-to-peer connections through two complementary features:

- **Networks** (formerly Network Routes): route traffic to entire subnets or LANs via designated routing peers — so devices that do not run NetBird can still be accessed.
- **DNS Management**: configure custom nameservers and domain routing so peers resolve internal hostnames correctly.

---

## Networks (Subnet Routing)

### Concepts

| Term | Description |
|------|-------------|
| **Network** | A named group of resources (IP ranges or domain-based routes) accessible through routing peers |
| **Routing peer** | A NetBird peer that forwards traffic between the overlay network and a local subnet |
| **Resource** | An IP prefix (e.g., `10.0.0.0/24`) or a domain (e.g., `*.internal`) to expose |
| **Policy group** | The access-control group that controls which peers can use this network |

### Use Cases

- **Remote access to office LAN**: employees on the NetBird mesh reach internal servers without a traditional VPN concentrator.
- **Site-to-site**: two offices, each with a routing peer, can exchange traffic across the internet.
- **Cloud VPC access**: a VM inside a private VPC acts as a routing peer; all mesh peers access the VPC subnet through it.
- **Exposing non-NetBird devices**: printers, IoT, and legacy servers that cannot run the client become reachable via a nearby routing peer.

### Creating a Network

Dashboard → **Networks → Add Network**:

1. **Name** – descriptive label (e.g., `office-london`).
2. **Resources** – add one or more resources:
   - **Single IP**: `192.168.10.1`
   - **IP range / prefix**: `192.168.10.0/24`
   - **Domain name**: `fileserver.corp.local`
   - **Wildcard domain**: `*.corp.example.com`
3. **Routing Peers** – select the peer(s) that will forward traffic to this network. Multiple routing peers provide high availability and load balancing.
4. **Access Groups** – groups whose peers are allowed to access this network (resolved via Access Policies).
5. **Masquerading** – enabled by default; the routing peer performs NAT so remote devices see requests from the routing peer's LAN IP. Disable for transparent routing when the remote subnet has a return route to the NetBird overlay.

### Domain Resources

When a resource is defined as a **domain name** or **wildcard domain** (e.g., `*.corp.example.com`):

- NetBird distributes the routing rule to peers alongside a DNS resolution rule.
- The **routing peer** acts as a DNS forwarder for that domain (on port `22054`; older clients below v0.59.0 use `5353`).
- Clients resolve the domain through the routing peer, which looks up the address on the local network and routes traffic accordingly.
- **Wildcard domain routing** (`*.example.com`) must be explicitly enabled in **Settings → Networks → Enable wildcard domain routing** before it can be used.

**Troubleshooting domain resources:**
```bash
# Check domain resource resolution via the routing peer's forwarder
nslookup -port=22054 fileserver.corp.local <routing-peer-ip>

# List domain-based resources and their status
netbird networks ls
```

### High Availability

Add multiple routing peers to the same network. NetBird automatically load-balances traffic across them and fails over if one becomes unavailable.

### Legacy Network Routes

Older NetBird deployments use **Network Routes** (Dashboard → **Network Routes**). The functionality is the same; **Networks** is the recommended approach for new configurations. Both features coexist.

---

## DNS Management

NetBird deploys a local DNS resolver on every peer. Centralized DNS settings are pushed to all peers so internal hostnames resolve automatically.

Dashboard → **DNS**

### Nameservers

Add custom nameservers that peers should query for specific domains:

| Field | Description |
|-------|-------------|
| **Name** | Label for this nameserver entry |
| **Nameserver IP** | IP address of the DNS server (e.g., `10.0.0.53`) |
| **Port** | DNS port (default: `53`) |
| **Match Domains** | Domain suffixes resolved by this server (e.g., `corp.example.com`, `internal`) |
| **Primary** | When enabled, this server handles all queries not matched by other rules |
| **Search Domains** | Appended to unqualified hostnames (e.g., `corp` → `corp.example.com`) |
| **Distribution Groups** | Groups of peers that receive this nameserver config |

#### Example: Internal AD/BIND Nameserver

```
Name:            corp-dns
Nameserver IP:   10.0.0.10
Port:            53
Match Domains:   corp.example.com, internal
Distribution:    All
```

Peers receiving this config will forward all queries for `*.corp.example.com` and `*.internal` to `10.0.0.10`; all other queries go to the system default resolver.

### Automatic Peer DNS Names

NetBird assigns every peer a DNS name in the form `<peer-name>.netbird.cloud` (cloud) or `<peer-name>.<your-domain>` (self-hosted). These names resolve to the peer's NetBird overlay IP without any manual configuration.

### DNS Resolution Flow

1. Peer queries a hostname.
2. NetBird's local resolver checks match-domain rules.
3. If matched → forwards to the configured internal nameserver.
4. If unmatched → forwards to the system/primary resolver.

---

## Site-to-Site Example

**Scenario**: connect `office-a` (`192.168.1.0/24`) and `office-b` (`192.168.2.0/24`).

**Step 1 – Install NetBird on a routing peer in each office**:
```bash
# office-a router (Linux)
netbird up --setup-key <KEY_A>

# office-b router (Linux)
netbird up --setup-key <KEY_B>
```

**Step 2 – Enable IP forwarding on each routing peer**:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
```

**Step 3 – Create Networks in the dashboard**:

| Network | Resource | Routing Peer | Masquerade |
|---------|----------|--------------|------------|
| `office-a` | `192.168.1.0/24` | peer in office-a | enabled |
| `office-b` | `192.168.2.0/24` | peer in office-b | enabled |

**Step 4 – Create an Access Policy** allowing both office groups to use each other's networks.

Devices in office-a can now reach `192.168.2.x` and vice versa through the routing peers.

---

## Tips

- Enable **Masquerading** unless the remote subnet has explicit routes back to the NetBird overlay (`100.64.0.0/10` or your configured range).
- Use **domain-based resources** (`*.internal`) to route DNS-resolved traffic without knowing specific IPs.
- Check routing peer connectivity with `netbird status --detail` — look for the route entry and relay/direct status.
- Multiple routing peers per network improve resilience; NetBird picks the lowest-latency available peer.

## References

- [Networks](https://docs.netbird.io/manage/networks)
- [Network Routes](https://docs.netbird.io/manage/network-routes)
- [Network Routes Use Cases](https://docs.netbird.io/manage/network-routes/use-cases)
- [DNS in NetBird](https://docs.netbird.io/manage/dns)
- [DNS Settings](https://docs.netbird.io/manage/dns/dns-settings)
