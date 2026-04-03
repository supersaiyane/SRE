# Disaster Recovery Runbook — Region Failover Promotion

## Purpose

Promote the secondary region (eu-west-1) to active when the primary (us-east-1) is degraded or unavailable. This runbook should be executable by any on-call engineer, not just the person who wrote it.

---

## Prerequisites

| Component | Requirement | How to Verify |
|-----------|------------|---------------|
| Aurora Global DB | Replica running in eu-west-1 | `aws rds describe-db-clusters --region eu-west-1` |
| Helm charts | Available in both clusters | `helm list --kube-context eu-west-1-context` |
| S3 | Cross-Region Replication enabled | `aws s3api get-bucket-replication --bucket my-bucket` |
| Prometheus + Thanos | Metrics visible from both regions | Check Thanos query UI |
| Route53 | DNS health check for `/healthz` | `aws route53 get-health-check --health-check-id <id>` |
| Access | kubectl contexts configured for both regions | `kubectl config get-contexts` |

---

## Decision Criteria: When to Failover

| Signal | Threshold | Source |
|--------|-----------|--------|
| Health check failures | > 3 consecutive failures | Route53 health check |
| Regional error rate | > 5% for 10+ minutes | Prometheus / Thanos |
| AWS status | Region-wide service event | AWS Health Dashboard |
| Burn rate | > 14x for 5 minutes (region-specific) | Alertmanager |

**Who can authorize failover**:
- P0 Incident Commander
- VP of Engineering (or delegate)
- If neither available and conditions met: Senior on-call engineer with written approval in incident channel

---

## Failover Procedure

### Step 0: Declare Incident and Document

```bash
# Post in incident channel
"DECLARING REGIONAL FAILOVER: us-east-1 → eu-west-1
 Reason: [health check failure / regional outage / error rate > 5%]
 IC: [your name]
 Time: $(date -u)"
```

### Step 1: Promote Aurora Replica

```bash
# Promote the read replica to a standalone writable cluster
aws rds promote-read-replica-db-cluster \
  --db-cluster-identifier my-db-replica-eu-west-1 \
  --region eu-west-1
```

**Wait for promotion to complete** (typically 1-5 minutes):

```bash
# Poll until status shows 'available' and role shows 'writer'
watch -n 10 'aws rds describe-db-clusters \
  --db-cluster-identifier my-db-replica-eu-west-1 \
  --region eu-west-1 \
  --query "DBClusters[0].{Status:Status,Role:ReaderEndpoint}" \
  --output table'
```

**Verification**:
```bash
# Connect and verify write access
psql -h my-db-replica-eu-west-1.cluster-xxx.eu-west-1.rds.amazonaws.com \
  -U admin -d mydb -c "SELECT 1;"
```

### Step 2: Redeploy Application with Primary Settings

```bash
# Deploy with primary configuration to eu-west-1
helm upgrade golden-signal golden-signal-chart/ \
  -f values-eu-west-1.yaml \
  --set global.primary=true \
  --set global.dbEndpoint=my-db-replica-eu-west-1.cluster-xxx.eu-west-1.rds.amazonaws.com \
  --kube-context eu-west-1-context
```

**Verify pods are running**:
```bash
kubectl get pods -n golden-signal --context eu-west-1-context
kubectl logs -f deployment/golden-signal -n golden-signal --context eu-west-1-context --tail=50
```

**Run smoke tests**:
```bash
# Health check
curl -f https://eu-west-1.internal.example.com/healthz

# Basic functionality check
curl -f https://eu-west-1.internal.example.com/api/checkout/status
```

### Step 3: Update Route53 DNS

**Option A**: Let Route53 failover routing handle it automatically (if health check-based):

```bash
# Verify failover has occurred
aws route53 test-dns-answer \
  --hosted-zone-id ZONEID \
  --record-name api.example.com \
  --record-type A
```

**Option B**: Manual DNS update (if automatic failover is not configured):

```bash
# Apply change batch to point traffic to eu-west-1
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONEID \
  --change-batch file://dns-failover-to-eu-west-1.json
```

**dns-failover-to-eu-west-1.json**:
```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "EU-WEST-1-ALB-ZONE-ID",
          "DNSName": "eu-west-1-alb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}
```

**Verify DNS propagation**:
```bash
# Check from multiple locations
dig api.example.com +short
nslookup api.example.com 8.8.8.8
```

### Step 4: Sync Monitoring and Alerts

```bash
# Update Prometheus alert labels to watch eu-west-1
# If using Thanos, queries already aggregate — verify data flow

# Verify Alertmanager can reach eu-west-1 targets
curl -s http://alertmanager:9093/api/v1/alerts | jq '.data[] | .labels.region'

# Check Grafana dashboards show eu-west-1 data
# Open: https://grafana.example.com/d/golden-signals?var-region=eu-west-1
```

**Confirm alerting works**:
- Check that Slack integration posts to the correct channels
- Verify PagerDuty service is configured for eu-west-1
- Test with a synthetic alert if time permits

### Step 5: Notify Stakeholders

| Audience | Channel | Message |
|----------|---------|---------|
| Engineering | #inc-YYYY-MM-DD-region-failover | Real-time updates |
| Support | #support-escalation | Customer-facing impact summary |
| Customers | Status page (Statuspage.io) | "We are aware of degraded performance..." |
| Enterprise accounts | Direct email/Slack | Personalized notification |
| Leadership | #leadership-incidents | Summary + ETA |

```
Status page update template:
"We identified an issue affecting [service] in [region].
Traffic has been rerouted to [backup region].
Service is [fully restored / being restored].
Next update in 30 minutes."
```

### Step 6: Post-Failover Verification

```bash
# Run full integration test suite against production
./scripts/integration-tests.sh --env production

# Check error rates are back to normal
# Verify in Grafana: golden signals dashboard should show green

# Monitor for 15 minutes before declaring stable
```

**Stability checklist**:
```
[ ] Error rate < 0.1% for 15 minutes
[ ] Latency P95 within normal range
[ ] No error budget burn-rate alerts firing
[ ] All health checks passing
[ ] No customer-reported issues
[ ] Queue depths normal (if applicable)
```

---

## Rollback Plan (Failback to us-east-1)

**Only initiate failback when us-east-1 is confirmed healthy.**

### Step 1: Verify Primary Region Recovery

```bash
# Check AWS region health
aws health describe-events --region us-east-1

# Test connectivity to primary infrastructure
curl -f https://us-east-1.internal.example.com/healthz
```

### Step 2: Restore Database Replication

```bash
# Re-establish Aurora Global Database
# This requires creating a new cluster in us-east-1 and adding it as a secondary
aws rds create-db-cluster \
  --db-cluster-identifier my-db-primary-us-east-1 \
  --engine aurora-postgresql \
  --source-region eu-west-1 \
  --global-cluster-identifier my-global-cluster
```

### Step 3: Failback Application

```bash
# Redeploy in us-east-1 with primary settings
helm upgrade golden-signal golden-signal-chart/ \
  -f values-us-east-1.yaml \
  --set global.primary=true \
  --kube-context us-east-1-context

# Demote eu-west-1 back to secondary
helm upgrade golden-signal golden-signal-chart/ \
  -f values-eu-west-1.yaml \
  --set global.primary=false \
  --kube-context eu-west-1-context
```

### Step 4: Restore DNS

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONEID \
  --change-batch file://dns-failback-to-us-east-1.json
```

### Step 5: Verify and Close

```
[ ] us-east-1 serving traffic
[ ] eu-west-1 is healthy secondary
[ ] Database replication healthy
[ ] All monitoring pointing to primary region
[ ] Status page updated to "Resolved"
[ ] Incident closed with postmortem scheduled
```

---

## Testing This Runbook

| Frequency | Test Type |
|-----------|-----------|
| Monthly | Table-top walkthrough (team reviews steps without executing) |
| Quarterly | Simulated failover in staging environment |
| Semi-annually | GameDay: actual failover in production during low-traffic window |

After each test, update this runbook with any corrections or improvements.

---

## Contacts & Escalation

| Role | Contact | When to Engage |
|------|---------|---------------|
| On-call SRE | PagerDuty rotation | First responder |
| SRE Manager | @sre-manager | P0 escalation, failover authorization |
| DBA | @dba-team | Database promotion issues |
| Platform team | @platform | Infrastructure/networking issues |
| VP Engineering | @vp-eng | External communications, SLA breach |
