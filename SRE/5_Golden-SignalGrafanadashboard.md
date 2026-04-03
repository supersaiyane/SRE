# Golden Signal Grafana Dashboard — Complete Setup Guide

This guide walks through creating a production-ready Grafana dashboard for the four Golden Signals, including PromQL queries, panel configuration, and an error-budget burn-down gauge.

---

## Dashboard Overview

```
+---------------------------+---------------------------+
|   Traffic (req/sec)       |   Errors (% by status)    |
|   Time series by status   |   Time series + threshold |
+---------------------------+---------------------------+
|   Latency (P50/P90/P99)  |   Saturation (CPU/Memory) |
|   Multi-line time series  |   Gauge + time series     |
+---------------------------+---------------------------+
|          Error Budget Burn-Down (Gauge)               |
+-------------------------------------------------------+
```

---

## Panel 1: Traffic — Requests/sec by Status

**Visualization**: Time series (stacked bar or lines)

**PromQL**:
```promql
sum(rate(http_requests_total{job="checkout"}[$__rate_interval])) by (status)
```

**Panel settings**:
- Legend: `{{status}}`
- Unit: `reqps` (requests per second)
- Color overrides:
  - `2xx` → green
  - `3xx` → blue
  - `4xx` → orange
  - `5xx` → red

**Why this matters**: Traffic is your demand signal. A sudden drop means users can't reach you. A sudden spike means you might be under attack or a campaign just launched.

**Additional query — total traffic trend line**:
```promql
sum(rate(http_requests_total{job="checkout"}[$__rate_interval]))
```

---

## Panel 2: Errors — Error Ratio (%)

**Visualization**: Time series with threshold line

**PromQL — Error ratio**:
```promql
100 * (1 - (
  sum(rate(http_requests_total{job="checkout",status=~"2.."}[$__rate_interval])) /
  sum(rate(http_requests_total{job="checkout"}[$__rate_interval]))
))
```

**Panel settings**:
- Unit: `percent`
- Threshold lines:
  - Green: < 0.1% (within SLO)
  - Yellow: 0.1% - 1% (warning)
  - Red: > 1% (SLO breach)

**Additional query — errors by endpoint**:
```promql
sum(rate(http_requests_total{job="checkout",status=~"5.."}[$__rate_interval])) by (route)
```

---

## Panel 3: Latency — P50 / P90 / P99 Overlay

**Visualization**: Time series (multi-line overlay)

**PromQL — P99**:
```promql
histogram_quantile(0.99, sum by (le) (
  rate(http_request_duration_seconds_bucket{job="checkout"}[$__rate_interval])
))
```

**PromQL — P90**:
```promql
histogram_quantile(0.90, sum by (le) (
  rate(http_request_duration_seconds_bucket{job="checkout"}[$__rate_interval])
))
```

**PromQL — P50 (median)**:
```promql
histogram_quantile(0.50, sum by (le) (
  rate(http_request_duration_seconds_bucket{job="checkout"}[$__rate_interval])
))
```

**Panel settings**:
- Unit: `seconds` (Grafana auto-formats to ms when appropriate)
- Legend: `P50`, `P90`, `P99`
- SLO threshold line: horizontal at your latency SLO (e.g., 300ms)
- Colors: P50 green, P90 yellow, P99 red

**Why overlay all three**:
- P50 = typical user experience
- P90 = "most users" experience
- P99 = worst-case experience (often caused by garbage collection, cold cache, or slow queries)
- A gap between P50 and P99 suggests tail latency issues

---

## Panel 4: Saturation — CPU / Memory Usage

**Visualization**: Time series + gauge

### CPU Usage by Pod

```promql
avg(rate(container_cpu_usage_seconds_total{pod=~"checkout-.*"}[$__rate_interval])) by (pod)
```

### Memory Usage by Pod

```promql
avg(container_memory_working_set_bytes{pod=~"checkout-.*"}) by (pod)
```

### CPU Utilization vs. Limit (%)

```promql
100 * (
  sum(rate(container_cpu_usage_seconds_total{pod=~"checkout-.*"}[$__rate_interval]))
  /
  sum(kube_pod_container_resource_limits{pod=~"checkout-.*",resource="cpu"})
)
```

### Memory Utilization vs. Limit (%)

```promql
100 * (
  sum(container_memory_working_set_bytes{pod=~"checkout-.*"})
  /
  sum(kube_pod_container_resource_limits{pod=~"checkout-.*",resource="memory"})
)
```

**Panel settings**:
- Gauges for current utilization %
- Time series for trends
- Thresholds: Green < 60%, Yellow 60-80%, Red > 80%

---

## Panel 5: Error-Budget Burn-Down

**Visualization**: Gauge or Stat panel

### Recording Rule (add to Prometheus rules)

First, create a recording rule so the dashboard query is efficient:

```yaml
groups:
- name: slo-recording-rules
  interval: 30s
  rules:
    # Current availability SLI over 30 days
    - record: slo:availability:ratio30d
      expr: |
        sum(rate(http_requests_total{job="checkout",status=~"2.."}[30d]))
        /
        sum(rate(http_requests_total{job="checkout"}[30d]))

    # Error budget remaining (percentage)
    - record: slo:error_budget_remaining:ratio
      expr: |
        1 - (
          (1 - slo:availability:ratio30d)
          /
          (1 - 0.999)
        )
```

### Dashboard Query — Budget Remaining %

```promql
slo:error_budget_remaining:ratio * 100
```

**Panel settings**:
- Type: Gauge or Stat
- Unit: `percent`
- Thresholds:
  - Green: > 50%
  - Yellow: 20-50%
  - Red: < 20%
- Min: 0, Max: 100

### Dashboard Query — Burn Rate (current)

```promql
(
  sum(rate(http_requests_total{job="checkout",status!~"2.."}[1h]))
  /
  sum(rate(http_requests_total{job="checkout"}[1h]))
)
/
(1 - 0.999)
```

**Panel settings**:
- Type: Stat
- Thresholds:
  - Green: < 1 (within budget)
  - Yellow: 1-6 (slow burn)
  - Red: > 6 (fast burn, need action)

---

## Dashboard Variables

Add these as Grafana template variables for flexibility:

| Variable | Type | Query | Purpose |
|----------|------|-------|---------|
| `$job` | Query | `label_values(http_requests_total, job)` | Select service |
| `$namespace` | Query | `label_values(kube_pod_info, namespace)` | Select K8s namespace |
| `$interval` | Interval | `$__rate_interval` | Auto-adjust for scrape interval |

Then replace hardcoded values in queries:
```promql
sum(rate(http_requests_total{job="$job"}[$__rate_interval])) by (status)
```

---

## Alerting from Dashboard

### Configure alert rules directly in Grafana or via Alertmanager:

```yaml
groups:
- name: golden-signal-alerts
  rules:
  - alert: HighErrorRate
    expr: |
      (1 - (
        sum(rate(http_requests_total{job="checkout",status=~"2.."}[5m]))
        / sum(rate(http_requests_total{job="checkout"}[5m]))
      )) > 0.01
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Error rate above 1% for checkout"
      dashboard: "https://grafana.example.com/d/golden-signals"

  - alert: HighLatencyP99
    expr: |
      histogram_quantile(0.99, sum by (le) (
        rate(http_request_duration_seconds_bucket{job="checkout"}[5m])
      )) > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "P99 latency above 500ms for checkout"

  - alert: HighSaturation
    expr: |
      100 * sum(rate(container_cpu_usage_seconds_total{pod=~"checkout-.*"}[5m]))
      / sum(kube_pod_container_resource_limits{pod=~"checkout-.*",resource="cpu"})
      > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "CPU utilization above 80% for checkout pods"
```

---

## Importing the Dashboard

### Option 1: JSON Model

Export your dashboard as JSON (`Dashboard Settings → JSON Model → Copy`) and check it into your repo:

```
monitoring/
  dashboards/
    golden-signals.json
  provisioning/
    dashboards.yaml
```

### Option 2: Grafana Provisioning

```yaml
# provisioning/dashboards.yaml
apiVersion: 1
providers:
  - name: 'SRE Dashboards'
    folder: 'SRE'
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

### Option 3: Grafana Dashboard as Code (Grafonnet / Jsonnet)

For large teams, consider generating dashboards programmatically using Grafonnet to ensure consistency across services.

---

## Quick Reference — All Queries in One Place

| Signal | Query |
|--------|-------|
| Traffic (total) | `sum(rate(http_requests_total{job="$job"}[$__rate_interval]))` |
| Traffic (by status) | `sum(rate(http_requests_total{job="$job"}[$__rate_interval])) by (status)` |
| Error ratio (%) | `100 * (1 - sum(rate(http_requests_total{job="$job",status=~"2.."}[$__rate_interval])) / sum(rate(http_requests_total{job="$job"}[$__rate_interval])))` |
| Latency P99 | `histogram_quantile(0.99, sum by (le)(rate(http_request_duration_seconds_bucket{job="$job"}[$__rate_interval])))` |
| Latency P95 | Same as above with `0.95` |
| Latency P50 | Same as above with `0.50` |
| CPU % of limit | `100 * sum(rate(container_cpu_usage_seconds_total{pod=~"$job-.*"}[$__rate_interval])) / sum(kube_pod_container_resource_limits{pod=~"$job-.*",resource="cpu"})` |
| Memory % of limit | `100 * sum(container_memory_working_set_bytes{pod=~"$job-.*"}) / sum(kube_pod_container_resource_limits{pod=~"$job-.*",resource="memory"})` |
| Burn rate | `(sum(rate(http_requests_total{job="$job",status!~"2.."}[1h])) / sum(rate(http_requests_total{job="$job"}[1h]))) / (1 - 0.999)` |
| Budget remaining | `slo:error_budget_remaining:ratio * 100` |
