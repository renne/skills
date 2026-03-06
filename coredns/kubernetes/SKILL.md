---
name: kubernetes
description: CoreDNS in Kubernetes covering default cluster DNS setup, Corefile ConfigMap management, customising forwarding and caching, stub zones, node-local DNS, debugging cluster DNS, and upgrading from kube-dns. Use when configuring or troubleshooting CoreDNS in a Kubernetes cluster, customising cluster DNS resolution, setting up stub zones for split-horizon DNS, or diagnosing DNS failures inside pods.
---
# CoreDNS in Kubernetes

Source: https://coredns.io/manual/toc/ · https://kubernetes.io/docs/tasks/administer-cluster/coredns/

## Overview

CoreDNS is the default cluster DNS server for Kubernetes (since v1.13). It runs as a `Deployment` in the `kube-system` namespace and is configured through a `ConfigMap` called `coredns`. The Kubernetes API server and kubelet advertise the CoreDNS `ClusterIP` as the DNS server for all pods.

## Default Corefile (Kubernetes)

A typical out-of-the-box Corefile for Kubernetes looks like:

```
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

### What each plugin does

| Plugin | Role |
|--------|------|
| `errors` | Log errors to stdout |
| `health` | Liveness probe at `:8080/health` (lameduck: 5 s grace on shutdown) |
| `ready` | Readiness probe at `:8182/ready` |
| `kubernetes` | Answer DNS for `cluster.local` and reverse zones |
| `prometheus` | Metrics at `:9153/metrics` |
| `forward` | Forward non-cluster queries to the node's resolvers |
| `cache 30` | Cache responses for up to 30 s |
| `loop` | Detect and abort forwarding loops |
| `reload` | Auto-reload ConfigMap changes |
| `loadbalance` | Round-robin A/AAAA records |

## Viewing and Editing the CoreDNS ConfigMap

```bash
# View current Corefile
kubectl -n kube-system get configmap coredns -o yaml

# Edit in-place
kubectl -n kube-system edit configmap coredns
```

After saving, CoreDNS detects the change (via `reload` plugin) and applies it within 30–60 seconds without restarting.

To force an immediate restart:

```bash
kubectl -n kube-system rollout restart deployment/coredns
```

## Kubernetes DNS Records

CoreDNS synthesises DNS records following the [Kubernetes DNS specification](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/):

### Services

| Query | Response |
|-------|----------|
| `<service>.<namespace>.svc.cluster.local` | ClusterIP (A/AAAA) |
| `<service>.<namespace>.svc.cluster.local` | SRV records for named ports |
| `<service>.<namespace>.svc.cluster.local` with `type: ExternalName` | CNAME to external name |

Headless services (ClusterIP = None) return all pod IPs.

### Pods

When `pods insecure` is set (default in most distributions):

```
<pod-ip-dashes>.<namespace>.pod.cluster.local
# e.g. 10-244-1-5.default.pod.cluster.local → 10.244.1.5
```

### Reverse DNS

CoreDNS handles `in-addr.arpa` and `ip6.arpa` for reverse lookups of pod/service IPs.

## Customising the Corefile

### Adding a stub zone (split-horizon DNS)

Route queries for a specific domain to a different upstream:

```
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    # Forward internal.corp. to a corporate DNS server
    forward internal.corp. 10.10.10.1 10.10.10.2
    # Everything else goes to node resolvers
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### Using rewrite to redirect a domain inside the cluster

```
.:53 {
    rewrite name api.legacy.corp api-service.default.svc.cluster.local
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
}
```

### Exposing additional namespaces

By default the `kubernetes` plugin exposes all namespaces. To restrict to specific ones:

```
kubernetes cluster.local {
    namespaces default production
    pods insecure
    fallthrough
}
```

### Serving custom static records inside the cluster

```
.:53 {
    hosts {
        192.168.1.100 legacy-app.internal
        fallthrough
    }
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    forward . /etc/resolv.conf
    cache 30
}
```

### Setting a custom TTL for cluster records

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
    ttl 60          # default is 5 seconds
    pods insecure
    fallthrough in-addr.arpa ip6.arpa
}
```

## Applying the ConfigMap Change

Edit the ConfigMap, then apply:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health { lameduck 5s }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        forward internal.corp. 10.10.10.1
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
        prometheus :9153
    }
EOF
```

## Node-Local DNS Cache

For large clusters, deploy a `NodeLocal DNSCache` DaemonSet to reduce latency and kube-dns load. It runs CoreDNS on each node listening on a link-local IP (typically `169.254.20.10`).

```bash
# Check if NodeLocal DNS is running
kubectl -n kube-system get daemonset node-local-dns
```

When NodeLocal DNS is enabled, pods are configured to use the node-local cache, which forwards to the cluster's CoreDNS deployment for cache misses.

## Scaling CoreDNS

```bash
# Scale CoreDNS replicas
kubectl -n kube-system scale deployment coredns --replicas=3

# Or use Horizontal Pod Autoscaler with the cluster-proportional-autoscaler
```

For large clusters, use the `cluster-proportional-autoscaler` sidecar to automatically scale CoreDNS based on cluster size.

## Deploying CoreDNS Manually (without kubeadm)

```bash
# Download the official deployment scripts
git clone https://github.com/coredns/deployment
cd deployment/kubernetes

# Generate manifest (sets correct cluster DNS IP)
./deploy.sh | kubectl apply -f -

# Verify pods are running
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

## Monitoring CoreDNS

### Prometheus metrics

```bash
# Port-forward metrics endpoint
kubectl -n kube-system port-forward svc/kube-dns 9153:9153

# Scrape metrics
curl http://localhost:9153/metrics
```

### Useful PromQL queries

```promql
# DNS queries per second
rate(coredns_dns_requests_total[5m])

# Error rate
sum(rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m]))

# Cache hit ratio
rate(coredns_cache_hits_total[5m]) /
  (rate(coredns_cache_hits_total[5m]) + rate(coredns_cache_misses_total[5m]))

# p99 query latency
histogram_quantile(0.99, rate(coredns_dns_request_duration_seconds_bucket[5m]))
```

### Viewing CoreDNS logs

```bash
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100 -f
```

## Debugging DNS in Pods

### Test DNS resolution from a pod

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- \
  nslookup kubernetes.default
```

### Use dnsutils for richer diagnostics

```bash
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  --rm -it --restart=Never -- bash

# Inside the pod:
dig +short kubernetes.default.svc.cluster.local
nslookup myservice.mynamespace.svc.cluster.local
cat /etc/resolv.conf
```

### Check pod DNS configuration

```bash
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

Expected output on a standard cluster:
```
nameserver 10.96.0.10        # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Temporarily enable query logging

Add the `log` plugin to the Corefile and restart:

```
.:53 {
    log            # ← add this
    errors
    health
    ...
}
```

Remove after troubleshooting to avoid excessive log volume.

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Pods can't resolve `<service>.svc.cluster.local` | CoreDNS not running or misconfigured | Check pod status; inspect logs |
| `NXDOMAIN` for an existing service | Namespace not in `kubernetes` plugin scope | Remove `namespaces` restriction or add the namespace |
| High DNS latency | Many cache misses | Increase `cache` TTL; use NodeLocal DNS |
| `SERVFAIL` for external domains | Upstream unreachable or forwarding loop | Verify `forward` upstreams; ensure `loop` plugin is present |
| CoreDNS `CrashLoopBackOff` | Corefile syntax error | `kubectl -n kube-system logs <pod>`; fix ConfigMap |
| Config change not taking effect | `reload` plugin missing | Add `reload` to Corefile or restart deployment |

## Upgrading from kube-dns to CoreDNS

```bash
cd deployment/kubernetes

# Scale down kube-dns
kubectl -n kube-system scale deployment kube-dns --replicas=0

# Deploy CoreDNS
./deploy.sh | kubectl apply -f -

# Verify
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

## References

- [Kubernetes CoreDNS Administration Guide](https://kubernetes.io/docs/tasks/administer-cluster/coredns/)
- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CoreDNS Kubernetes Plugin](https://coredns.io/plugins/kubernetes/)
- [Customising DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
- [NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
- [CoreDNS Deployment Scripts](https://github.com/coredns/deployment)
- [CoreDNS Manual](https://coredns.io/manual/toc/)
