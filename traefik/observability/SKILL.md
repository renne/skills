---
name: observability
description: Traefik observability configuration covering Prometheus metrics, OpenTelemetry and Jaeger distributed tracing, access logs, application logs, and the API/dashboard. Use when setting up monitoring and alerting for Traefik, configuring metrics scraping with Prometheus, enabling distributed tracing, troubleshooting via access logs, or exposing the Traefik management API.
---
# Traefik Observability

Source: https://doc.traefik.io/traefik/observe/overview/

## Overview

Traefik provides built-in observability through four main mechanisms:

1. **Logs** — Application-level events (startup, errors, provider updates)
2. **Access Logs** — Per-request logs (IP, method, URL, status, duration)
3. **Metrics** — Numeric time-series data (request count, latency, errors)
4. **Tracing** — Distributed request tracing across services

All observability options are configured in **static configuration** (`traefik.yaml`).

---

## Application Logs

Control Traefik's internal log level and format.

```yaml
log:
  level: INFO        # TRACE, DEBUG, INFO, WARN, ERROR, FATAL, PANIC
  format: json       # common (default) or json
  filePath: /var/log/traefik/traefik.log  # optional: default is stdout
  noColor: false     # Disable ANSI color in console format
```

Log levels:
| Level | Description |
|-------|-------------|
| `TRACE` | Very verbose; includes internal operations |
| `DEBUG` | Detailed debug information |
| `INFO` | Normal operational messages (default) |
| `WARN` | Non-critical warnings |
| `ERROR` | Errors that don't stop Traefik |
| `FATAL` | Critical errors; Traefik stops |

---

## Access Logs

Log every HTTP request/response. Disabled by default; enable with an empty `accessLog:` block.

```yaml
accessLog:
  filePath: /var/log/traefik/access.log   # default: stdout
  format: json                             # common (CLF) or json
  bufferingSize: 100                       # Buffer N lines before writing to disk
  filters:
    statusCodes:
      - "200"
      - "400-499"
      - "500-599"
    retryAttempts: true       # Log entries for retry attempts
    minDuration: 10ms         # Only log requests taking longer than this
  fields:
    defaultMode: keep          # keep or drop all fields by default
    names:
      ClientHost: keep
      RequestMethod: keep
      RequestPath: keep
      DownstreamStatus: keep
      Duration: keep
      RouterName: keep
      ServiceName: keep
      # Redact sensitive fields:
      Authorization: redact
    headers:
      defaultMode: drop        # drop all request headers by default
      names:
        User-Agent: keep
        Authorization: redact
```

### Common Access Log Fields

| Field | Description |
|-------|-------------|
| `ClientHost` | Client IP address |
| `RequestMethod` | HTTP method (GET, POST, …) |
| `RequestPath` | URL path |
| `RequestProtocol` | HTTP/1.1, HTTP/2, etc. |
| `DownstreamStatus` | HTTP status code returned |
| `Duration` | Total request duration (ms) |
| `RouterName` | Matched router name |
| `ServiceName` | Backend service name |
| `ServiceURL` | Backend server URL |
| `TLSVersion` | TLS version (for HTTPS) |

---

## Metrics

### Prometheus

Traefik natively exposes Prometheus metrics at `/metrics`.

```yaml
metrics:
  prometheus:
    entryPoint: metrics          # Dedicated entrypoint for metrics
    addEntryPointsLabels: true   # Add entrypoint labels to metrics
    addRoutersLabels: true       # Add router labels (more cardinality)
    addServicesLabels: true      # Add service labels to metrics
    buckets:
      - 0.05
      - 0.1
      - 0.3
      - 1.2
      - 5.0
    headerLabels:                # Create labels from request headers
      username: X-Auth-User
```

Define the metrics entrypoint:
```yaml
entryPoints:
  metrics:
    address: ":8082"
```

Prometheus scrape configuration:
```yaml
# prometheus.yml
scrape_configs:
  - job_name: traefik
    static_configs:
      - targets: ["traefik:8082"]
    metrics_path: /metrics
```

#### Key Prometheus Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `traefik_entrypoint_requests_total` | Counter | Total requests per entrypoint |
| `traefik_entrypoint_request_duration_seconds` | Histogram | Request duration per entrypoint |
| `traefik_router_requests_total` | Counter | Total requests per router |
| `traefik_router_request_duration_seconds` | Histogram | Request duration per router |
| `traefik_service_requests_total` | Counter | Total requests per service |
| `traefik_service_request_duration_seconds` | Histogram | Request duration per service |
| `traefik_service_server_up` | Gauge | Backend server health (1=up, 0=down) |
| `traefik_config_reloads_total` | Counter | Number of config reloads |
| `traefik_config_last_reload_success` | Gauge | Timestamp of last successful reload |
| `traefik_tls_certs_not_after` | Gauge | Certificate expiry timestamp |

### OpenTelemetry Metrics (OTLP)

```yaml
metrics:
  openTelemetry:
    address: "otel-collector:4318"
    grpc: {}     # Use gRPC; omit for HTTP
    insecure: true
    headers:
      Authorization: "Bearer my-token"
    addEntryPointsLabels: true
    addServicesLabels: true
    addRoutersLabels: true
    pushInterval: 10s
```

### Other Metrics Backends

```yaml
metrics:
  # InfluxDB v2
  influxDB2:
    address: "http://influxdb:8086"
    token: "my-influxdb-token"
    org: "my-org"
    bucket: "traefik"
    addEntryPointsLabels: true
    addServicesLabels: true

  # StatsD
  statsD:
    address: "statsd:8125"
    pushInterval: 10s
    prefix: traefik
    addEntryPointsLabels: true
    addServicesLabels: true
```

---

## Tracing

### OpenTelemetry (OTLP) — Recommended

```yaml
tracing:
  openTelemetry:
    address: "otel-collector:4318"   # OTLP HTTP endpoint
    # OR for gRPC:
    # address: "otel-collector:4317"
    # grpc: {}
    insecure: true
    headers:
      Authorization: "Bearer my-token"
  serviceName: "traefik"
  sampleRate: 1.0                    # 1.0 = 100% of requests, 0.1 = 10%
  capturedRequestHeaders:
    - "X-Request-ID"
  capturedResponseHeaders:
    - "X-Request-ID"
  safeQueryParams:
    - "lang"
    - "version"
```

### Jaeger

```yaml
tracing:
  jaeger:
    samplingServerURL: "http://jaeger-agent:5778/sampling"
    samplingType: const
    samplingParam: 1.0
    localAgentHostPort: "jaeger-agent:6831"
    gen128Bit: false
    propagation: jaeger
    traceContextHeaderName: uber-trace-id
    collector:
      endpoint: "http://jaeger-collector:14268/api/traces"
```

### Zipkin

```yaml
tracing:
  zipkin:
    httpEndpoint: "http://zipkin:9411/api/v2/spans"
    sameSpan: false
    id128Bit: true
    sampleRate: 1.0
```

---

## API & Dashboard

The Traefik API exposes runtime configuration and health information. The dashboard is a web UI built on the API.

```yaml
api:
  dashboard: true    # Enable dashboard (default: false)
  insecure: false    # Expose API on dedicated port without auth (TESTING ONLY)
  debug: false       # Enable debug endpoints
```

When `insecure: true`, the API is available on port 8080. **Never use this in production.**

### Secure Dashboard Setup

Expose the dashboard via a proper router with authentication:

```yaml
# dynamic/dashboard.yaml
http:
  routers:
    dashboard:
      rule: "Host(`traefik.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      entryPoints:
        - websecure
      service: api@internal
      middlewares:
        - dashboard-auth
      tls:
        certResolver: letsencrypt

  middlewares:
    dashboard-auth:
      basicAuth:
        users:
          - "admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"
```

### API Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /api/version` | Traefik version |
| `GET /api/rawdata` | Complete runtime configuration |
| `GET /api/http/routers` | List HTTP routers |
| `GET /api/http/services` | List HTTP services |
| `GET /api/http/middlewares` | List HTTP middlewares |
| `GET /api/tcp/routers` | List TCP routers |
| `GET /api/tcp/services` | List TCP services |
| `GET /api/udp/routers` | List UDP routers |
| `GET /api/entrypoints` | List entrypoints |
| `GET /api/overview` | Summary statistics |
| `GET /ping` | Health check (returns 200 OK) |

### Health Check (Ping)

```yaml
ping:
  entryPoint: traefik   # Dedicated entrypoint for ping
  terminatingStatusCode: 204   # Status code when Traefik is shutting down
  manualRouting: false         # Manage the route yourself
```

Define a dedicated entrypoint:
```yaml
entryPoints:
  traefik:
    address: ":8080"
```

Use for load balancer health checks: `GET http://traefik:8080/ping`

---

## Kubernetes: Prometheus with ServiceMonitor

If using the Prometheus Operator:

```yaml
# values.yaml for Traefik Helm chart
metrics:
  prometheus:
    entryPoint: metrics
    serviceMonitor:
      enabled: true
      namespace: monitoring
      interval: 30s
      labels:
        release: kube-prometheus-stack
```

---

## References

- [Observability Overview](https://doc.traefik.io/traefik/observe/overview/)
- [Logs](https://doc.traefik.io/traefik/observe/logs/)
- [Access Logs](https://doc.traefik.io/traefik/observe/access-logs/)
- [Prometheus Metrics](https://doc.traefik.io/traefik/v3.4/observability/metrics/prometheus/)
- [OpenTelemetry Metrics](https://doc.traefik.io/traefik/v3.4/observability/metrics/opentelemetry/)
- [OpenTelemetry Tracing](https://doc.traefik.io/traefik/v3.4/observability/tracing/opentelemetry/)
- [Jaeger Tracing](https://doc.traefik.io/traefik/v3.4/observability/tracing/jaeger/)
- [API & Dashboard](https://doc.traefik.io/traefik/operations/api/)
- [Ping (Health Check)](https://doc.traefik.io/traefik/operations/ping/)
