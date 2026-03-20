---
name: docker-compose-integration
description: Integrating NetBird with Docker Compose using two approaches: shared network namespace (sidecar/pod pattern) and subnet routing. Use when exposing Docker Compose services via NetBird mesh, giving containers a NetBird mesh IP, or routing traffic to Docker subnets from remote peers.
---
# NetBird Docker Compose Integration

## Overview

Two main approaches exist for integrating NetBird with Docker Compose:

| Approach | Best For |
|----------|----------|
| **A) Shared network namespace** | Exposing specific containers directly on the NetBird mesh (each group gets a mesh IP) |
| **B) Subnet routing** | Exposing an entire Docker subnet to all mesh peers via a routing peer on the host |

---

## Approach A: Shared Network Namespace (Sidecar/Pod Pattern)

Run a NetBird container and point other service(s) at it using `network_mode: "service:<name>"`. All attached services share the same network stack — same mesh IP, differentiated by port.

### Key properties
- **Not 1:1**: one NetBird container can serve multiple Docker services simultaneously.
- All services sharing the namespace get the **same single NetBird mesh IP** (e.g. `100.64.x.x`).
- Services are distinguished by **port number** only.
- Services communicate with each other via `localhost` inside the shared namespace.

### Example: one NetBird container, multiple services

```yaml
services:
  netbird:
    image: netbirdio/netbird:latest
    cap_add: [NET_ADMIN, SYS_ADMIN]
    cap_drop: [NET_RAW]
    environment:
      NB_SETUP_KEY: <setup-key>
      NB_HOSTNAME: my-docker-host
    volumes:
      - netbird-config:/etc/netbird
    restart: unless-stopped

  app1:
    image: myapp1:latest
    network_mode: "service:netbird"
    depends_on: [netbird]

  app2:
    image: myapp2:latest
    network_mode: "service:netbird"
    depends_on: [netbird]

  app3:
    image: myapp3:latest
    network_mode: "service:netbird"
    depends_on: [netbird]

volumes:
  netbird-config:
```

After startup, `app1`, `app2`, and `app3` are all reachable via the same mesh IP on their respective ports.

### Limitations
- **Port conflicts**: services sharing the namespace must use different ports.
- **No independent scaling**: `docker compose up --scale app1=3` does not work for services with `network_mode: service:...`. Scale by duplicating the entire netbird+service group instead.
- `depends_on: [netbird]` is required so attached services wait for the NetBird container to be ready before starting.
- Services using `network_mode: service:...` cannot define their own `networks:`, `ports:`, or `links:` keys.

---

## Approach B: Subnet Routing via NetBird Networks

Install the NetBird agent on the **Docker host** (not inside a container), then advertise the Docker subnet (e.g. `172.18.0.0/16`) as a network resource in the NetBird dashboard.

All peers on the mesh can then reach every container in that subnet via the routing peer, without any agent running inside the containers.

### Steps
1. Install NetBird on the Docker host and connect it to your management server.
2. In the NetBird dashboard, go to **Networks** → **Add Network**.
3. Add a resource with the Docker bridge subnet (find it with `docker network inspect <network>`).
4. Assign the Docker host peer as the routing peer.
5. Create an access policy to allow the desired peer groups to reach the resource.

### Trade-offs vs Approach A

| | Approach A | Approach B |
|---|---|---|
| Agent location | Inside Docker (container) | Host OS |
| Granularity | Per service/group | Entire subnet |
| Container changes needed | Yes (`network_mode`) | None |
| Scales to many services | Yes (share one container) | Yes (automatic) |
| Individual service mesh IPs | No (shared IP) | No (original container IPs) |

---

## What is `--scale`?

`docker compose up --scale <service>=N` runs N identical replicas of a service (useful for load balancing or redundancy). It **cannot** be used with services that have `network_mode: service:<name>` because all replicas would conflict over the same shared network namespace.

---

## Known Quirks and Limitations

- ⚠️ Services with `network_mode: service:netbird` **lose access to Compose-defined networks** — they can no longer reach other services by service name unless those services also share the same namespace.
- ⚠️ If the `netbird` container restarts, all services sharing its namespace lose connectivity until they are also restarted.
- The sidecar pattern is conceptually identical to a Kubernetes pod — multiple containers sharing one network namespace.

---

## References

- [SignalNine: Expose a service using NetBird mesh IP](https://signalnine.dev/posts/expose-service-using-netbird-mesh-ip)
- [NetBird Knowledge Hub: Zero-trust access without agents](https://netbird.io/knowledge-hub/zero-trust-access-to-internal-resources-without-installing-agents)
- [NetBird Docs: Networks (Subnet Routing)](https://docs.netbird.io/manage/networks)
- [Docker Docs: Compose networking](https://docs.docker.com/compose/how-tos/networking/)
