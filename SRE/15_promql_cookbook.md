# PromQL Cookbook — Practical Queries for SRE

A copy-paste-ready collection of PromQL queries organized by use case. Every query includes what it does, when to use it, and how to customize it.

---

## How PromQL Works — 30-Second Primer

```
Metric Types:
  Counter   → only goes up (requests_total, errors_total)
  Gauge     → goes up and down (temperature, memory_usage)
  Histogram → counts values in buckets (request_duration_seconds)
  Summary   → pre-calculated percentiles (rarely used now)

Key Functions:
  rate()    → per-second rate of a counter over a window
  increase()→ total increase of a counter over a window
  sum()     → aggregate across labels
  avg()     → average across labels
  histogram_quantile() → calculate percentiles from histograms
  by ()     → group results by label
  without() → aggregate removing specific labels
```

---

## Golden Signals

### Traffic — Requests Per Second

```promql
# Total requests per second
sum(rate(http_requests_total[5m]))

# Requests per second by status code
sum(rate(http_requests_total[5m])) by (status)

# Requests per second by service
sum(rate(http_requests_total[5m])) by (job)

# Requests per second by endpoint
sum(rate(http_requests_total[5m])) by (method, route)

# Top 10 busiest endpoints
topk(10, sum(rate(http_requests_total[5m])) by (route))
```

### Errors — Error Rate

```promql
# Error ratio (0-1 scale)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Error percentage (0-100)
100 * sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Error rate by endpoint (find which endpoint is failing)
sum(rate(http_requests_total{status=~"5.."}[5m])) by (route)
/
sum(rate(http_requests_total[5m])) by (route)

# Client errors (4xx) vs server errors (5xx)
sum(rate(http_requests_total{status=~"4.."}[5m]))  # Client errors
sum(rate(http_requests_total{status=~"5.."}[5m]))  # Server errors

# Error rate excluding 404s (404s are often expected)
sum(rate(http_requests_total{status=~"5..",status!="404"}[5m]))
/
sum(rate(http_requests_total[5m]))
```

### Latency — Response Time Percentiles

```promql
# P50 (median) latency
histogram_quantile(0.50,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# P90 latency
histogram_quantile(0.90,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# P95 latency
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# P99 latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# P99 latency per endpoint (find the slow endpoint)
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
)

# Average latency (less useful than percentiles, but simple)
sum(rate(http_request_duration_seconds_sum[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))

# Latency per service
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
)
```

### Saturation — Resource Usage

```promql
# CPU usage by pod (cores)
sum(rate(container_cpu_usage_seconds_total{container!="POD",container!=""}[5m])) by (pod)

# CPU usage as percentage of limit
100 * sum(rate(container_cpu_usage_seconds_total{container!="POD"}[5m])) by (pod)
/
sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod)

# Memory usage by pod (bytes)
sum(container_memory_working_set_bytes{container!="POD",container!=""}) by (pod)

# Memory usage as percentage of limit
100 * sum(container_memory_working_set_bytes{container!="POD"}) by (pod)
/
sum(kube_pod_container_resource_limits{resource="memory"}) by (pod)

# Disk usage percentage (node level)
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

# Network received bytes per second (node)
sum(rate(node_network_receive_bytes_total[5m])) by (instance)
```

---

## SLO & Error Budget

### Availability SLI

```promql
# Availability over last 30 days
sum(increase(http_requests_total{status=~"2.."}[30d]))
/
sum(increase(http_requests_total[30d]))

# Availability over last 1 hour (for dashboards)
sum(rate(http_requests_total{status=~"2.."}[1h]))
/
sum(rate(http_requests_total[1h]))

# Availability by service
sum(rate(http_requests_total{status=~"2.."}[5m])) by (job)
/
sum(rate(http_requests_total[5m])) by (job)
```

### Burn Rate

```promql
# Burn rate (SLO = 99.9%, error budget = 0.001)
(
  sum(rate(http_requests_total{status!~"2.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
/ 0.001

# Fast burn alert (14x over 5 minutes)
(
  sum(rate(http_requests_total{status!~"2.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) / 0.001 > 14

# Slow burn alert (6x over 1 hour)
(
  sum(rate(http_requests_total{status!~"2.."}[1h]))
  /
  sum(rate(http_requests_total[1h]))
) / 0.001 > 6

# Burn rate per service
(
  sum(rate(http_requests_total{status!~"2.."}[1h])) by (job)
  /
  sum(rate(http_requests_total[1h])) by (job)
) / 0.001
```

### Error Budget Remaining

```promql
# Error budget consumed (as a ratio)
# If > 1.0, budget is exhausted
(
  1 - (
    sum(increase(http_requests_total{status=~"2.."}[30d]))
    /
    sum(increase(http_requests_total[30d]))
  )
) / 0.001

# Error budget remaining percentage
100 * (1 - (
  (1 - (
    sum(increase(http_requests_total{status=~"2.."}[30d]))
    /
    sum(increase(http_requests_total[30d]))
  ))
  / 0.001
))
```

---

## Kubernetes Cluster Health

### Pod Status

```promql
# Number of pods not in Running state
sum(kube_pod_status_phase{phase!="Running",phase!="Succeeded"}) by (namespace, phase)

# Pods in CrashLoopBackOff
sum(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}) by (namespace, pod)

# Pod restart count (high restarts = instability)
sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod) > 3

# Pods stuck in Pending
sum(kube_pod_status_phase{phase="Pending"}) by (namespace) > 0

# OOMKilled containers in the last hour
sum(increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[1h])) by (namespace, pod)
```

### Node Health

```promql
# Node readiness (0 = not ready)
kube_node_status_condition{condition="Ready",status="true"}

# Count of not-ready nodes
count(kube_node_status_condition{condition="Ready",status="true"} == 0)

# Node CPU utilization
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory utilization
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))

# Node disk utilization
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

# Disk will be full in N days (linear prediction)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[7d], 86400 * 30) < 0
```

### Deployment Health

```promql
# Deployments with unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# Deployments not at desired replica count
kube_deployment_spec_replicas != kube_deployment_status_replicas_available

# HPA at max capacity (can't scale further)
kube_horizontalpodautoscaler_status_current_replicas
==
kube_horizontalpodautoscaler_spec_max_replicas
```

---

## Database Monitoring

### PostgreSQL (with postgres_exporter)

```promql
# Active connections
pg_stat_activity_count{state="active"}

# Connection utilization (% of max)
100 * pg_stat_activity_count / pg_settings_max_connections

# Transactions per second
sum(rate(pg_stat_database_xact_commit[5m])) + sum(rate(pg_stat_database_xact_rollback[5m]))

# Cache hit ratio (should be > 99%)
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read) * 100

# Replication lag (seconds)
pg_replication_lag

# Slow queries (requires pg_stat_statements)
# Track in application metrics instead
```

### Redis (with redis_exporter)

```promql
# Connected clients
redis_connected_clients

# Memory usage
redis_memory_used_bytes

# Cache hit ratio
redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)

# Commands per second
rate(redis_commands_processed_total[5m])

# Evicted keys (memory pressure signal)
rate(redis_evicted_keys_total[5m])
```

---

## Alerting Rules — Ready to Use

### High Error Rate

```yaml
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m])) > 0.01
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 1%"
    description: "{{ $labels.job }} has {{ $value | humanizePercentage }} error rate"
```

### High Latency

```yaml
- alert: HighLatencyP95
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
    ) > 0.5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "P95 latency above 500ms"
    description: "{{ $labels.job }} P95 latency is {{ $value | humanizeDuration }}"
```

### Pod Crash Looping

```yaml
- alert: PodCrashLooping
  expr: |
    sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod) > 5
  for: 0m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.pod }} is crash-looping"
    description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted {{ $value }} times in the last hour"
```

### Disk Running Out

```yaml
- alert: DiskWillFillIn4Days
  expr: |
    predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 86400 * 4) < 0
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Disk on {{ $labels.instance }} will fill within 4 days"
```

### HPA Maxed Out

```yaml
- alert: HPAMaxedOut
  expr: |
    kube_horizontalpodautoscaler_status_current_replicas
    ==
    kube_horizontalpodautoscaler_spec_max_replicas
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "HPA {{ $labels.horizontalpodautoscaler }} is at max replicas"
```

### Node Not Ready

```yaml
- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Node {{ $labels.node }} is not ready"
```

---

## Useful PromQL Patterns

### Safe Division (avoid divide-by-zero)

```promql
# Bad: will return NaN if denominator is 0
a / b

# Good: returns 0 when denominator is 0
a / (b > 0) or vector(0)

# Alternative: filter out low-traffic cases entirely
a / b > 0 and sum(rate(http_requests_total[5m])) > 1
```

### Rate Interval Best Practice

```promql
# In Grafana, always use $__rate_interval instead of hardcoded windows
# It automatically adjusts based on scrape interval and dashboard resolution
sum(rate(http_requests_total[$__rate_interval])) by (status)

# For alerting rules, use explicit windows
# 5m for fast detection, 1h for stable trends
```

### Aggregation Across Labels

```promql
# Sum across all labels (total)
sum(metric)

# Sum keeping specific labels
sum(metric) by (job, status)

# Sum removing specific labels
sum(metric) without (instance, pod)

# Count number of time series
count(up{job="my-service"})
```

### Comparing Over Time

```promql
# Current value vs. 1 day ago
metric - metric offset 1d

# Current value vs. 1 week ago
metric - metric offset 7d

# Percentage change from 1 week ago
100 * (metric - metric offset 7d) / metric offset 7d
```

### Label Filtering

```promql
# Exact match
metric{status="200"}

# Regex match
metric{status=~"2.."}           # Any 2xx
metric{status=~"4..|5.."}      # Any 4xx or 5xx
metric{status!~"2.."}          # NOT 2xx (errors)

# Multiple labels
metric{job="api", status=~"5..", method="POST"}
```

---

## Quick Reference Card

| What You Want | PromQL Pattern |
|---------------|---------------|
| Rate of counter | `rate(counter[5m])` |
| Total increase | `increase(counter[1h])` |
| Percentile | `histogram_quantile(0.99, sum(rate(bucket[5m])) by (le))` |
| Ratio | `sum(rate(a[5m])) / sum(rate(b[5m]))` |
| Top N | `topk(10, metric)` |
| Bottom N | `bottomk(10, metric)` |
| Predict future | `predict_linear(metric[7d], 86400*30)` |
| Compare to past | `metric - metric offset 7d` |
| Count series | `count(metric)` |
| Check if exists | `absent(metric)` → returns 1 if metric is missing |
| Clamp values | `clamp_min(metric, 0)` / `clamp_max(metric, 100)` |
