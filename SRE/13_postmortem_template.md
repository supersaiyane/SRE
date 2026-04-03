# Blameless Postmortem — Template & Guide

A postmortem is a structured review of an incident, focused on learning and prevention. The goal is never to assign blame, but to understand what happened and prevent recurrence.

---

## When to Write a Postmortem

| Trigger | Required? |
|---------|-----------|
| P0 incident (critical, user-facing) | Mandatory |
| P1 incident (significant degradation) | Mandatory |
| P2 incident that exposed systemic weakness | Recommended |
| Near-miss that could have been a P0 | Recommended |
| Any incident where MTTR > 1 hour | Recommended |
| On-call engineer requests one | Always |

**Timeline**: Draft within 48 hours of resolution. Review meeting within 5 business days.

---

## Postmortem Template

Copy and use this template for every incident:

---

### Incident Title

**Date**: YYYY-MM-DD

**Duration**: HH:MM (from detection to resolution)

**Severity**: P0 / P1 / P2

**Incident Commander**: [Name]

**Author**: [Name]

**Status**: Draft / In Review / Complete

---

### Summary

_One to three sentences describing what happened and its impact._

Example: "On March 15, the checkout service experienced a 47-minute outage caused by a database connection pool exhaustion following deployment of v2.4.1. Approximately 12,000 users were unable to complete purchases, resulting in an estimated $85,000 in lost revenue."

---

### Impact

| Metric | Value |
|--------|-------|
| **Duration** | 47 minutes |
| **Users affected** | ~12,000 |
| **Requests failed** | 34,500 |
| **Error budget consumed** | 62% of monthly budget |
| **Revenue impact** | ~$85,000 estimated |
| **SLO breached?** | Yes — availability dropped to 98.2% (SLO: 99.9%) |
| **Customer notifications** | Status page updated; 3 enterprise customers contacted |

---

### Timeline

_Chronological record of events. Use UTC timestamps._

| Time (UTC) | Event |
|------------|-------|
| 14:00 | Deploy v2.4.1 to production (automated via ArgoCD) |
| 14:12 | First error budget burn-rate alert fires (14x/5m threshold) |
| 14:14 | On-call engineer acknowledges page |
| 14:16 | IC declares P0 incident, opens #inc-2024-03-15-checkout |
| 14:18 | Grafana shows checkout error rate at 15% |
| 14:22 | Logs show "connection pool exhausted" errors |
| 14:25 | Hypothesis: new ORM library has connection leak |
| 14:28 | Decision: rollback to v2.4.0 |
| 14:30 | `kubectl rollout undo deployment/checkout` executed |
| 14:35 | New pods healthy, error rate dropping |
| 14:45 | Error rate returns to baseline (< 0.1%) |
| 14:50 | Smoke tests pass; status page updated to "Resolved" |
| 14:59 | All-clear posted in incident channel |

---

### Root Cause Analysis

_What was the fundamental cause of the incident?_

**Root cause**: Version 2.4.1 included an upgrade of the database ORM library (TypeORM 0.3.x to 0.4.x). The new version changed connection pooling behavior, creating a new connection per query instead of reusing from the pool. Under production load (~500 req/s), the connection pool was exhausted within 12 minutes.

**5 Whys**:

```
1. Why did checkout fail?
   → Database queries returned "connection refused" errors

2. Why were connections refused?
   → Connection pool (max 100) was fully consumed with no connections returning

3. Why weren't connections returning to the pool?
   → New ORM version creates connections outside the pool for certain query patterns

4. Why wasn't this caught before production?
   → Staging load is 10x lower; pool exhaustion only occurs above ~200 req/s

5. Why don't we load-test ORM changes?
   → Dependency updates don't trigger load tests in our CI pipeline
```

---

### Contributing Factors

_Things that didn't cause the incident but made it worse or delayed resolution._

| Factor | Impact |
|--------|--------|
| No connection pool monitoring dashboard | Added 3 minutes to diagnosis |
| Staging environment too small to reproduce | Issue was invisible in pre-prod |
| ORM changelog didn't mention pooling behavior change | Team wasn't aware of the breaking change |

---

### What Went Well

_Celebrate effective responses — this reinforces good behavior._

- Burn-rate alert fired within 12 minutes (before user-reported tickets)
- On-call acknowledged within 2 minutes
- IC was declared quickly, clear war room established
- Rollback was clean and fast (5 minutes)
- Status page was updated throughout the incident

---

### What Went Poorly

_Be specific about process failures, not people._

- No automated canary analysis for the deploy (would have caught the regression)
- Database connection pool metrics not on the golden signal dashboard
- Staging environment doesn't represent production load
- Dependency updates bypass load testing in CI

---

### Action Items

| # | Action | Owner | Priority | Deadline | Status |
|---|--------|-------|----------|----------|--------|
| 1 | Add DB connection pool metrics to golden signal dashboard | @engineer-a | High | 2024-03-22 | TODO |
| 2 | Add Argo Rollouts canary analysis for checkout deploys | @engineer-b | Critical | 2024-03-29 | TODO |
| 3 | Create load test that runs for dependency-update PRs | @engineer-c | High | 2024-04-05 | TODO |
| 4 | Scale staging environment to 50% of prod traffic capacity | @infra-team | Medium | 2024-04-12 | TODO |
| 5 | Add connection pool size to standard service template | @platform | Medium | 2024-04-19 | TODO |

---

### Lessons Learned

_Key takeaways for the broader team._

1. Dependency updates can change fundamental behavior even in minor version bumps — treat them with the same scrutiny as feature changes
2. Connection pooling is a critical resource that should be monitored with the same priority as CPU and memory
3. Our staging environment is too small to catch performance regressions — we need a load testing step for high-risk changes

---

## End of Template

---

## Running a Postmortem Meeting

### Meeting Structure (45-60 minutes)

```
 5 min — IC presents timeline and impact (no discussion yet)
10 min — Walk through root cause analysis as a group
10 min — Review contributing factors
 5 min — Celebrate what went well
10 min — Discuss what went poorly (facilitate, don't blame)
10 min — Review and assign action items
 5 min — Summarize lessons learned
```

### Ground Rules

1. **No blame**: Focus on systems and processes, not individuals
2. **Assume good intent**: Everyone was trying their best with the information they had
3. **Be specific**: "We should improve monitoring" is not useful. "Add connection pool metric to checkout dashboard" is.
4. **Limit action items**: 5-7 max. More than that and nothing gets done.
5. **Assign owners and deadlines**: Action items without owners are wishes, not plans.

### Facilitator Tips

- If someone starts blaming, redirect: "What about our systems allowed this to happen?"
- If discussion gets stuck, ask: "What would have prevented this entirely?"
- If action items are vague, push back: "Can you make that specific enough that someone could start tomorrow?"
- End by thanking the team for their response and participation

---

## Postmortem Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Skipping postmortems for "small" incidents | Misses systemic patterns | Set clear triggers (above) |
| Writing postmortems months later | Memory fades, details lost | Draft within 48 hours |
| No action items | Learning without action is just discussion | Every postmortem needs 3+ concrete actions |
| Action items that never get done | Erodes trust in the process | Track in sprint backlog, review monthly |
| Blame-driven language | Creates fear, reduces reporting | Review language before publishing |
| Only the IC writes it | Misses perspectives | Collaborate with responders |
| Never sharing externally | Same mistakes repeated across teams | Share in engineering-wide channels |

---

## Tracking Postmortem Effectiveness

| Metric | What to Track |
|--------|---------------|
| **Action item completion rate** | Target: > 80% completed within deadline |
| **Repeat incidents** | Same root cause should never cause a second P0 |
| **Time to draft** | Target: < 48 hours |
| **Time to review meeting** | Target: < 5 business days |
| **Postmortems per quarter** | Track trend — increasing may signal systemic issues |

---

## Interview Tips

When asked about postmortems:

1. **Lead with "blameless"**: "We focus on what the system allowed to happen, not who did what"
2. **Show structure**: Walk through the template sections
3. **Give a real example**: "We had an incident where a dependency update changed connection pooling behavior..."
4. **Mention follow-through**: "Every action item gets a Jira ticket, an owner, and a deadline. We review completion monthly"
5. **Connect to SLOs**: "Postmortems are triggered by SLO impact — if an incident burns more than X% of our error budget, it automatically triggers a review"
