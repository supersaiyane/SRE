# SRE Interview Mega Guide

A comprehensive collection of scenario-based interview questions organized by topic. Each question includes what interviewers are looking for, a structured answer framework, and scoring criteria.

---

## How to Use This Guide

1. **Practice out loud** — say your answer, don't just read it
2. **Use the STAR format** for behavioral questions: Situation, Task, Action, Result
3. **Whiteboard your architecture** — draw diagrams even if not asked
4. **Ask clarifying questions** — interviewers want to see your thinking process
5. **Connect everything to SLOs** — this is the thread that ties SRE together

---

## Section 1: SLOs, SLIs & Error Budgets

### Q1: "How would you implement SLOs for a payment service?"

**What they're testing**: Can you go from business requirements to measurable objectives?

**Framework**:
```
1. Identify the user journey (payment checkout)
2. Define SLIs (availability, latency)
3. Set targets with data (historical performance + business needs)
4. Calculate error budget
5. Define error budget policy
6. Implement monitoring and alerting
```

**Strong answer includes**:
- Specific SLI formulas with PromQL
- Realistic targets with rationale ("99.95% because payment is revenue-critical, but 99.99% would be prohibitively expensive")
- Burn-rate alerts at multiple windows
- Error budget policy with concrete actions at each threshold
- Distinction between SLO (internal) and SLA (external, with penalties)

**Red flags**: Setting 99.999% "because we want high reliability," no error budget policy, no burn-rate alerts

---

### Q2: "Your error budget is 80% consumed with 2 weeks left in the window. What do you do?"

**What they're testing**: Operational judgment and ability to balance reliability vs. velocity.

**Strong answer**:
```
Immediate (today):
- Review what consumed the budget (incident? gradual degradation?)
- Check current burn rate — is it accelerating or stabilizing?

If burn rate is high (> 2x):
- Freeze non-critical deploys
- Assign engineer to investigate top error sources
- Increase monitoring granularity

If burn rate is normal (< 1x):
- Budget should recover naturally
- Continue with caution, review each deploy
- Notify stakeholders of budget status

Prevention:
- Add canary analysis to deployments
- Review if SLO target is realistic
- Improve test coverage for error-prone paths
```

---

### Q3: "Explain the difference between fast burn and slow burn alerts."

**What they're testing**: Understanding of multi-window burn-rate alerting.

**Answer**:
```
Fast burn (14x/5m):
- Catches sudden spikes (deploy breaks everything)
- Alerts within minutes
- Action: Page on-call, likely rollback
- If sustained, budget exhausts in ~2 days

Slow burn (6x/1h):
- Catches gradual degradation (memory leak, growing latency)
- Alerts within an hour
- Action: Create ticket, investigate within shift
- If sustained, budget exhausts in ~5 days

Why both:
- Fast burn alone misses gradual problems
- Slow burn alone reacts too late to sudden outages
- Together they cover the full spectrum
```

---

## Section 2: Incident Management

### Q4: "Walk me through how you'd handle a P0 incident."

**What they're testing**: Structured incident response, clear communication, decision-making under pressure.

**Framework** (hit all of these):
```
1. Acknowledge (< 5 min)
   - ACK the page, assess initial severity

2. Organize (< 10 min)
   - Declare incident, assign IC role
   - Open war room (Slack channel + Zoom)
   - Post initial status update

3. Triage (10-20 min)
   - What changed? Check recent deploys, config changes
   - What's the blast radius? One endpoint, one region, all users?
   - Check golden signals: which one is anomalous?

4. Mitigate (ASAP)
   - Can we rollback? (fastest option)
   - Can we feature-flag it off?
   - Can we shift traffic away?

5. Resolve
   - Verify fix: error rates normal, latency normal
   - Run smoke tests
   - Update status page

6. Follow up
   - Blameless post-mortem within 48 hours
   - Action items assigned and tracked
```

**Bonus points**: Mention that mitigation speed trumps perfection, reference specific tools (PagerDuty, Grafana), emphasize blameless culture.

---

### Q5: "Tell me about a time you were on-call and something went wrong."

**What they're testing**: Real experience, composure, learning from incidents.

**STAR Format**:
```
Situation: "At [company], our checkout service started returning 5xx errors
            at 2 AM on a Saturday. Error rate hit 15% within 5 minutes."

Task:       "As primary on-call, I needed to restore service before significant
            revenue loss. Our SLO was 99.95% and this was burning budget at 150x."

Action:     "I acknowledged the page within 2 minutes. Checked recent deploys —
            a config change had been pushed at 1:55 AM that changed the database
            connection string for a new replica. I rolled back the config change
            using our GitOps pipeline, which took 4 minutes to propagate."

Result:     "Service was fully restored by 2:12 AM. Total impact: 12 minutes,
            ~400 failed requests. We conducted a post-mortem and added a
            validation step to prevent invalid connection strings in config
            changes. The same class of error hasn't recurred."
```

---

### Q6: "How do you write a good post-mortem?"

**Key points to hit**:
- Blameless — focus on systems, not people
- Timeline with timestamps
- 5 Whys for root cause
- Contributing factors (what made it worse)
- What went well (reinforce good behavior)
- Concrete action items with owners and deadlines
- Shared widely (team meeting, engineering-wide channel)
- Tracked for completion (80%+ action item completion rate)

---

## Section 3: System Design for Reliability

### Q7: "Design a highly available system for a global e-commerce platform with 99.99% availability."

**What they're testing**: Architecture skills, trade-off awareness, multi-region thinking.

**Structure your answer**:

```
1. Clarify requirements (5 min)
   - 99.99% = 4.3 minutes downtime/month
   - "Global" = multiple regions
   - Traffic estimate? (ask the interviewer)

2. Architecture (10 min)
   - Multi-region: active-active in us-east-1 + eu-west-1
   - DNS: Route53 with latency-based routing + health checks
   - Compute: EKS per region, 3 AZs each
   - Database: Aurora Global Database (write in primary, read everywhere)
   - Cache: Regional Redis clusters with cache-aside pattern
   - CDN: CloudFront for static assets

3. Reliability patterns (5 min)
   - Circuit breakers between services
   - Retries with exponential backoff
   - Rate limiting at API gateway
   - Graceful degradation (serve cached catalog if DB is down)

4. Observability (5 min)
   - Per-region golden signal dashboards
   - Cross-region burn-rate alerts
   - Thanos for global metrics view
   - Distributed tracing with correlation IDs

5. Failure scenarios (5 min)
   - Region failure: Route53 failover, promote DB replica
   - AZ failure: K8s spreads pods across AZs, ALB reroutes
   - DB failure: Aurora auto-failover (< 30s)
   - Dependency failure: Circuit breaker + cached fallback

6. Testing (2 min)
   - Quarterly GameDay: simulate region failure
   - Monthly chaos tests: pod kill, network latency
   - Weekly load tests: validate capacity
```

---

### Q8: "How would you handle a cascading failure?"

**What they're testing**: Understanding of failure modes in distributed systems.

**What cascading failure looks like**:
```
1. Database gets slow (disk I/O spike)
2. Service A waits longer for DB responses → thread pool fills up
3. Service B calls Service A → times out → retries → amplifies load
4. Service A's thread pool is exhausted → starts rejecting all requests
5. Service C also depends on Service A → fails too
6. Users see errors everywhere, even on services unrelated to the DB issue
```

**How to prevent/stop it**:
```
Prevention:
- Timeouts on ALL dependency calls (never wait forever)
- Circuit breakers to stop calling failing services
- Bulkheads to isolate connection pools per dependency
- Rate limiting to prevent retry storms
- Backpressure to slow producers when consumers are overloaded

During a cascade:
1. Identify the root cause (which service started failing first?)
2. Shed load (reject low-priority traffic)
3. Trip circuit breakers manually if not auto-tripping
4. Scale the bottleneck if possible
5. Communicate to dependent teams
```

---

## Section 4: Monitoring & Observability

### Q9: "You see a latency spike on the dashboard. Walk me through your investigation."

**Framework**:
```
1. Scope the problem (1 min)
   - Which service? Which endpoints? Which region?
   - When did it start? Does it correlate with a deploy?

2. Check golden signals (2 min)
   - Traffic: Did request volume spike? (could be load-related)
   - Errors: Is error rate also up? (could be failing fast on some)
   - Saturation: CPU/memory/disk? (resource bottleneck?)

3. Drill into latency (2 min)
   - Which percentile? (P50 vs P99)
   - P50 high = everything is slow (system-wide issue)
   - P99 high but P50 normal = tail latency (specific cases)

4. Follow the trace (5 min)
   - Pick a slow request, find its trace ID
   - Open in Jaeger/Tempo: which span is slow?
   - Is it the service itself, a dependency, or the network?

5. Root cause candidates
   - Recent deploy → rollback
   - Resource saturation → scale up
   - Slow dependency → check dependency's health
   - GC pauses → check heap usage
   - Lock contention → check database locks, mutex waits
   - Cold cache → check cache hit rate
```

---

### Q10: "How do you decide what to alert on vs. what to just dashboard?"

**Answer framework**:
```
Alert on (pages/tickets):
- SLO burn rate thresholds
- Complete service unavailability
- Data integrity issues
- Security events

Dashboard only (no alert):
- Resource utilization (unless approaching limits)
- Individual pod restarts
- Cache hit rates
- Traffic volume (unless anomalous)

The test: "If this fires at 3 AM, would I do something right now?"
- Yes → Page
- "I'd look at it in the morning" → Ticket
- "I'd check it when convenient" → Dashboard only
- "I wouldn't do anything" → Delete the alert
```

---

## Section 5: Kubernetes & Infrastructure

### Q11: "How do you make a Kubernetes deployment production-ready?"

**Checklist to walk through**:
```
Resource Management:
[ ] Resource requests and limits set
[ ] HPA configured for auto-scaling
[ ] PodDisruptionBudget defined

Health & Lifecycle:
[ ] Readiness probe (dependency checks)
[ ] Liveness probe (lightweight, no dependency checks)
[ ] Startup probe (for slow-starting apps)
[ ] Graceful shutdown handler (SIGTERM)
[ ] preStop hook if needed

Reliability:
[ ] Multiple replicas (≥ 3 for production)
[ ] Pod anti-affinity (spread across nodes/AZs)
[ ] PodDisruptionBudget (minAvailable: 2)

Observability:
[ ] /metrics endpoint for Prometheus
[ ] Structured JSON logging
[ ] Trace context propagation

Security:
[ ] Non-root container
[ ] Read-only root filesystem
[ ] Network policies
[ ] Secrets from external secret manager (not K8s secrets directly)

Deployment:
[ ] Rolling update strategy with maxSurge/maxUnavailable
[ ] Or canary via Argo Rollouts with AnalysisTemplate
```

---

### Q12: "A pod keeps getting OOMKilled. How do you troubleshoot?"

**Step-by-step answer**:
```
1. Confirm OOMKilled
   kubectl describe pod <name> | grep -A5 "Last State"
   Look for: Reason: OOMKilled, Exit Code: 137

2. Check current vs. limit
   kubectl top pod <name>
   kubectl get pod <name> -o jsonpath='{.spec.containers[*].resources}'

3. Determine if it's a leak or undersized
   - If memory grows steadily over time → leak
   - If memory jumps immediately on startup → undersized
   - If memory spikes under load → need more headroom

4. For leaks:
   - Profile the application (heap dump, memory profiler)
   - Check for unbounded caches, connection pools, event listeners
   - Check for large payloads being buffered in memory

5. For undersized:
   - Increase memory limit
   - For JVM: set -Xmx to ~75% of container limit
   - For Node.js: set --max-old-space-size

6. Prevention:
   - Set up VPA in recommendation mode
   - Alert when usage > 80% of limit
   - Add memory usage to golden signal dashboard
```

---

## Section 6: Chaos Engineering

### Q13: "How would you introduce chaos engineering to a team that's never done it?"

**Maturity model**:
```
Level 0: Manual testing
- Start with table-top exercises: "What happens if X fails?"
- Review existing runbooks and identify gaps

Level 1: Controlled experiments in staging
- Install LitmusChaos or Chaos Mesh
- Start with pod-kill experiments on non-critical services
- Define steady state and success criteria first
- Observe in Grafana: did alerts fire? Did auto-healing work?

Level 2: Production experiments with guardrails
- Run during business hours with team watching
- Start with read-only or stateless services
- Define abort conditions (SLO breach → auto-stop)
- Limit blast radius (one pod, one AZ, not entire region)

Level 3: Automated chaos in CI/CD
- Add chaos step to deployment pipeline
- Run after canary phase: "Does the new version survive chaos?"
- Weekly automated experiments with reporting

Level 4: GameDays
- Quarterly team exercises simulating multi-service failures
- Practice incident response end-to-end
- Include non-engineering stakeholders (support, communications)
```

---

## Section 7: Behavioral / Culture

### Q14: "How do you balance reliability with feature velocity?"

**Framework**:
```
"This is exactly what error budgets solve."

- When error budget is healthy (> 50%):
  Ship features freely. Reliability is not a concern right now.

- When error budget is depleting (25-50%):
  Continue shipping but add more testing, canary analysis.

- When error budget is low (< 25%):
  Shift focus to reliability. Freeze non-critical releases.

- When error budget is exhausted (0%):
  Full stop on features. Reliability sprint until budget recovers.

"The key insight is that reliability isn't a static goal — it's a budget
that gets spent and replenished. Product and engineering negotiate using
data, not opinions."
```

---

### Q15: "Tell me about a time you improved a system's reliability significantly."

**STAR format with metrics**:
```
Situation: "Our API had 99.7% availability — below our 99.9% SLO.
            We were burning error budget every month."

Task:       "Identify and fix the top reliability issues to get above SLO."

Action:     "I analyzed 3 months of incidents and found 3 root causes:
            1. No circuit breakers → dependency failures cascaded (40% of incidents)
            2. No retry budgets → retry storms during transient failures (30%)
            3. Missing readiness probes → traffic sent to unready pods (20%)

            I implemented:
            - Istio circuit breakers with outlier detection
            - Retry policies with exponential backoff and jitter
            - Proper readiness/liveness probes on all services"

Result:     "Availability improved from 99.7% to 99.96% over 2 months.
            Monthly incidents dropped from 8 to 2. MTTR improved from
            45 min to 12 min. Error budget went from consistently negative
            to consistently positive."
```

---

## Section 8: Scenario-Based Questions

### Q16: "It's Black Friday. Your e-commerce platform needs to handle 10x normal traffic. How do you prepare?"

```
4 weeks before:
- Load test at 10x (identify bottlenecks)
- Pre-scale infrastructure (don't rely only on auto-scaling)
- Review and tune HPA settings (increase max replicas)
- Verify CDN caching for static assets
- Test database connection pool limits
- Review circuit breaker thresholds

1 week before:
- Freeze all non-critical deploys
- Run GameDay at 5x traffic
- Verify runbooks are current
- Brief the on-call team
- Pre-stage rollback artifacts

Day of:
- War room staffed, dashboards open
- Feature flags ready to disable non-critical features
- Additional on-call engineers on standby
- Communication templates pre-drafted for status page
- Load shedding thresholds configured (shed recommendations before checkout)

After:
- Post-mortem on any issues
- Document what worked and what didn't
- Update capacity planning models with actual data
```

---

### Q17: "You join a new team. The service has no SLOs, no alerts, and MTTR is 2+ hours. Where do you start?"

```
Week 1: Observe
- Shadow on-call, attend incident reviews
- Read recent incident tickets/post-mortems
- Identify the top 3 sources of pain

Week 2-3: Quick wins
- Add golden signal dashboards (even basic ones)
- Add a single availability SLO with burn-rate alert
- Write runbooks for the top 3 incident types

Month 2: Foundation
- Implement proper SLIs for critical endpoints
- Set up Alertmanager with PagerDuty integration
- Add readiness/liveness probes if missing
- Establish on-call rotation and handoff process

Month 3: Mature
- Error budget policy agreed with engineering manager
- Canary deployments for production releases
- Monthly reliability review meeting
- First chaos engineering experiment

"I wouldn't try to boil the ocean. The goal is measurable improvement
each sprint, not perfection in month one."
```

---

## Scoring Rubric (What Interviewers Use)

| Criteria | Junior (L3-L4) | Senior (L5) | Staff+ (L6+) |
|----------|----------------|-------------|---------------|
| **Technical depth** | Knows the tools | Knows when and why to use each | Designs the strategy, considers trade-offs |
| **Incident response** | Can follow a runbook | Can lead an incident | Can build the incident management program |
| **SLO thinking** | Understands SLI/SLO | Can implement and iterate on SLOs | Can drive SLO adoption across an organization |
| **System design** | Can explain components | Can design for reliability | Can evaluate architectural trade-offs at scale |
| **Communication** | Clear answers | Structured frameworks | Executive-level communication, stakeholder management |
| **Culture** | Practices blameless | Advocates for blameless culture | Builds the culture, mentors others |

---

## Final Tips

1. **Don't memorize — internalize**: Understand the "why" behind each answer so you can adapt
2. **Draw diagrams**: Even on a phone screen, offer to share a diagram
3. **Quantify everything**: "We reduced MTTR" is weak. "We reduced MTTR from 45 min to 12 min" is strong
4. **Admit what you don't know**: "I haven't worked with X, but here's how I'd approach it based on Y" is much better than making something up
5. **Ask good questions**: "What's the current SLO?" or "How many regions?" shows you think about context
6. **Practice the uncomfortable ones**: If a question makes you nervous, that's the one to practice most
