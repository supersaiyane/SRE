# Incident Lifecycle — Stage-by-Stage Deep Dive

This guide walks through the complete incident lifecycle, from the first signal to knowledge-base closure. Each stage includes what to do, who's responsible, and the key terms you need to know.

---

## Overview

```
Monitor → Alert → Acknowledge → Triage → Mitigate → Resolve → Post-Mortem → Action Items → Close
   1        2          3           4         5          6          7              8            9
```

---

## Stage 1: Monitoring & Signal

**Goal**: Continuously watch your systems so you detect problems before users do.

| Activity | Details |
|----------|---------|
| Ingest SLIs | Prometheus scrapes metrics every 15s; track latency, error rate, saturation |
| Dashboards | Grafana panels for golden signals per service |
| Anomaly Detection | Burn-rate alerts catch both sudden spikes and slow degradation |

**Key Metrics**:
- **MTTD (Mean Time to Detect)**: How quickly your monitoring catches an issue. Target: < 5 minutes for P0.
- **White-box monitoring**: Internal metrics from your application (Prometheus, OpenTelemetry).
- **Black-box monitoring**: External probes that test from a user's perspective (synthetic checks, uptime monitors).

**What good looks like**:
```
- Every service exposes /metrics endpoint
- Golden signal dashboards exist for all critical services
- Burn-rate alerts configured at 14x/5m and 6x/1h thresholds
- Synthetic monitors check key user journeys every 60s
```

---

## Stage 2: Alerting & Paging

**Goal**: Get the right person's attention at the right urgency level.

| Activity | Details |
|----------|---------|
| Alert fires | Prometheus evaluates rules → Alertmanager routes to the correct channel |
| On-call paged | PagerDuty/Opsgenie sends page based on severity and escalation policy |
| Escalation | If not acknowledged within 5 min → escalate to secondary on-call |

**Alert Severity Mapping**:

| Severity | Burn Rate | Response Time | Channel |
|----------|-----------|---------------|---------|
| P0 (Critical) | > 14x for 5m | Immediate page | PagerDuty + phone call |
| P1 (High) | > 6x for 1h | 15 min | PagerDuty + Slack |
| P2 (Medium) | > 2x for 6h | 1 hour | Slack + ticket |
| P3 (Low) | Informational | Next business day | Ticket only |

**Anti-patterns to avoid**:
- Alert fatigue: too many non-actionable alerts. Every alert should require human action.
- Missing escalation: always have a backup on-call.
- Alerting on symptoms vs. causes: alert on SLO burn, not individual metric spikes.

---

## Stage 3: Acknowledgement & Incident Declaration

**Goal**: Take ownership and set up communication channels.

| Activity | Details |
|----------|---------|
| ACK the page | First responder acknowledges within MTTA target (< 5 min for P0) |
| Assess severity | Label as P0/P1/P2 based on user impact and blast radius |
| Declare incident | Create incident in your tool (PagerDuty, Jira, Statuspage) |
| War room | Spin up a dedicated Slack channel: `#inc-YYYY-MM-DD-short-desc` |

**Incident Roles**:

| Role | Responsibility |
|------|---------------|
| **Incident Commander (IC)** | Single decision-maker. Coordinates response, delegates tasks, makes rollback calls |
| **Technical Lead** | Hands-on-keyboard person diagnosing and fixing |
| **Communications Lead** | Posts updates to stakeholders, status page, customer support |
| **Scribe** | Documents timeline, actions taken, and decisions in real-time |

**MTTA (Mean Time to Acknowledge)**: Measures how fast your on-call rotation responds. High MTTA signals problems with rotation schedules, alert routing, or notification channels.

---

## Stage 4: Triage & Diagnosis

**Goal**: Understand what's broken, what's affected, and form a hypothesis.

**Triage Checklist**:
```
1. What changed recently? (deployments, config changes, infra updates)
2. What's the blast radius? (one service, one region, all users?)
3. What do the golden signals show? (latency up? errors up? saturation high?)
4. What do the logs say? (grep for errors in the last 30 minutes)
5. What do the traces show? (slow spans, failed dependencies)
6. Is this a known issue? (check runbooks, recent incidents)
```

**Tools to Use**:
| Tool | Purpose |
|------|---------|
| Grafana | Check golden signal dashboards |
| Prometheus | Run ad-hoc PromQL queries |
| Jaeger/Tempo | Trace slow or failing requests |
| Loki/ELK | Search logs for error patterns |
| `kubectl` | Check pod status, events, resource usage |

**Decision Point**: Based on triage, decide between:
- **Rollback**: If a recent deployment caused it, revert immediately
- **Fix Forward**: If the fix is small and well-understood, ship a hotfix
- **Mitigate**: If root cause is unclear, reduce impact first (traffic shift, feature flag, circuit break)

---

## Stage 5: Mitigation / Workaround

**Goal**: Stop the bleeding. Reduce or eliminate user impact as fast as possible.

| Technique | When to Use | Example |
|-----------|-------------|---------|
| **Rollback** | Recent deploy caused it | `kubectl rollout undo deployment/api` |
| **Feature Flag** | New feature is broken | Disable flag in LaunchDarkly/ConfigMap |
| **Traffic Shift** | One region/instance is bad | Route traffic away via Istio VirtualService |
| **Circuit Breaker** | Downstream dependency failing | Trip circuit to return cached/default response |
| **Scale Up** | Saturation-driven issue | Increase replicas or node pool |
| **Hotfix** | Root cause is clear and fix is small | Emergency PR → fast-track deploy |

**Key Principle**: Mitigation is about **speed**, not perfection. A 90% fix in 5 minutes beats a 100% fix in 2 hours.

---

## Stage 6: Resolution

**Goal**: Service is fully healthy. All users are back to normal.

**Resolution Checklist**:
```
[ ] Error rate is back within SLO
[ ] Latency percentiles are nominal
[ ] No more error log entries related to the incident
[ ] Smoke tests pass (automated or manual)
[ ] Status page updated to "Resolved"
[ ] Monitoring confirms stability for at least 15 minutes
[ ] All-clear posted in incident channel
```

**MTTR (Mean Time to Resolve)**: Measured from detection to full resolution. This is the metric leadership cares about most. Track it per severity level.

---

## Stage 7: Blameless Post-Mortem

**Goal**: Learn from the incident. Prevent it from happening again.

**When to Write One**:
- All P0 and P1 incidents (mandatory)
- Any P2 that exposed a systemic weakness
- Any incident where mitigation took longer than expected

**Post-Mortem Structure**:
```
1. Summary (1-2 sentences: what happened and impact)
2. Timeline (minute-by-minute from detection to resolution)
3. Impact (users affected, duration, SLO burn, revenue impact)
4. Root Cause Analysis (use 5 Whys or Fishbone diagram)
5. Contributing Factors (what made it worse)
6. What Went Well (celebrate good responses)
7. What Went Poorly (identify process gaps)
8. Action Items (specific, assigned, with deadlines)
```

**5 Whys Example**:
```
Why did checkout fail?       → API returned 500s
Why did API return 500s?     → Database connection pool exhausted
Why was pool exhausted?      → Connection leak in new ORM version
Why wasn't leak caught?      → No integration test for connection pooling
Why no integration test?     → Testing gap in deployment pipeline
→ Action: Add connection pool integration test to CI
```

---

## Stage 8: Action Items & Follow-Up

**Goal**: Turn lessons into concrete improvements.

**Action Item Categories**:

| Category | Examples | Priority |
|----------|----------|----------|
| **Detect faster** | Add missing alert, reduce scrape interval | High |
| **Prevent recurrence** | Fix root cause, add validation | Critical |
| **Reduce blast radius** | Add circuit breakers, improve isolation | Medium |
| **Automate response** | Auto-rollback on burn rate, auto-scale | Medium |
| **Improve process** | Update runbook, improve handoff | Low |

**Rules for Good Action Items**:
- Each item has a single owner
- Each item has a deadline
- Items are tracked in your project management tool (Jira, Linear)
- Review progress in weekly SRE team meeting
- Don't create more than 5-7 action items per incident (focus on highest impact)

---

## Stage 9: Knowledge-Base Update & Closure

**Goal**: Make sure the next person who faces a similar issue has a head start.

| Activity | Details |
|----------|---------|
| Update runbooks | Add or refine the runbook for this failure mode |
| Tag metrics | Add annotation in Grafana marking the incident window |
| Update dashboards | Add any new panels that were useful during diagnosis |
| Close incident | Mark resolved in incident management tool |
| Share learnings | Present at team meeting or SRE guild |

**MTBF (Mean Time Between Failures)**: Track this for recurring issues. If the same type of incident keeps happening, the action items from post-mortems aren't effective enough.

---

## Quick Reference Card

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| **MTTD** | Detection speed | < 5 min (P0) |
| **MTTA** | Acknowledgement speed | < 5 min (P0) |
| **MTTR** | Full resolution time | < 1 hour (P0) |
| **MTBF** | Time between incidents | Increasing trend |

---

## Interview Tips

When asked about incident management:

1. **Walk through the lifecycle** — show you understand all 9 stages, not just "fix the thing"
2. **Mention roles** — IC, Tech Lead, Comms Lead shows you understand coordination
3. **Reference SLOs** — tie incident severity to error budget impact
4. **Emphasize blameless culture** — interviewers care about this
5. **Give a real example** — "In a previous role, we had a database connection leak that..." carries more weight than theory
