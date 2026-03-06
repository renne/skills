---
name: plugins
description: CoreDNS plugin reference covering all major built-in plugins including forward, cache, rewrite, file, hosts, kubernetes, acl, dnssec, log, errors, health, ready, metrics (prometheus), bind, reload, template, auto, secondary, loadbalance, loop, tls, whoami, chaos, nsid, any, dns64, route53, etcd, and pprof. Use when configuring specific CoreDNS plugins, looking up plugin syntax and options, or combining multiple plugins in a Corefile.
---
# CoreDNS Plugins Reference

Source: https://coredns.io/plugins/

## Plugin Overview

CoreDNS functionality is entirely implemented through **plugins**. Plugins are chained in a fixed order (defined in `plugin.cfg`); each plugin can choose to handle a query, pass it to the next plugin, or return an error. Most plugins are configured inside a server block in the Corefile.

---

## forward

Forwards queries to one or more upstream resolvers. This is the most common plugin for recursive resolution.

```
.:53 {
    forward . 8.8.8.8 8.8.4.4 {
        max_fails 3
        health_check 5s
        policy round_robin   # or sequential, random
        prefer_udp
    }
}
```

### Domain-specific forwarding

```
.:53 {
    # Internal domain to corporate DNS
    forward internal.corp. 10.0.0.1

    # Everything else to public DNS
    forward . 1.1.1.1 9.9.9.9
}
```

### Options

| Option | Description |
|--------|-------------|
| `max_fails N` | Upstream marked unhealthy after N consecutive failures (default: 2) |
| `expire DURATION` | Expire health-checked upstreams after this duration |
| `health_check DURATION` | Interval for health check probes (default: 0.5s) |
| `policy POLICY` | Upstream selection: `random` (default), `round_robin`, `sequential` |
| `prefer_udp` | Prefer UDP even if request arrived over TCP |
| `force_tcp` | Always use TCP to upstreams |
| `tls CERT KEY CA` | Use TLS when connecting to upstreams (DoT) |
| `tls_servername NAME` | SNI name for TLS upstream connections |

---

## cache

Caches DNS responses in memory to reduce upstream load and improve latency.

```
.:53 {
    forward . 8.8.8.8
    cache 300          # TTL cap in seconds (default: 3600)
}
```

### Extended configuration

```
.:53 {
    forward . 8.8.8.8
    cache {
        success 9984 300   # max entries, TTL cap for positive responses
        denial  9984 5     # max entries, TTL cap for NXDOMAIN/NODATA
        prefetch 10        # prefetch popular items when 10% of TTL remains
        serve_stale 5m     # serve stale entries for up to 5m if upstream unreachable
    }
}
```

---

## rewrite

Rewrites DNS queries and/or responses before further processing or after receiving an upstream answer.

```
.:53 {
    # Rewrite a name
    rewrite name old.example.com new.example.com

    # Rewrite name with regex
    rewrite name regex (.*).legacy.example.com {1}.new.example.com

    # Rewrite record type
    rewrite type ANY HINFO

    # Rewrite with response rewriting enabled (answer section rewritten back)
    rewrite name exact old.example.com new.example.com answer auto

    forward . 8.8.8.8
}
```

### Rewrite targets

| Target | Example |
|--------|---------|
| `name exact` | `rewrite name exact src.example.com dst.example.com` |
| `name regex` | `rewrite name regex (.*).src {1}.dst` |
| `type` | `rewrite type ANY HINFO` |
| `class` | `rewrite class CH IN` |
| `edns0` | `rewrite edns0 local set 0xffee 0x000a` |

---

## file

Serves DNS records from an RFC 1035 zone file. Acts as an authoritative server for the zone.

```
example.org {
    file /zones/example.org.db {
        reload 1m      # check SOA serial every minute; reload if changed
        fallthrough    # pass to next plugin if record not found
    }
}
```

### Zone file format (RFC 1035)

```
$ORIGIN example.org.
$TTL 3600
@   IN  SOA ns1.example.org. admin.example.org. (
            2024010101 ; serial
            3600       ; refresh
            900        ; retry
            604800     ; expire
            300 )      ; minimum TTL

    IN  NS  ns1.example.org.
ns1 IN  A   203.0.113.1
www IN  A   203.0.113.2
    IN  AAAA 2001:db8::2
```

---

## auto

Automatically loads and reloads zone files from a directory. Useful when managing many zones.

```
. {
    auto {
        directory /zones
        reload 60s         # rescan directory interval
        transfer to *      # allow outbound zone transfers
    }
}
```

CoreDNS discovers files in the directory and infers zone names from filenames or SOA records.

---

## secondary

Acts as a secondary (slave) DNS server. Transfers zones via AXFR from a primary server and keeps them in memory.

```
example.org {
    secondary {
        transfer from 10.0.1.1 10.1.2.1
    }
}
```

If all primaries are unreachable at startup, CoreDNS retries indefinitely every 10 seconds.

---

## hosts

Serves A/AAAA records and reverse PTR records from a hosts-file-format file (like `/etc/hosts`).

```
.:53 {
    hosts /etc/hosts {
        10.0.0.1 myhost.internal myhost
        192.168.1.50 dev.example.com
        fallthrough          # continue to next plugin if no match
        reload 1m            # reload file every minute
        no_reverse           # do not generate PTR records
    }
}
```

When no file is specified, CoreDNS reads `/etc/hosts`.

---

## kubernetes

Provides in-cluster DNS resolution for Kubernetes services and pods. This is the core of Kubernetes cluster DNS.

```
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure          # pods verified = require pod IP match (more secure)
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
        namespaces default     # only expose these namespaces (omit = all)
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### kubernetes options

| Option | Description |
|--------|-------------|
| `pods disabled` | Do not create DNS records for pods (default) |
| `pods insecure` | Always create pod records |
| `pods verified` | Create pod records only when pod IP matches request source IP |
| `ttl N` | TTL for synthesized records (default: 5) |
| `fallthrough [ZONES]` | Continue to next plugin for these zones if no record found |
| `namespaces NS...` | Limit to specific namespaces |
| `endpoint URL` | Kubernetes API endpoint (default: auto-detected in-cluster) |
| `kubeconfig FILE [CTX]` | Use kubeconfig file (for out-of-cluster use) |

---

## dnssec

Signs responses on-the-fly with DNSSEC. CoreDNS uses NSEC (not NSEC3). Signing keys must be generated externally (e.g., with `dnssec-keygen`).

```
example.org {
    file /zones/example.org.db
    dnssec {
        key file /keys/Kexample.org.+008+12345
    }
}
```

For zones served by the `file` plugin, you can also pre-sign the zone with `dnssec-signzone` / `ldns-signzone` and serve the signed file directly.

---

## template

Generates DNS responses dynamically based on the query name and type using Go templates.

```
.:53 {
    # Synthesize A records for *.dynamic.example.com
    template IN A dynamic.example.com {
        match   ^(.*)\.dynamic\.example\.com\.$
        answer  "{{ .Name }} 60 IN A 192.0.2.100"
        fallthrough
    }
}
```

### Template variables

| Variable | Value |
|----------|-------|
| `{{ .Name }}` | FQDN of the query |
| `{{ .Type }}` | Query type (e.g., `A`) |
| `{{ .Class }}` | Query class (e.g., `IN`) |
| `{{ index .Match 1 }}` | First capture group from the regex match |

---

## loadbalance

Randomly shuffles A, AAAA, and MX records in responses to distribute load across multiple IPs. Optionally prefers records matching specified subnets (useful for returning geographically closer addresses first).

```
.:53 {
    loadbalance round_robin   # only option: round_robin
    forward . 8.8.8.8
}
```

---

## acl (v1.14.1+)

Enforces access control policies on source IP. Rules are evaluated top-to-bottom; first match wins. Supports `allow`, `block` (REFUSED), `drop` (silent), and `filter` (NOERROR empty answer) actions. Also supports filtering by record type and FQDN pattern.

```
.:53 {
    acl {
        allow net 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
        block                # block everything else
    }
}
```

---

## azure

Serves zone data from Microsoft Azure DNS. Requires Azure credentials with DNS Zone Reader permission.

```
.:53 {
    azure {
        tenant-id     <AZURE_TENANT_ID>
        client-id     <AZURE_CLIENT_ID>
        client-secret <AZURE_CLIENT_SECRET>
        subscription  <SUBSCRIPTION_ID>
        resource-group <RESOURCE_GROUP>
        zone-name     example.com
    }
}
```

---

## bufsize

Limits the EDNS0 buffer size advertised by CoreDNS to prevent IP fragmentation. Useful on networks where large UDP packets are dropped.

```
.:53 {
    bufsize 512   # advertise max 512-byte EDNS0 buffer
    forward . 8.8.8.8
}
```

---

## cancel

Cancels a request's context after a configurable timeout (default 5001 milliseconds). Downstream plugins that respect context cancellation will abort processing, preventing resource leaks from slow upstream queries.

```
.:53 {
    cancel 3000ms   # cancel after 3 seconds
    forward . 8.8.8.8
}
```

---

## clouddns

Serves zone data from Google Cloud Platform (GCP) Cloud DNS. Authenticates via Application Default Credentials or a service account key file.

```
.:53 {
    clouddns {
        project my-gcp-project
        zone-name example-com
    }
}
```

---

## grpc

Proxies DNS messages to an upstream server via gRPC (instead of plain DNS-over-UDP/TCP). Supports TLS for the gRPC connection.

```
.:53 {
    grpc . 10.0.0.1:5300 {
        tls /certs/client.crt /certs/client.key /certs/ca.crt
    }
}
```

---

## header

Modifies DNS message headers for queries or responses. Can set or clear flags (e.g., `AA`, `TC`, `RD`, `RA`).

```
.:53 {
    header {
        response set ra   # set RA (recursion available) flag on responses
    }
    forward . 8.8.8.8
}
```

---

## https

Configures DNS-over-HTTPS (DoH) server options for a listener already declared with the `https://` scheme in the server block address.

```
https://.:443 {
    tls /certs/server.crt /certs/server.key
    https
    forward . 8.8.8.8
}
```

---

## https3

Configures DNS-over-HTTPS/3 (DoH3) using HTTP/3 (QUIC) transport. Requires the listener to be declared with the `h3://` scheme.

```
h3://.:443 {
    tls /certs/server.crt /certs/server.key
    https3
    forward . 8.8.8.8
}
```

---

## k8s_external

Resolves load balancer external IPs and node IPs for Kubernetes services from **outside** the cluster, using a user-defined zone. Complements the `kubernetes` plugin which handles in-cluster resolution.

```
external.example.org {
    k8s_external {
        apex external.example.org
    }
}
```

Queries for `<svc>.<ns>.external.example.org` return the LoadBalancer external IP of that service.

---

## minimal

Minimizes the size of DNS response messages by removing unnecessary records from the Authority and Additional sections, while preserving the Answer section. Useful to stay within UDP packet limits.

```
.:53 {
    minimal
    forward . 8.8.8.8
}
```

---

## multisocket

Allows starting multiple server instances that listen on the **same port** using `SO_REUSEPORT`. Improves multi-core throughput under high query load.

```
.:53 {
    multisocket
    forward . 8.8.8.8
}
```

---

## nomad

Reads zone data from a HashiCorp Nomad cluster's service catalog and exposes services as DNS records.

```
.:53 {
    nomad {
        address http://127.0.0.1:4646
        token   <NOMAD_TOKEN>
        ttl     30
    }
}
```

---

## quic

Configures DNS-over-QUIC (DoQ) server options for a listener declared with the `quic://` scheme. Requires a TLS certificate.

```
quic://.:853 {
    tls /certs/server.crt /certs/server.key
    quic
    forward . 8.8.8.8
}
```

---

## loop

Detects and aborts forwarding loops. Recommended in any configuration that uses `forward`.

```
.:53 {
    loop
    forward . 8.8.8.8
}
```

---

## log

Logs each DNS query to stdout.

```
.:53 {
    log
    # or log with a custom format:
    log . {
        class all          # log all queries (default: all)
        format combined    # format: combined or common
    }
}
```

---

## errors

Logs errors to stdout and collects them into Prometheus metrics.

```
.:53 {
    errors
    forward . 8.8.8.8
}
```

---

## health

Exposes an HTTP health check endpoint for liveness probes.

```
.:53 {
    health :8080       # listen on port 8080; default is :8080
}
```

Access: `curl http://localhost:8080/health` → `OK`

---

## ready

Exposes an HTTP readiness endpoint. Returns `200 OK` only when all plugins that implement the `Readiness` interface are ready. Used for Kubernetes readiness probes.

```
.:53 {
    ready :8181
}
```

Access: `curl http://localhost:8181/ready` → `OK`

---

## prometheus (metrics)

Exposes a Prometheus `/metrics` endpoint. Plugin name in Corefile is `prometheus`.

```
.:53 {
    prometheus :9153    # default port 9153
}
```

### Key metrics

| Metric | Description |
|--------|-------------|
| `coredns_dns_requests_total` | Total DNS queries received |
| `coredns_dns_responses_total` | Total DNS responses sent (labelled by rcode) |
| `coredns_dns_request_duration_seconds` | Request duration histogram |
| `coredns_cache_hits_total` | Cache hits |
| `coredns_cache_misses_total` | Cache misses |
| `coredns_panics_total` | Server panics |
| `coredns_build_info` | Build/version info |

---

## bind

Binds CoreDNS to specific network interfaces or IP addresses.

```
.:53 {
    bind 127.0.0.1 ::1    # listen only on loopback
}
```

---

## tls

Configures TLS for DNS-over-TLS (DoT) or DNS-over-HTTPS (DoH) listeners.

```
tls://.:853 {
    tls /certs/server.crt /certs/server.key /certs/ca.crt
    forward . 8.8.8.8
}
```

When used without a CA file, client certificate verification is disabled.

---

## reload

Automatically reloads the Corefile when it changes (detected by hash). No server restart needed.

```
.:53 {
    reload 10s          # check interval (default: 30s)
    forward . 8.8.8.8
}
```

---

## any

Handles `ANY` queries. By default responds with a `HINFO` record (per RFC 8482) to deter amplification attacks.

```
.:53 {
    any
    forward . 8.8.8.8
}
```

---

## chaos

Answers `CH TXT` queries for `version.bind` and `version.server`. Useful for identifying a CoreDNS instance.

```
.:53 {
    chaos CoreDNS-1.x.y coredns@example.org
}
```

---

## whoami

Responds to queries with the server's local address and transport. Useful for testing.

```
whoami.example.org {
    whoami
}
```

Query: `dig +short whoami.example.org` returns `<server-IP> <local-port> <transport>`

---

## nsid

Adds a Name Server Identifier (RFC 5001) to responses. Helps identify which server answered a query in multi-server deployments.

```
.:53 {
    nsid "server-1"
}
```

---

## dns64

Synthesizes AAAA records from A records to support IPv6-only clients accessing IPv4 services (RFC 6147).

```
.:53 {
    dns64 {
        prefix 64:ff9b::/96     # NAT64 prefix (default: 64:ff9b::/96)
    }
    forward . 8.8.8.8
}
```

---

## route53

Serves DNS zones from AWS Route 53 hosted zones.

```
.:53 {
    route53 example.org.:Z1234567890 {
        aws_access_key     AKIAIOSFODNN7EXAMPLE
        aws_secret_key     wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        # or use AWS_PROFILE / instance role
    }
}
```

---

## etcd

Serves DNS records stored in etcd (SkyDNS-compatible format).

```
.:53 {
    etcd {
        path    /skydns
        endpoint http://localhost:2379
    }
    forward . 8.8.8.8
}
```

---

## pprof

Exposes Go runtime profiling endpoints (CPU, memory) at `http://localhost:6053/debug/pprof`.

```
.:53 {
    pprof localhost:6053
}
```

Only enable in non-production or secured environments.

---

## Plugin Combination Patterns

### Typical recursive resolver

```
.:53 {
    errors
    health
    ready
    log
    prometheus :9153
    cache 30 {
        prefetch 10
    }
    loop
    reload
    loadbalance
    forward . 1.1.1.1 8.8.8.8
}
```

### Authoritative + fallback

```
example.org {
    file /zones/example.org.db {
        reload 1m
    }
    dnssec {
        key file /keys/Kexample.org.+008+12345
    }
    log
    errors
}

. {
    forward . 8.8.8.8
    cache
    errors
}
```

### Access-controlled corporate resolver

```
.:53 {
    acl {
        allow net 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
        block
    }
    hosts /etc/custom-hosts {
        fallthrough
    }
    rewrite name api.legacy.corp api.new.corp answer auto
    forward internal.corp. 10.10.10.1
    forward . 1.1.1.1 8.8.8.8
    cache 60
    log
    errors
    prometheus :9153
}
```

## References

- [CoreDNS Plugins](https://coredns.io/plugins/)
- [forward plugin](https://coredns.io/plugins/forward/)
- [cache plugin](https://coredns.io/plugins/cache/)
- [rewrite plugin](https://coredns.io/plugins/rewrite/)
- [file plugin](https://coredns.io/plugins/file/)
- [kubernetes plugin](https://coredns.io/plugins/kubernetes/)
- [acl plugin](https://coredns.io/plugins/acl/)
- [template plugin](https://coredns.io/plugins/template/)
- [prometheus plugin](https://coredns.io/plugins/metrics/)
- [CoreDNS GitHub](https://github.com/coredns/coredns)
