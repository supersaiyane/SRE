# Interview Design Question: End-to-End Observability

## Scenario

You're designing a global e-commerce platform. Users access the service via browser and mobile apps. The architecture involves:

- CDN + Frontend (React/Next.js)
- Backend APIs (microservices on Kubernetes)
- Data layer (Redis cache, PostgreSQL, Elasticsearch)
- 99.95% Availability SLO

**The question**: Design an observability strategy that covers the full request path from client to backend, including how you'd detect, diagnose, and resolve issues.

---

## 1. The Three Pillars + Events

| Pillar | What It Provides | Tools |
|--------|-----------------|-------|
| **Metrics** | Quantitative health signals (what's happening now) | Prometheus, Grafana, Thanos |
| **Logs** | Detailed event records (what happened) | Loki, ELK, CloudWatch Logs |
| **Traces** | Request-level paths across services (where is it slow) | Tempo, Jaeger, Zipkin |
| **Events** | Change records (what changed) | Kubernetes events, deploy markers, Grafana annotations |

**Key insight for interviews**: Metrics tell you *something* is wrong. Logs tell you *what* went wrong. Traces tell you *where* it went wrong. Events tell you *why* it started.

---

## 2. Golden Signals by Layer

### Frontend (Client-Side)

| Signal | SLI | Tool |
|--------|-----|------|
| **Latency** | Time to Interactive (TTI) < 3s | RUM (Grafana Faro / Datadog RUM) |
| **Traffic** | Page views per second | RUM + analytics |
| **Errors** | JavaScript errors, failed API calls | RUM error tracking |
| **Saturation** | Browser memory, long tasks (> 50ms) | Performance Observer API |

### API Gateway / Ingress

| Signal | SLI | Tool |
|--------|-----|------|
| **Latency** | Request duration P95 < 200ms | Prometheus (ingress-nginx metrics) |
| **Traffic** | Requests per second by route | Prometheus |
| **Errors** | 4xx and 5xx rate | Prometheus |
| **Saturation** | Active connections, queue depth | Prometheus |

### Backend Microservices

| Signal | SLI | Tool |
|--------|-----|------|
| **Latency** | gRPC/HTTP handler duration | OpenTelemetry + Prometheus |
| **Traffic** | Method call rate | Prometheus counters |
| **Errors** | Error rate by status code | Prometheus |
| **Saturation** | CPU, memory, goroutine count, thread pool | container metrics |

### Data Layer

| Signal | SLI | Tool |
|--------|-----|------|
| **Latency** | Query P95 < 50ms (cache), < 200ms (DB) | Prometheus exporters |
| **Traffic** | Queries per second | DB/cache metrics |
| **Errors** | Connection errors, query failures | Prometheus exporters |
| **Saturation** | Connection pool usage, replication lag, disk I/O | Prometheus exporters |

---

## 3. The Observability Stack

```
+---------------------------------------------------------------------+
|                        VISUALIZATION                                 |
|  Grafana (dashboards, alerts, explore)                               |
+---------------------------------------------------------------------+
        |                    |                    |
+-------v------+   +--------v-------+   +--------v-------+
|   METRICS    |   |     LOGS       |   |    TRACES      |
|  Prometheus  |   |  Loki          |   |  Tempo         |
|  (15s scrape)|   |  (structured   |   |  (OTLP ingest) |
|  Thanos      |   |   JSON logs)   |   |                |
|  (long-term) |   |                |   |                |
+--------------+   +----------------+   +----------------+
        ^                    ^                    ^
        |                    |                    |
+-------+--------------------+--------------------+-------+
|              OpenTelemetry Collector                     |
|  (receives, processes, exports telemetry data)          |
+----------------------------------------------------------+
        ^                    ^                    ^
+-------+------+   +--------+-------+   +--------+-------+
| Frontend     |   | Backend APIs   |   | Data Layer     |
| (RUM agent)  |   | (OTel SDK)     |   | (exporters)    |
+--------------+   +----------------+   +----------------+
```

### Component Selection

| Component | Choice | Why |
|-----------|--------|-----|
| **Metrics** | Prometheus + Thanos | Industry standard, PromQL, long-term storage with Thanos |
| **Logs** | Loki | Integrates with Grafana, label-based (no full-text indexing cost) |
| **Traces** | Tempo | Trace-by-ID storage (cost-efficient), Grafana native |
| **Collection** | OpenTelemetry | Vendor-neutral, auto-instrumentation, single SDK |
| **Frontend** | Grafana Faro | Self-hosted RUM, integrates with the rest of the stack |
| **Dashboards** | Grafana | Unified view across all three pillars |
| **Alerting** | Alertmanager | Native Prometheus integration, routing, silencing |

---

## 4. SLIs & SLOs

| Layer | SLI | SLO Target | PromQL |
|-------|-----|-----------|--------|
| Frontend | TTI < 3s | 95% of sessions | `sum(rate(rum_tti_seconds_bucket{le="3"}[5m])) / sum(rate(rum_tti_seconds_count[5m]))` |
| API Gateway | HTTP 2xx rate | 99.95% | `sum(rate(http_requests_total{status=~"2.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| API Latency | P95 < 300ms | 99% | `histogram_quantile(0.95, sum by(le)(rate(http_request_duration_seconds_bucket[5m]))) < 0.3` |
| Database | Query P95 < 200ms | 99% | `histogram_quantile(0.95, sum by(le)(rate(db_query_duration_seconds_bucket[5m]))) < 0.2` |
| Cache | Hit rate | > 95% | `sum(rate(cache_hits_total[5m])) / sum(rate(cache_requests_total[5m]))` |

---

## 5. Alerting Strategy

### Multi-Window Burn Rate Alerts

```yaml
groups:
- name: slo-burn-rate
  rules:
  # Fast burn — page immediately
  - alert: APIAvailabilityFastBurn
    expr: |
      (
        sum(rate(http_requests_total{status!~"2.."}[5m]))
        / sum(rate(http_requests_total[5m]))
      ) / (1 - 0.9995) > 14
    for: 2m
    labels:
      severity: critical
      layer: api

  # Slow burn — create ticket
  - alert: APIAvailabilitySlowBurn
    expr: |
      (
        sum(rate(http_requests_total{status!~"2.."}[1h]))
        / sum(rate(http_requests_total[1h]))
      ) / (1 - 0.9995) > 6
    for: 15m
    labels:
      severity: warning
      layer: api
```

### Alert Routing by Layer

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'layer']
  routes:
    - match:
        layer: frontend
      receiver: frontend-team
    - match:
        layer: api
      receiver: backend-team
    - match:
        layer: data
      receiver: dba-team
    - match:
        severity: critical
      receiver: pagerduty-oncall    # Override: all criticals page on-call
```

---

## 6. Correlation: Connecting the Dots

The real power of observability is **correlating** signals across pillars:

### Trace-to-Logs

```
1. User reports slow checkout
2. Find the trace ID from RUM or API logs
3. Search Loki for that trace ID: {trace_id="abc123"}
4. See exact log entries from every service in the request path
```

### Metrics-to-Traces (Exemplars)

```
1. Grafana dashboard shows latency spike at 14:32
2. Click the spike → Prometheus exemplars show trace IDs
3. Click trace ID → opens Tempo trace view
4. See which service/span caused the latency
```

### RUM-to-Backend Correlation

```
1. RUM shows TTI spike for users in India
2. Check: Is it CDN? Network? Backend?
3. Filter backend metrics by region
4. If backend latency is normal → CDN or network issue
5. If backend latency is high → trace to find slow service
```

### Correlation IDs

Every request should carry a correlation ID from frontend to database:

```
Browser (generates X-Request-ID)
  → CDN (passes through)
    → API Gateway (logs it)
      → Service A (logs it, passes in gRPC metadata)
        → Service B (logs it)
          → Database (logged in slow query log)
```

---

## 7. Synthetic Monitoring

In addition to RUM (real users), run synthetic checks:

```yaml
# Blackbox exporter probe config
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]
      method: GET
      fail_if_body_not_matches_regexp:
        - '"status":"ok"'

# Probe targets
scrape_configs:
  - job_name: synthetic
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://api.example.com/healthz
          - https://api.example.com/api/products
          - https://api.example.com/api/checkout/status
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

---

## 8. Architecture Diagram

```
[User Browser/Mobile]
       |
       | RUM Agent (Faro SDK) ──────────────────────┐
       |                                              |
       v                                              v
    [CDN] ──────────────────────────────────> [Grafana Faro Collector]
       |                                              |
       v                                              v
  [API Gateway / Ingress]                      [Loki] [Tempo]
       |    |
       |    └── OTel SDK ──> [OTel Collector] ──> [Prometheus]
       |                          |                    |
       v                          v                    v
  [Microservices]           [Loki] [Tempo]      [Thanos / Mimir]
       |    |                                         |
       |    └── OTel SDK ──> [OTel Collector]         |
       |                                              |
       v                                              v
  [PostgreSQL / Redis]                          [Grafana]
       |                                     (unified dashboard)
       └── Exporters ──> [Prometheus]
```

---

## 9. Interview Scoring Guide

| Category | What They're Looking For |
|----------|------------------------|
| **Completeness** | Covers all layers (frontend to database) |
| **Tool Selection** | Justified choices, not just brand names |
| **SLO-Driven** | Alerts based on SLIs/SLOs, not raw thresholds |
| **Correlation** | Can explain how to trace a user problem across all pillars |
| **Practical** | Mentions real PromQL, real config, not just theory |
| **Trade-offs** | Acknowledges cost, complexity, and operational burden |

### Bonus Points

- Mention **synthetic monitoring** for uptime checks independent of real traffic
- Discuss **chaos injection** to validate observability catches failures
- Explain **exemplars** for connecting metrics to traces
- Reference **OpenTelemetry** as the vendor-neutral collection standard
- Describe **on-call workflows** — how alerts lead to dashboards lead to action
- Mention **cost considerations** — Thanos vs. Mimir, Loki vs. Elasticsearch, log retention policies
