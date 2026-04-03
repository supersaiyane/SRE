# Distributed Systems Reliability Patterns

These are the building blocks for making distributed systems resilient. Every SRE should know when and how to apply each pattern. This guide covers the pattern, when to use it, how to implement it, and the trade-offs.

---

## Pattern Map

```
Incoming Request
    │
    ├── Rate Limiting          → Protect from overload
    ├── Load Shedding          → Drop low-priority work under pressure
    │
    ▼
  Your Service
    │
    ├── Timeout                → Don't wait forever for dependencies
    ├── Retry + Backoff        → Handle transient failures
    ├── Circuit Breaker        → Stop calling a broken dependency
    ├── Bulkhead               → Isolate failures to one component
    ├── Fallback / Degradation → Return something useful when broken
    │
    ▼
  Downstream Dependency
```

---

## 1. Timeouts

**What**: Set a maximum time to wait for a response from a dependency. If exceeded, fail the request.

**Why**: Without timeouts, a slow dependency can hold connections open indefinitely, eventually exhausting your connection pool and bringing down your entire service.

**Guidelines**:

| Context | Timeout | Rationale |
|---------|---------|-----------|
| HTTP API call | 1-5 seconds | Users won't wait longer |
| Database query | 5-30 seconds | Complex queries may be slow but shouldn't block forever |
| Background job | 30-300 seconds | More tolerance, but still needs a limit |
| Health check | 2-5 seconds | Must be fast to be useful |

**Implementation (Node.js)**:
```javascript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 3000); // 3s timeout

try {
  const response = await fetch('https://payment-service/charge', {
    signal: controller.signal,
  });
  clearTimeout(timeout);
  return response.json();
} catch (err) {
  if (err.name === 'AbortError') {
    // Timeout — return error or fallback
    throw new Error('Payment service timeout');
  }
  throw err;
}
```

**Implementation (Istio VirtualService)**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - timeout: 3s
      route:
        - destination:
            host: payment-service
```

**Trade-offs**:
- Too short → false failures during normal slow responses
- Too long → cascading failures when dependency is actually down
- Rule of thumb: set timeout at P99 latency + small buffer

---

## 2. Retries with Exponential Backoff and Jitter

**What**: When a request fails, retry it — but wait increasingly longer between attempts, with random jitter to avoid thundering herd.

**Why**: Many failures are transient (network blip, temporary overload). A single retry often succeeds. But naive retries (immediate, without backoff) can amplify the problem.

**Retry Decision Tree**:
```
Request failed?
├── Is the error retryable? (5xx, timeout, connection reset)
│   ├── Yes → Retry with backoff
│   └── No (4xx, auth failure) → Don't retry, return error
├── Have we exceeded max retries?
│   ├── Yes → Return error, open circuit breaker
│   └── No → Wait, then retry
```

**Backoff formula**:
```
wait_time = min(base_delay * 2^attempt + random_jitter, max_delay)

Example (base=100ms, max=30s):
  Attempt 1: 100ms + jitter  = ~150ms
  Attempt 2: 200ms + jitter  = ~280ms
  Attempt 3: 400ms + jitter  = ~520ms
  Attempt 4: 800ms + jitter  = ~950ms
  ...
  Attempt 8: 25.6s + jitter = ~27s (capped at 30s)
```

**Why jitter matters**:
```
Without jitter:                    With jitter:
  All clients retry at 100ms         Client A retries at 87ms
  All clients retry at 200ms         Client B retries at 134ms
  All clients retry at 400ms         Client C retries at 112ms
  → Thundering herd on recovery      → Spread load, smooth recovery
```

**Implementation (pseudocode)**:
```python
import random
import time

def retry_with_backoff(func, max_retries=5, base_delay=0.1, max_delay=30):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableError:
            if attempt == max_retries - 1:
                raise
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay * 0.5)
            time.sleep(delay + jitter)
```

**Implementation (Istio)**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
      route:
        - destination:
            host: payment-service
```

**When NOT to retry**:
- Non-idempotent operations (POST that creates a resource) — unless you have idempotency keys
- Auth failures (401/403) — retrying won't help
- Validation errors (400) — fix the request instead
- Resource not found (404) — it's not coming back

---

## 3. Circuit Breaker

**What**: Monitor calls to a dependency. If failures exceed a threshold, "open" the circuit — stop calling the dependency entirely and fail fast. After a cooldown period, allow a test request through. If it succeeds, close the circuit.

**Why**: When a dependency is down, continuing to call it wastes resources, increases latency, and can prevent recovery (flooding a struggling service with requests).

**State Machine**:
```
                    failure threshold exceeded
    ┌─────────┐  ──────────────────────────────►  ┌─────────┐
    │  CLOSED │                                    │  OPEN   │
    │ (normal)│  ◄──────────────────────────────  │ (fail   │
    └─────────┘     test request succeeds          │  fast)  │
         ▲                                         └────┬────┘
         │                                              │
         │         cooldown timer expires               │
         │              ┌──────────┐                    │
         └──────────────│HALF-OPEN │◄───────────────────┘
           test passes  │(test one)│
                        └──────────┘
```

**Configuration**:

| Parameter | Typical Value | Description |
|-----------|--------------|-------------|
| Failure threshold | 5 failures in 60s | Number of failures to trip the circuit |
| Cooldown period | 30-60 seconds | Time before attempting a test request |
| Success threshold | 3 consecutive | Successes needed to close the circuit |
| Timeout | 3 seconds | Request timeout that counts as a failure |

**Implementation (Istio DestinationRule)**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5        # Trip after 5 consecutive 5xx
      interval: 30s                  # Check every 30s
      baseEjectionTime: 60s          # Remove from pool for 60s
      maxEjectionPercent: 50         # Never eject more than 50% of hosts
```

**What to do when circuit is open**:
- Return a cached response (if available)
- Return a degraded response (see Fallback pattern)
- Return an error immediately (fail fast — still better than hanging)
- Queue the request for later processing

---

## 4. Bulkhead

**What**: Isolate components so that a failure in one doesn't bring down the others. Like compartments in a ship — if one floods, the others stay dry.

**Why**: Without isolation, a single slow dependency can consume all threads/connections and affect unrelated endpoints.

**Types of Bulkheads**:

### Thread Pool Bulkhead
```
Total threads: 100
├── /checkout endpoint:  30 threads max
├── /search endpoint:    40 threads max
├── /profile endpoint:   20 threads max
└── Reserved:            10 threads

If /search gets slow and uses all 40 threads,
/checkout and /profile are unaffected.
```

### Connection Pool Bulkhead
```
Database connections: 100 total
├── Critical path (checkout): dedicated pool of 40
├── Analytics queries: dedicated pool of 30
├── Background jobs: dedicated pool of 20
└── Reserved: 10

If analytics runs a slow query using all 30 connections,
checkout still has its dedicated 40.
```

### Kubernetes Bulkhead
```yaml
# Resource limits act as bulkheads
resources:
  requests:
    cpu: 500m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

# PodDisruptionBudget prevents too many pods going down at once
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: checkout

# Separate critical workloads onto dedicated node pools
nodeSelector:
  workload-type: critical
tolerations:
  - key: "dedicated"
    value: "critical"
    effect: "NoSchedule"
```

### Namespace Isolation
```yaml
# ResourceQuota prevents one team from consuming all cluster resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

---

## 5. Rate Limiting

**What**: Limit the number of requests a client can make in a given time window. Reject excess requests with 429 (Too Many Requests).

**Why**: Protects your service from abuse, accidental overload, and ensures fair resource sharing between clients.

**Algorithms**:

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Token Bucket** | Bucket fills at constant rate; each request takes a token | Allows bursts up to bucket size |
| **Sliding Window** | Count requests in a sliding time window | Smooth rate enforcement |
| **Fixed Window** | Count requests per fixed time interval | Simple but allows burst at window boundary |
| **Leaky Bucket** | Process requests at a fixed rate, queue excess | Smooth output rate |

**Implementation (NGINX)**:
```nginx
http {
    # Define rate limit zone: 10 requests per second per IP
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    server {
        location /api/ {
            # Allow burst of 20, delay excess requests
            limit_req zone=api burst=20 delay=10;
            limit_req_status 429;
        }
    }
}
```

**Implementation (Kubernetes — Istio)**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit
spec:
  workloadSelector:
    labels:
      app: api-gateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            stat_prefix: http_local_rate_limiter
            token_bucket:
              max_tokens: 100
              tokens_per_fill: 10
              fill_interval: 1s
```

**Rate limit headers to return**:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1617235200
```

---

## 6. Load Shedding

**What**: When the system is overloaded, proactively drop low-priority requests to protect high-priority ones.

**Why**: Under extreme load, it's better to serve some requests well than all requests poorly.

**Priority Levels**:

| Priority | Examples | Under Pressure |
|----------|---------|---------------|
| **Critical** | Checkout, payment, auth | Always serve |
| **High** | Search, product pages | Serve if capacity allows |
| **Medium** | Recommendations, reviews | Shed early |
| **Low** | Analytics, telemetry | Shed first |

**Implementation approach**:
```
1. Measure current load (request queue depth, CPU, concurrent requests)
2. Define thresholds per priority level
3. When threshold exceeded:
   - Reject lowest priority requests first (return 503)
   - Include Retry-After header
   - Log shed events for monitoring
```

**Kubernetes approach — PriorityClass**:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
globalDefault: false
description: "Critical services (checkout, payments)"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard
value: 100000
globalDefault: true
description: "Standard workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 10000
description: "Batch jobs that can be preempted"
```

---

## 7. Fallback / Graceful Degradation

**What**: When a dependency fails, return a degraded but useful response instead of an error.

**Why**: A partial experience is almost always better than a complete failure.

**Fallback Strategies**:

| Strategy | Example | When to Use |
|----------|---------|-------------|
| **Cached response** | Return last-known product price | Data doesn't change frequently |
| **Default value** | Show "Shipping: calculated at checkout" | Non-critical field |
| **Simplified response** | Return products without personalized recommendations | Recommendation service is down |
| **Static content** | Serve a static product catalog from CDN | Full backend outage |
| **Queue for later** | Accept the order, process payment later | Payment gateway temporarily down |
| **Feature flag** | Disable feature entirely | New feature causing issues |

**Implementation pattern**:
```python
async def get_product(product_id):
    try:
        # Try the primary source
        product = await product_service.get(product_id)
        cache.set(f"product:{product_id}", product, ttl=3600)
        return product
    except (TimeoutError, ServiceUnavailable):
        # Fallback 1: Return cached version
        cached = cache.get(f"product:{product_id}")
        if cached:
            cached['_stale'] = True  # Mark as stale
            return cached
        # Fallback 2: Return minimal response
        return {
            'id': product_id,
            'name': 'Product details temporarily unavailable',
            'price': None,
            '_degraded': True
        }
```

---

## 8. Backpressure

**What**: When a system is overloaded, signal upstream to slow down instead of dropping requests silently.

**Why**: Without backpressure, producers keep sending data at full speed, overwhelming consumers and causing data loss or OOM crashes.

**Backpressure Mechanisms**:

| Mechanism | How It Works | Example |
|-----------|-------------|---------|
| **Bounded queues** | Queue fills up → producer blocks or gets rejected | Kafka with `max.block.ms` |
| **Flow control** | Consumer tells producer to slow down | TCP flow control, gRPC flow control |
| **Rate limiting at source** | Limit how fast producers can send | API rate limits on ingest endpoints |
| **Scaling consumers** | Add more consumers when queue grows | HPA based on queue depth |

**Kubernetes HPA based on queue depth**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumer_group_lag
          selector:
            matchLabels:
              topic: orders
        target:
          type: AverageValue
          averageValue: "100"    # Scale up when lag > 100 per pod
```

---

## 9. Idempotency

**What**: An operation is idempotent if performing it multiple times produces the same result as performing it once.

**Why**: In distributed systems, retries are inevitable. Without idempotency, retries can cause duplicate charges, double-sends, or data corruption.

**Implementation — Idempotency Key**:
```
Client sends:
  POST /payments
  Idempotency-Key: abc-123-def
  { "amount": 100, "currency": "USD" }

Server behavior:
  1. Check if abc-123-def was already processed
  2. If yes → return the same response (don't charge again)
  3. If no  → process payment, store result keyed by abc-123-def
```

**Database pattern**:
```sql
-- Use unique constraint to prevent duplicates
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE,  -- Prevents duplicates
    amount DECIMAL(10,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

-- INSERT will fail if idempotency_key already exists
INSERT INTO payments (id, idempotency_key, amount, status)
VALUES (gen_random_uuid(), 'abc-123-def', 100.00, 'pending')
ON CONFLICT (idempotency_key) DO NOTHING;
```

---

## 10. Health Checks

**What**: Endpoints that report whether a service is alive and ready to serve traffic.

**Types**:

| Check | Purpose | K8s Probe | Example |
|-------|---------|-----------|---------|
| **Liveness** | Is the process alive? | `livenessProbe` | Simple "pong" response |
| **Readiness** | Can it serve traffic? | `readinessProbe` | DB connected, cache warm |
| **Startup** | Has it finished starting? | `startupProbe` | Migrations done, init complete |

**Implementation**:
```yaml
containers:
  - name: api
    image: api:v1
    ports:
      - containerPort: 8080

    # Don't restart during slow startup
    startupProbe:
      httpGet:
        path: /healthz/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # Restart if process is hung
    livenessProbe:
      httpGet:
        path: /healthz/live
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3

    # Remove from service if not ready
    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 2
```

**What each endpoint checks**:
```python
# /healthz/live — Am I alive?
# Should be LIGHTWEIGHT. Just return 200.
# Don't check dependencies here (that's readiness).
def liveness():
    return {"status": "ok"}, 200

# /healthz/ready — Can I serve traffic?
# Check critical dependencies.
def readiness():
    db_ok = check_database_connection()
    cache_ok = check_cache_connection()
    if db_ok and cache_ok:
        return {"status": "ready"}, 200
    return {"status": "not ready", "db": db_ok, "cache": cache_ok}, 503

# /healthz/startup — Am I done starting?
# Check one-time initialization tasks.
def startup():
    if migrations_complete and config_loaded:
        return {"status": "started"}, 200
    return {"status": "starting"}, 503
```

**Common mistakes**:
- Putting dependency checks in liveness probe → if DB goes down, all pods restart → makes things worse
- Not having a startup probe → slow-starting apps get killed by liveness probe before they're ready
- Readiness probe too strict → minor blips cause traffic drops

---

## Pattern Combinations

Most real-world systems combine multiple patterns:

```
Request arrives
  → Rate Limiting (protect from overload)
    → Timeout (don't wait forever)
      → Retry with Backoff (handle transient failures)
        → Circuit Breaker (stop calling broken service)
          → Fallback (return cached/degraded response)

All wrapped in:
  → Bulkhead (isolate this from other endpoints)
  → Health Checks (let K8s manage lifecycle)
  → Idempotency (safe to retry)
```

**Example: Checkout service calling Payment service**:
```
1. Rate limit: Max 1000 checkout requests/sec
2. Timeout: 3s for payment call
3. Retry: 2 retries with exponential backoff (only if idempotency key set)
4. Circuit breaker: Open after 5 failures in 60s, half-open after 30s
5. Fallback: Queue payment for async processing if circuit is open
6. Bulkhead: Dedicated connection pool for payment service (separate from other deps)
7. Idempotency: Payment idempotency key prevents double-charge on retry
```

---

## Interview Tips

When asked about reliability patterns:

1. **Start with the problem**: "The main risks in distributed systems are partial failures, cascading failures, and resource exhaustion"
2. **Layer the patterns**: "We use timeouts as the first defense, retries for transient failures, circuit breakers to stop cascading, and fallbacks for user experience"
3. **Mention trade-offs**: "Every pattern has a cost — retries can amplify load, circuit breakers can cause availability dips, caching can serve stale data"
4. **Give a real scenario**: "In our checkout flow, the payment service timeout is 3s based on P99 latency, with 2 retries using idempotency keys, and a circuit breaker that trips after 5 consecutive failures"
5. **Reference Istio/Envoy**: "Many of these patterns are built into the service mesh, so we configure them declaratively rather than implementing in every service"
