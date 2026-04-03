# Process: From "What Matters" to SLI to SLO to SLA

This guide walks through the complete process of defining reliability targets, starting from understanding your users and ending with enforceable SLAs.

---

## The Full Pipeline

```
User Journeys → Service Boundaries → SLIs → SLOs → Error Budget → Error Budget Policy → SLA → Instrument & Dashboard
     1                 2               3       4          5               6               7            8
```

---

## Step 1: Map Critical User Journeys

**What you do**: Identify the interactions that make or break user trust.

**How to do it**:
1. List every user-facing workflow in your product
2. Rank by business impact (revenue, user retention, regulatory)
3. Pick the top 3-5 as your initial SLO candidates

**Example for an e-commerce platform**:

| Journey | Business Impact | Priority |
|---------|----------------|----------|
| Search products | High — users leave if search is slow | P1 |
| Add to cart | High — directly tied to conversion | P1 |
| Checkout & payment | Critical — directly tied to revenue | P0 |
| View order history | Medium — important but not urgent | P2 |
| Account settings | Low — rarely used | P3 |

**Interview sound-bite**: "We start with User-Journey Mapping to keep metrics user-centred, not infrastructure-centred."

---

## Step 2: Decompose into Service Boundaries & Signals

**What you do**: For each journey, identify which service owns it and tie it to one or more of the Four Golden Signals.

**Example decomposition**:

```
Checkout Journey
├── API Gateway        → Traffic (requests/sec)
├── Checkout Service   → Latency (P95 response time)
├── Payment Service    → Errors (failed payment rate)
├── Database           → Saturation (connection pool usage)
└── CDN / Frontend     → Latency (Time to Interactive)
```

**Key principle**: Signals must be controllable by the team that's on-call. Don't assign an SLO to a team that can't affect the metric.

**Golden Signals quick reference**:

| Signal | What It Measures | Example Metric |
|--------|-----------------|----------------|
| **Latency** | How long requests take | `http_request_duration_seconds` |
| **Traffic** | How much demand the system handles | `http_requests_total` |
| **Errors** | How many requests fail | `http_requests_total{status=~"5.."}` |
| **Saturation** | How full your resources are | `container_cpu_usage_seconds_total` |

---

## Step 3: Define SLIs (Formulas)

**What you do**: Turn each signal into a precise, measurable metric expression.

**SLI types and formulas**:

### Availability SLI
```
SLI = (Successful requests / Total requests) x 100

PromQL:
sum(rate(http_requests_total{status=~"2.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

### Latency SLI
```
SLI = (Requests faster than threshold / Total requests) x 100

PromQL:
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))
```

### Quality / Correctness SLI
```
SLI = (Correct responses / Total responses) x 100

(Requires application-level validation)
```

### Freshness SLI (for data pipelines)
```
SLI = (Records processed within threshold / Total records) x 100
```

**Rules for good SLIs**:
- Directly measurable from production telemetry
- Reflects user experience, not internal health
- Has a clear "good event" vs "total event" definition
- Avoidable: don't use metrics you can't control

**Interview sound-bite**: "An SLI is a ratio or percentile derivable from Prometheus — never a boolean up/down."

---

## Step 4: Set SLO Targets

**What you do**: Pick a reliability target for a rolling window that your business cares about.

**How to choose a target**:

1. **Look at historical data**: What has your actual performance been over the last 90 days?
2. **Ask the business**: What level of reliability do users expect?
3. **Consider the cost**: Each additional nine is exponentially more expensive

| Target | Allowed Downtime (30 days) | Cost/Complexity | Typical Use Case |
|--------|---------------------------|-----------------|------------------|
| 99% (2 nines) | 7h 18m | Low | Internal tools, batch jobs |
| 99.9% (3 nines) | 43m 49s | Moderate | Most web applications |
| 99.95% | 21m 54s | High | E-commerce, SaaS platforms |
| 99.99% (4 nines) | 4m 23s | Very High | Payment systems, auth services |
| 99.999% (5 nines) | 26s | Extreme | Core infrastructure, DNS |

**Common windows**:
- **28-day rolling**: Most common. Aligns with monthly reviews. Avoids calendar-month edge cases.
- **7-day rolling**: For fast-moving teams that want quicker feedback loops.
- **Quarterly**: For executive reporting and SLA alignment.

**Interview sound-bite**: "SLO = what we shoot for, shorter than the SLA, guarding feature velocity."

---

## Step 5: Compute the Error Budget

**What you do**: Calculate how much failure you can tolerate.

```
Error Budget = 1 - SLO
```

**Worked example**:

```
SLO = 99.9% (0.999)
Error Budget = 1 - 0.999 = 0.001 (0.1%)

In 30 days = 43,200 minutes:
Allowed failure time = 0.001 x 43,200 = 43.2 minutes

In requests (at 10,000 req/day = 300,000/month):
Allowed failed requests = 0.001 x 300,000 = 300 requests
```

**Why error budgets matter**:
- They turn reliability into a **currency** that product and engineering can negotiate over
- When budget is healthy → ship features faster
- When budget is burning → slow down, fix reliability
- They eliminate the "ops vs dev" conflict by making the tradeoff explicit

**Interview sound-bite**: "Error Budget lets product & ops negotiate release pace using data, not opinions."

---

## Step 6: Write an Error-Budget Policy

**What you do**: Define what happens when the budget is being consumed too fast or is exhausted.

**Example policy**:

| Budget Remaining | Required Actions |
|-----------------|-----------------|
| > 50% | Normal operations. Ship features freely. |
| 25-50% | Review recent changes. Increase monitoring. Run chaos tests. |
| 10-25% | Freeze non-critical releases. Assign reliability sprint. |
| < 10% | Emergency mode. All engineering effort on reliability. |
| 0% (exhausted) | Full release freeze until budget recovers. Mandatory post-mortem. |

**Policy should include**:
- Who has authority to freeze releases (typically SRE lead + engineering manager)
- How to request exceptions (critical security patches, etc.)
- How long the freeze lasts (until X% budget is recovered)
- What "reliability sprint" means concretely (fix top 3 reliability issues)

---

## Step 7: Draft the SLA

**What you do**: Take a subset of SLOs that you're contractually willing to guarantee to customers.

**SLA = SLO with consequences**:

| Aspect | SLO (Internal) | SLA (External) |
|--------|----------------|----------------|
| Target | 99.95% | 99.9% |
| Window | 28-day rolling | Calendar month |
| Consequence | Engineering action | Credits/refunds |
| Audience | Engineering teams | Customers |

**Key rules**:
- SLA should always be **looser** than SLO (if your SLO is 99.95%, your SLA might be 99.9%)
- The gap between SLO and SLA is your safety margin
- SLA adds remedies: refunds, service credits, contract exit clauses
- SLA is a legal document — involve legal/finance in drafting

**Interview sound-bite**: "SLA adds remedies (refunds, credits). It's legal, not just technical."

---

## Step 8: Instrument & Dashboard

**What you do**: Wire up the actual monitoring that tracks everything above.

### Prometheus instrumentation (Node.js example)

```javascript
const client = require('prom-client');

// Track request duration for latency SLI
const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 3, 5]
});

// Track total requests for availability SLI
const httpTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status']
});
```

### Grafana dashboard panels

| Panel | PromQL | Visualization |
|-------|--------|---------------|
| Availability SLI | `sum(rate(http_requests_total{status=~"2.."}[5m])) / sum(rate(http_requests_total[5m]))` | Gauge (target line at SLO) |
| Latency P95 | `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` | Time series |
| Error Budget Remaining | Custom recording rule | Gauge (red/yellow/green) |
| Burn Rate | `error_ratio / error_budget` | Time series with threshold lines |

### Alerting rules

```yaml
groups:
- name: slo-alerts
  rules:
  - alert: FastBurn
    expr: burn_rate_5m > 14
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Error budget burning 14x faster than allowed"

  - alert: SlowBurn
    expr: burn_rate_1h > 6
    for: 15m
    labels:
      severity: warning
      annotations:
        summary: "Sustained error budget burn at 6x rate"
```

---

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| SLO on infrastructure metrics (CPU, memory) | Users don't care about CPU | Use user-facing SLIs |
| Same SLO for all services | Not all services are equally critical | Tier your services |
| SLO = SLA | No safety margin | SLO should be tighter than SLA |
| Too many SLOs | Cognitive overload, alert fatigue | 2-4 SLOs per service max |
| No error budget policy | SLOs without consequences are just dashboards | Define actions per budget level |
| Setting 99.999% because "we want to be reliable" | Unrealistic, prevents all deployments | Start with 99.9% and tighten based on data |

---

## Interview Walkthrough

When asked "How would you implement SLOs for a checkout service?":

1. **Start with the user journey**: "Checkout is our most critical flow — it directly generates revenue"
2. **Define the SLI**: "We measure availability as the ratio of successful checkout requests to total checkout requests"
3. **Set the SLO**: "Based on our historical 99.97% success rate, we set the SLO at 99.95% over a 28-day window"
4. **Calculate budget**: "That gives us about 21 minutes of allowed downtime per month"
5. **Define policy**: "Below 50% budget we freeze non-critical releases; at 0% we go full reliability sprint"
6. **Instrument**: "We use Prometheus with burn-rate alerts at 14x/5m and 6x/1h"
7. **SLA**: "Our external SLA promises 99.9% — giving us a 0.05% safety margin"
