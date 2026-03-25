---
name: use-cases
description: NetBird connectivity use-case guide — when to use Networks vs Network Routes, VPN-to-Site, Site-to-VPN, Site-to-Site, and Exit Nodes scenarios. Use when deciding which NetBird feature to configure for a specific connectivity requirement (remote access, clientless devices, full-tunnel internet routing).
---
# NetBird Use Cases

Sources:
- https://docs.netbird.io/manage/networks/use-cases
- https://docs.netbird.io/manage/network-routes/use-cases

## Feature Selection Matrix

| Scenario | Feature to use | Notes |
|----------|---------------|-------|
| **VPN-to-Site** — NetBird peer accesses remote resources | **Networks** (recommended) | Simpler per-resource access control |
| **VPN-to-Site** — alternative | **Network Routes** | More HA options; masquerade control |
| **Site-to-VPN** — clientless device initiates to NetBird peer | **Network Routes** only | Networks does not support clientless initiators |
| **Site-to-Site** — two networks, neither runs NetBird | **Network Routes** only | Both sides need routing peers |
| **Exit Node** — route all internet traffic through a peer | **Network Routes** only | Requires `0.0.0.0/0` route |

**When to use Networks:**
- Accessing individual services or resources on a remote LAN
- Per-resource access policies (each resource can have its own policy group)
- Simpler setup without masquerade/metric configuration

**When to use Network Routes (instead of or in addition to Networks):**
- Site-to-VPN or Site-to-Site (clientless device initiates the connection)
- Exit node (route all traffic, not just specific destinations)
- Need source IP preservation (masquerade disabled)
- Need ACL Groups for route-level access control
- Need route-level high availability with metric priority

---

## VPN-to-Site

**Goal**: NetBird peers access devices/services on a remote network that do not run NetBird.

**Steps:**
1. Install NetBird on a routing peer inside the remote network.
2. Enable IP forwarding on the routing peer:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
   ```
3. In the dashboard, create a **Network** (or Network Route):
   - Resource: the remote subnet (e.g., `192.168.10.0/24`)
   - Routing peer: the peer inside the remote network
   - Policy group: groups of peers allowed to use this route
4. Create an **Access Policy** allowing the relevant group to access the resource group.

**Notes:**
- Resources in a Network are **not** automatically added to the `All` group — access requires an explicit policy.
- With masquerade enabled (default), the routing peer performs NAT; the remote devices see its IP, not the original peer's.

---

## Site-to-VPN

**Goal**: Clientless devices on Network A initiate connections to NetBird peers (e.g., reach a service running on a NetBird peer).

**Requirement**: Network Routes only (Networks does not support this direction).

**Steps:**
1. Deploy a routing peer on Network A (the side with the clientless devices).
2. Configure Network A's devices/firewall to send traffic for the NetBird overlay (`100.64.0.0/10`) to the routing peer.
3. Create a **Network Route** on the routing peer for the NetBird overlay range.
4. Create an **Access Policy** allowing the routing peer to reach the target NetBird peer.

---

## Site-to-Site

**Goal**: Two networks (neither running NetBird clients on end devices) connect to each other.

**Requirement**: Network Routes only.

**Steps:**
1. Deploy a routing peer in each site.
2. Enable IP forwarding on both routing peers.
3. Create a Network Route on each routing peer for its own subnet:
   - **Site A routing peer**: route for `192.168.1.0/24`
   - **Site B routing peer**: route for `192.168.2.0/24`
4. Add each routing peer to the other site's **Distribution Group** so it receives the route configuration.
5. Create an **Access Policy** allowing the two routing peers to communicate.
6. Configure each site's non-NetBird devices to route traffic for the other subnet through the local routing peer.

**Network Routes key settings for site-to-site:**
- **Masquerade**: enable for simpler setup (routing peer's IP used as source). Disable for source-IP visibility (requires manual return routes on both sides).
- **Distribution Groups**: controls which peers receive the route config (must include the routing peer of the other site).
- **ACL Groups**: optional, for granular per-route access control beyond policy groups.

---

## Exit Nodes

**Goal**: Route all internet traffic (or specific traffic) through a designated peer.

**Requirement**: Network Routes only (route `0.0.0.0/0`).

**Steps:**
1. Choose an exit node peer (e.g., a cloud VM or home router).
2. Enable IP forwarding and configure NAT on the exit node:
   ```bash
   # IP forwarding
   sudo sysctl -w net.ipv4.ip_forward=1

   # NAT (iptables example)
   sudo iptables -t nat -A POSTROUTING -o <WAN_INTERFACE> -j MASQUERADE
   ```
3. Create a **Network Route**:
   - Network: `0.0.0.0/0`
   - Routing peer: the exit node peer
   - Distribution group: peers that should use this exit node
4. Ensure an **Access Policy** allows these peers to communicate with the exit node.

**Notes:**
- Only one `0.0.0.0/0` route should be active per client at a time; multiple routes with different metrics allow failover.
- The exit node itself accesses the internet; DNS leaks may occur — consider configuring DNS via NetBird as well.

---

## High Availability for Routes

Both Networks and Network Routes support multiple routing peers for the same resource/route:

- **Networks**: add multiple routing peers to the same network.
- **Network Routes**: create multiple routes with the same **network identifier** (`network_id`) but different routing peers.
  - Lower **metric** = higher priority (preferred peer).
  - If the preferred peer is unavailable, NetBird automatically fails over to the next lowest-metric peer.
  - NetBird also considers connection type (direct vs. relayed) when selecting among equal-metric peers.

---

## Masquerade vs. Transparent Routing

| Mode | Default | Behavior | When to use |
|------|---------|----------|------------|
| **Masquerade on** | ✓ Yes | Routing peer NATs — remote sees routing peer's LAN IP | Simple setup; no return routes needed on remote devices |
| **Masquerade off** | No | Source IP preserved end-to-end | When compliance/audit requires original source IPs; requires manual return routes |

---

## References

- [Networks](https://docs.netbird.io/manage/networks)
- [Networks Use Cases](https://docs.netbird.io/manage/networks/use-cases)
- [Network Routes](https://docs.netbird.io/manage/network-routes)
- [Network Routes Use Cases](https://docs.netbird.io/manage/network-routes/use-cases)
