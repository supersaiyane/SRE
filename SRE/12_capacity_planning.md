# Capacity Planning for SRE

Capacity planning ensures your systems can handle current load and future growth without over-provisioning (wasting money) or under-provisioning (causing outages). This guide covers the full process from measurement to scaling strategy.

---

## Why Capacity Planning Matters

| Without It | With It |
|-----------|---------|
| Surprise outages during traffic spikes | Proactive scaling before load arrives |
| Over-provisioned resources wasting money | Right-sized infrastructure |
| Reactive firefighting | Predictable operations |
| Can't answer "can we handle Black Friday?" | Data-driven growth planning |

---

## The Capacity Planning Process

```
Measure Current Usage → Forecast Growth → Model Scenarios → Plan Actions → Validate → Repeat
        1                    2                  3                4            5
```

---

## Step 1: Measure Current Usage

### Key Resources to Track

| Resource | Metric | PromQL Example |
|----------|--------|----------------|
| **CPU** | Usage vs. request vs. limit | `sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace)` |
| **Memory** | Working set vs. limit | `sum(container_memory_working_set_bytes) by (namespace)` |
| **Disk I/O** | Read/write throughput, IOPS | `sum(rate(node_disk_read_bytes_total[5m])) by (instance)` |
| **Network** | Bandwidth, connections | `sum(rate(node_network_receive_bytes_total[5m])) by (instance)` |
| **Application** | Requests/sec, concurrent users | `sum(rate(http_requests_total[5m]))` |
| **Database** | Connections, query latency, storage | `pg_stat_activity_count` |
| **Queue** | Depth, consumer lag | `kafka_consumer_group_lag` |

### Utilization Targets

| Resource | Green | Yellow | Red |
|----------|-------|--------|-----|
| CPU | < 50% | 50-70% | > 70% |
| Memory | < 60% | 60-80% | > 80% |
| Disk | < 70% | 70-85% | > 85% |
| DB Connections | < 60% | 60-80% | > 80% |

**Why not target higher utilization?**
- Headroom for traffic spikes (normal variance is 2-3x)
- Time to scale up (auto-scaler needs minutes)
- Graceful degradation during failures (N-1 redundancy)

---

## Step 2: Forecast Growth

### Data Sources for Forecasting

| Source | What It Tells You |
|--------|-------------------|
| **Historical metrics** | Traffic trends over past 6-12 months |
| **Business roadmap** | Planned launches, marketing campaigns, new regions |
| **Seasonal patterns** | Holiday spikes, end-of-quarter surges |
| **User growth** | Signup rate, active user growth |

### Forecasting Methods

#### Linear Projection
```
Simple: Current load x (1 + monthly_growth_rate)^months

Example:
  Current: 1,000 req/s
  Growth: 10% per month
  6 months: 1,000 x 1.1^6 = 1,771 req/s
```

#### PromQL for Trend Analysis
```promql
# Linear prediction: where will the metric be in 30 days?
predict_linear(
  sum(rate(http_requests_total[1h]))[30d:1h],
  86400 * 30
)
```

#### Seasonal Decomposition
```
Look at the same period last year:
  - Black Friday 2024 was 5x normal traffic
  - Plan for 6x this year (growth + safety margin)
```

---

## Step 3: Model Scenarios

### Scenario Table

| Scenario | Traffic Multiplier | When | Action |
|----------|-------------------|------|--------|
| **Normal** | 1x | Day-to-day | Current capacity |
| **Growth** | 1.5x | 6 months from now | Planned scaling |
| **Spike** | 3x | Marketing campaign | Pre-scale + auto-scale |
| **Peak** | 5-10x | Black Friday / launch | Dedicated capacity plan |
| **Failure** | N-1 | Region or AZ failure | Redundancy verification |

### N+1 / N+2 Redundancy

```
N+1: System survives the loss of 1 component
  - If you need 3 pods to handle load → run 4
  - If you need 2 AZs → deploy in 3

N+2: System survives the loss of 2 components
  - Critical systems (payments, auth)
  - Multi-region deployments
```

---

## Step 4: Scaling Strategies

### Horizontal vs. Vertical Scaling

| Aspect | Horizontal (Scale Out) | Vertical (Scale Up) |
|--------|----------------------|---------------------|
| **How** | Add more instances | Bigger instances |
| **Limit** | Near-infinite (with good architecture) | Hardware ceiling |
| **Cost** | Linear | Exponential (bigger instances cost more per unit) |
| **Downtime** | Zero (rolling) | Usually requires restart |
| **Best for** | Stateless services, web servers | Databases, caches |

### Kubernetes Auto-Scaling

#### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

#### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
```

#### Cluster Autoscaler / Karpenter

```
Cluster Autoscaler:
  - Watches for pending pods that can't be scheduled
  - Adds nodes to the cluster
  - Removes underutilized nodes

Karpenter (AWS):
  - Faster than Cluster Autoscaler (seconds vs minutes)
  - Provisions right-sized nodes (no fixed node groups)
  - Supports spot instances with fallback to on-demand
```

---

## Step 5: Load Testing

### Tools

| Tool | Type | Best For |
|------|------|----------|
| **k6** | Script-based | API load testing, developer-friendly |
| **Locust** | Python-based | Complex user behavior simulation |
| **JMeter** | GUI + CLI | Enterprise, protocol variety |
| **hey** | CLI | Quick HTTP benchmarks |
| **wrk** | CLI | High-throughput HTTP benchmarks |

### Load Test Strategy

```
1. Baseline Test
   - Normal expected traffic for 30 minutes
   - Verify latency P95, error rate, resource usage

2. Stress Test
   - Ramp from 1x to 5x traffic over 15 minutes
   - Find the breaking point
   - Measure degradation curve

3. Spike Test
   - Sudden jump to 10x traffic
   - Verify auto-scaler response time
   - Measure recovery time

4. Soak Test
   - Normal traffic for 4-8 hours
   - Detect memory leaks, connection pool exhaustion
   - Verify long-running stability

5. Failure Test
   - Load test with one AZ/node down
   - Verify N-1 capacity handling
```

### k6 Example Script

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '5m',  target: 100 },   // Ramp up
    { duration: '10m', target: 100 },   // Hold steady
    { duration: '5m',  target: 500 },   // Stress
    { duration: '10m', target: 500 },   // Hold at stress
    { duration: '5m',  target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],    // P95 latency < 300ms
    http_req_failed: ['rate<0.01'],      // Error rate < 1%
  },
};

export default function () {
  const res = http.get('https://api.example.com/health');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'latency < 300ms': (r) => r.timings.duration < 300,
  });
  sleep(1);
}
```

---

## Capacity Planning Dashboard

### Recommended Grafana Panels

| Panel | PromQL | Purpose |
|-------|--------|---------|
| CPU Headroom | `1 - (sum(rate(container_cpu_usage[5m])) / sum(kube_pod_container_resource_limits{resource="cpu"}))` | How much CPU capacity is left |
| Memory Headroom | `1 - (sum(container_memory_working_set_bytes) / sum(kube_pod_container_resource_limits{resource="memory"}))` | How much memory is left |
| Pods vs. Limit | `count(kube_pod_info) / sum(kube_node_status_allocatable{resource="pods"})` | Cluster pod saturation |
| Request Growth | `sum(rate(http_requests_total[1h]))` with 30d range | Traffic trend over time |
| Disk Growth | `predict_linear(node_filesystem_avail_bytes[7d], 86400*30)` | When will disk fill up |

---

## Cost Optimization

| Strategy | Savings | Risk |
|----------|---------|------|
| **Right-sizing** (reduce over-provisioned resources) | 20-40% | Low — based on actual usage data |
| **Spot/Preemptible instances** (stateless workloads) | 60-90% | Medium — instances can be reclaimed |
| **Reserved instances / Savings Plans** | 30-60% | Low — commitment required |
| **Auto-scaling** (scale down during low traffic) | 20-30% | Low — if configured correctly |
| **Bin packing** (consolidate workloads on fewer nodes) | 10-20% | Medium — may affect isolation |

---

## Capacity Review Cadence

| Frequency | What to Review |
|-----------|---------------|
| **Weekly** | Utilization trends, auto-scaler events, cost anomalies |
| **Monthly** | Growth forecasts, upcoming campaigns, budget vs. actual |
| **Quarterly** | Architecture review, reserved instance renewals, capacity model update |
| **Annually** | Multi-year growth plan, major infrastructure decisions |

---

## Interview Tips

When asked about capacity planning:

1. **Start with measurement**: "You can't plan what you can't measure — we track CPU, memory, disk, and request rate with Prometheus"
2. **Mention headroom**: "We target 50-60% utilization to leave room for spikes and failures"
3. **Reference load testing**: "We run monthly load tests at 3x expected traffic to validate our capacity model"
4. **Talk about cost**: "Capacity planning isn't just about having enough — it's about not having too much"
5. **Describe automation**: "HPA handles day-to-day scaling; Karpenter handles node provisioning; we plan for the scenarios automation can't handle"
