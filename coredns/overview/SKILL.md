---
name: overview
description: CoreDNS overview covering what CoreDNS is, how to install it, Corefile syntax, server blocks, zones, plugin ordering, snippets, environment variables, and general operation. Use when setting up CoreDNS for the first time, writing or debugging a Corefile, understanding how server blocks and plugin chains work, or looking for general CoreDNS configuration guidance.
---
# CoreDNS Overview & Configuration

Source: https://coredns.io/manual/toc/

## What is CoreDNS?

CoreDNS is a fast, flexible DNS server written in Go. It chains **plugins** to form a DNS processing pipeline, making it highly composable. Each server block in the configuration file (`Corefile`) defines a set of zones and attaches one or more plugins to handle queries for those zones.

CoreDNS is the default in-cluster DNS server for Kubernetes but can also be used as a standalone resolver, authoritative server, forwarder, or any combination thereof.

## Installation

### Pre-compiled binaries

Download a release binary from https://github.com/coredns/coredns/releases, then run it directly:

```bash
./coredns -conf Corefile
```

### Docker

```bash
docker run --rm -p 53:53/udp -p 53:53/tcp \
  -v $(pwd)/Corefile:/etc/coredns/Corefile \
  coredns/coredns -conf /etc/coredns/Corefile
```

### From source (requires Go ≥ 1.21)

```bash
git clone https://github.com/coredns/coredns
cd coredns
make
```

### Startup flags

| Flag | Description |
|------|-------------|
| `-conf <file>` | Path to the Corefile (default: `./Corefile`) |
| `-dns.port <port>` | Default DNS port (default: `53`) |
| `-pidfile <file>` | Write PID to file |
| `-version` | Print version and exit |
| `-quiet` | Suppress version output |

## The Corefile

All configuration lives in a file named `Corefile`. On startup CoreDNS reads the Corefile in the current directory unless `-conf` overrides it.

### Server block syntax

```
[SCHEME://]ZONE[,ZONE...][:PORT] {
    PLUGIN [ARGUMENTS...]
    PLUGIN {
        option value
    }
}
```

- **SCHEME** – optional; defaults to `dns://`. Other values: `tls://`, `https://`, `grpc://`.
- **ZONE** – the DNS zone this block is authoritative/responsible for. Use `.` for the root (match everything). Comma-separate multiple zones.
- **PORT** – optional; defaults to `53`.
- **PLUGIN** – plugin name, optionally followed by arguments or a block.

### Minimal working example

```
.:53 {
    forward . 8.8.8.8 8.8.4.4
    log
    errors
}
```

This forwards all queries to Google DNS and logs queries and errors.

## Plugin Execution Order

The order of plugins **inside** a server block does **not** control execution order. Execution order is fixed at build time in `plugin.cfg`. This means you can list plugins in any order in your Corefile; CoreDNS will always run them in the correct sequence. Consult https://coredns.io/plugins/ for the authoritative sequence.

## Multiple Server Blocks

CoreDNS can serve multiple independent zones simultaneously:

```
example.org {
    file /zones/example.org.db
}

10.0.0.0/24 {
    file /zones/db.10.0.0.0
}

. {
    forward . 1.1.1.1
    cache
}
```

Each block is an independent server. A query is dispatched to the most-specific matching block.

## Multiple Zones in One Block

```
example.org, example.com {
    file /zones/example.org.db
    file /zones/example.com.db
}
```

## Listening on Specific Interfaces

Use the `bind` plugin to restrict which interfaces CoreDNS listens on:

```
.:53 {
    bind lo eth0
    forward . 8.8.8.8
}
```

## Comments

Lines starting with `#` are comments:

```
# This is a comment
.:53 {
    # forward all queries
    forward . 8.8.8.8
}
```

## Environment Variable Substitution

Reference environment variables in the Corefile using `{$VAR}` or `{%VAR%}`:

```
.:53 {
    forward . {$UPSTREAM_DNS}
}
```

Set the variable before starting CoreDNS:

```bash
export UPSTREAM_DNS=9.9.9.9
./coredns
```

## Snippets (Reusable Config Blocks)

Define a named snippet and import it in multiple server blocks to avoid duplication:

```
(common) {
    log
    errors
    prometheus :9153
}

example.org {
    file /zones/example.org.db
    import common
}

. {
    forward . 8.8.8.8
    cache
    import common
}
```

## DNS Transport Schemes

| Scheme | Protocol | Default Port |
|--------|----------|--------------|
| `dns://` | DNS over UDP/TCP (plain) | 53 |
| `tls://` | DNS over TLS (DoT) | 853 |
| `https://` | DNS over HTTPS (DoH) | 443 |
| `grpc://` | DNS over gRPC | 443 |

Example – listen on DoT:

```
tls://.:853 {
    tls /certs/server.crt /certs/server.key
    forward . 1.1.1.1
}
```

## Zone Delegation

CoreDNS respects DNS delegation via NS records in zone files. The `file` plugin can serve a parent zone while queries for delegated child zones fall through to another server block or upstream.

## Health & Readiness (Summary)

```
.:53 {
    health :8080        # /health endpoint for liveness probes
    ready :8181         # /ready endpoint for readiness probes
    forward . 8.8.8.8
}
```

Both return `200 OK` when the server is healthy/ready.

## Logging and Observability (Summary)

```
.:53 {
    log                 # query log to stdout
    errors              # error log to stdout
    prometheus :9153    # Prometheus metrics endpoint
    forward . 8.8.8.8
}
```

## References

- [CoreDNS Manual](https://coredns.io/manual/toc/)
- [CoreDNS GitHub](https://github.com/coredns/coredns)
- [Corefile man page](https://github.com/coredns/coredns/blob/master/corefile.5.md)
- [Plugin ordering (plugin.cfg)](https://github.com/coredns/coredns/blob/master/plugin.cfg)
