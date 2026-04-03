# On-Call Best Practices for SRE Teams

On-call is the frontline of reliability. Done well, it protects users and gives engineers confidence. Done poorly, it burns people out and degrades service quality. This guide covers how to build a sustainable, effective on-call program.

---

## On-Call Rotation Design

### Rotation Structures

| Structure | Pros | Cons | Best For |
|-----------|------|------|----------|
| **Weekly rotation** | Simple, predictable | Long shifts cause fatigue | Small teams (3-5 people) |
| **3-4 day rotation** | Less fatigue per shift | More handoffs | Medium teams (5-8 people) |
| **Follow-the-sun** | No night pages | Requires global team | Large distributed teams |
| **Primary + Secondary** | Backup available | Requires 2x coverage | Critical services |

### Minimum Team Size

- **Minimum**: 5 engineers in the rotation (to avoid more than one week in five on-call)
- **Ideal**: 8+ engineers (one week in eight, with room for vacations and illness)
- **Anti-pattern**: 2-3 person rotation leads to burnout within months

### Shift Coverage

```
Primary On-Call: Receives all pages. Expected to respond within 5 minutes.
Secondary On-Call: Escalation target if primary doesn't ACK within 10 minutes.
Escalation Manager: Engineering manager or team lead for P0 incidents.
```

---

## Alert Hygiene

### The Golden Rules

1. **Every alert must be actionable**: If you can't do anything about it, it shouldn't page you
2. **Every alert must require human judgment**: If a script can handle it, automate it
3. **Every alert must be urgent**: If it can wait until morning, it's not a page — it's a ticket

### Alert Tiers

| Tier | Channel | Response Time | Example |
|------|---------|---------------|---------|
| **Page** | Phone call / push notification | < 5 min | SLO burn rate > 14x |
| **Notify** | Slack / email | < 1 hour | Disk usage > 80% |
| **Log** | Dashboard only | Next business day | Non-critical deprecation warning |

### Reducing Alert Fatigue

| Problem | Solution |
|---------|----------|
| Too many alerts per shift | Target < 2 pages per on-call shift |
| Same alert repeating | Fix root cause or tune threshold |
| Alerts during deploys | Add deploy-aware silencing |
| Low-signal alerts | Review and delete alerts that don't lead to action |
| Noisy dashboards | Separate operational alerts from informational metrics |

### Weekly Alert Review

Every week, review the previous week's alerts:

```
For each alert that fired:
1. Was it actionable? (If not → delete or convert to ticket)
2. Was it urgent? (If not → lower severity)
3. Was it a duplicate? (If yes → deduplicate)
4. Did the responder know what to do? (If not → improve runbook)
5. Could it have been automated? (If yes → file automation ticket)
```

---

## Handoff Protocol

### End-of-Shift Handoff Checklist

```
Outgoing on-call should communicate:
[ ] Any ongoing issues or incidents
[ ] Alerts that fired and their resolution status
[ ] Pending changes (deployments, migrations, config changes)
[ ] Known noisy alerts to expect
[ ] Any follow-up tasks from incidents
```

### Handoff Formats

| Method | When to Use |
|--------|-------------|
| **Async (written)** | Normal weeks with no incidents. Post in #oncall Slack channel |
| **Sync (quick call)** | After a busy week or if incidents are ongoing |
| **Detailed doc** | After major incidents or infrastructure changes |

---

## Runbook Standards

Every service should have a runbook covering common failure scenarios.

### Runbook Template

```markdown
# [Service Name] — Runbook

## Service Overview
- What it does: [1-2 sentences]
- Team owner: [team name]
- Slack channel: [#channel]
- Dashboard: [Grafana link]
- Logs: [Loki/ELK query link]

## Common Alerts

### Alert: HighErrorRate
- **What it means**: Error rate exceeds SLO burn threshold
- **Impact**: Users seeing 5xx errors on [endpoint]
- **Steps**:
  1. Check recent deployments: `kubectl rollout history deploy/[name]`
  2. Check error logs: [link to log query]
  3. Check dependency health: [link to dependency dashboard]
  4. If caused by recent deploy → rollback: `kubectl rollout undo deploy/[name]`
  5. If dependency issue → check circuit breaker status
- **Escalation**: If not resolved in 15 min → page [secondary team]

### Alert: HighLatency
- **What it means**: P95 latency exceeding threshold
- **Steps**:
  1. Check saturation (CPU/memory): [dashboard link]
  2. Check database query latency: [dashboard link]
  3. If saturated → scale up: `kubectl scale deploy/[name] --replicas=N`
  4. If DB slow → check for long-running queries
```

### Runbook Quality Criteria

- Any engineer on the team (not just the author) can follow it
- Links to dashboards and queries are up-to-date
- Steps are concrete commands, not vague instructions
- Escalation path is clear
- Last-updated date is recent (review quarterly)

---

## Compensation & Sustainability

### On-Call Compensation Models

| Model | How It Works |
|-------|-------------|
| **Flat stipend** | Fixed amount per on-call shift (e.g., $500/week) |
| **Per-page bonus** | Additional payment per page received |
| **Time-off-in-lieu** | Comp day after on-call week |
| **Combined** | Stipend + comp time + per-page bonus |

### Preventing Burnout

| Practice | Why It Matters |
|----------|---------------|
| **Max 25% on-call time** | Google SRE rule: no more than 25% of time on ops work |
| **No back-to-back shifts** | At least 1 week gap between on-call rotations |
| **Protected focus time** | Day after on-call shift should be meeting-light |
| **Incident review, not blame** | Blameless culture reduces stress |
| **Invest in automation** | Reduce toil = fewer pages = happier engineers |
| **Track pages per shift** | If trending up, the system is getting worse — fix it |

---

## On-Call Metrics to Track

| Metric | Target | Why |
|--------|--------|-----|
| **Pages per shift** | < 2 | More than 2 indicates alert hygiene problems |
| **MTTA (Acknowledge)** | < 5 min | Measures rotation responsiveness |
| **MTTR** | Decreasing trend | Shows process improvement |
| **False positive rate** | < 10% | Non-actionable alerts waste time |
| **Escalation rate** | < 20% | High rate means team needs training or better runbooks |
| **After-hours pages** | Minimized | Track and work to eliminate non-urgent night pages |

---

## Tools

| Category | Tools |
|----------|-------|
| **Paging** | PagerDuty, Opsgenie, Grafana OnCall |
| **Scheduling** | PagerDuty schedules, Opsgenie rotations |
| **Communication** | Slack (incident channels), Zoom (war rooms) |
| **Status Pages** | Statuspage.io, Cachet, Instatus |
| **Runbooks** | Notion, Confluence, GitHub Wiki, Backstage |

---

## Interview Tips

When asked about on-call:

1. **Mention sustainability**: "We target fewer than 2 pages per shift and review alert quality weekly"
2. **Show empathy**: "On-call should be a shared responsibility, not a punishment"
3. **Reference Google SRE**: "We follow the 50% rule — no more than 50% of engineering time on ops, ideally 25%"
4. **Talk about improvement**: "Every page that doesn't result in a meaningful action is a bug in our alerting system"
5. **Describe your handoff process**: Shows operational maturity
