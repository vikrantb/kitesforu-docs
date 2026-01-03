# Incident Response Playbook

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Severity Levels

| Level | Description | Response Time | Examples |
|-------|-------------|---------------|----------|
| **SEV1** | Complete service outage | Immediate | All jobs failing, API down |
| **SEV2** | Major functionality broken | < 30 min | One worker failing, slow response |
| **SEV3** | Minor issues | < 4 hours | Cosmetic bugs, non-critical errors |
| **SEV4** | Improvements | Next sprint | Performance optimization, tech debt |

## Incident Response Process

### 1. Detection

**Automated**:
- Cloud Monitoring alerts
- Error rate thresholds
- Health check failures

**Manual**:
- User reports
- Team observation
- Log analysis

### 2. Triage

Quick assessment questions:
1. What is broken?
2. How many users affected?
3. Is data at risk?
4. When did it start?
5. Any recent deployments?

### 3. Communication

**SEV1/SEV2**:
1. Post in #kitesforu-incidents
2. Tag on-call engineer
3. Update status page (if public)

**Template**:
```
🚨 INCIDENT: [Brief Description]
Severity: SEV[1/2/3]
Impact: [Users/Jobs affected]
Status: Investigating
Lead: [@name]
```

### 4. Investigation

```bash
# Quick health check
gcloud run services list --region=us-central1

# Recent errors
gcloud logging read "severity>=ERROR" --limit=50 --format="table(timestamp,resource.labels.service_name,textPayload)"

# Recent deployments
gcloud run revisions list --service=kitesforu-api --region=us-central1 --limit=5
```

### 5. Mitigation

**Options by priority**:
1. **Rollback** - If recent deployment caused issue
2. **Scale** - If capacity issue
3. **Restart** - If transient failure
4. **Disable** - If feature is broken
5. **Fix forward** - If quick fix available

### 6. Resolution

- Verify fix is working
- Monitor for recurrence
- Update incident channel
- Clear any alerts

### 7. Post-Incident

- Write post-mortem (SEV1/SEV2)
- Update runbooks if needed
- Create follow-up tickets
- Share learnings

## Incident Scenarios

### Scenario: Complete API Outage

**Detection**: 5xx errors > 90%

**Steps**:
1. Check API health:
   ```bash
   curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/health
   ```

2. Check recent deployments:
   ```bash
   gcloud run revisions list --service=kitesforu-api --region=us-central1 --limit=5
   ```

3. Check logs:
   ```bash
   gcloud run services logs read kitesforu-api --region=us-central1 --limit=100
   ```

4. If deployment-related, rollback:
   ```bash
   gcloud run services update-traffic kitesforu-api \
     --region=us-central1 \
     --to-revisions=PREVIOUS_REVISION=100
   ```

5. If not deployment-related:
   - Check external dependencies (Clerk, Stripe, Firestore)
   - Check for GCP outage
   - Scale up if needed

### Scenario: Jobs Not Processing

**Detection**: Jobs stuck in queued/running status

**Steps**:
1. Check Pub/Sub:
   ```bash
   gcloud pubsub subscriptions describe job-initiate-sub
   ```

2. Check dead letter queue:
   ```bash
   gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=10
   ```

3. Check worker status:
   ```bash
   gcloud run services describe kitesforu-worker-initiator --region=us-central1
   ```

4. Check worker logs:
   ```bash
   gcloud run services logs read kitesforu-worker-initiator --region=us-central1 --limit=50
   ```

5. Mitigate:
   - Rollback worker if recent deployment
   - Clear dead letter messages
   - Retry failed jobs

### Scenario: External API Failure (OpenAI/Anthropic)

**Detection**: Research or script generation failing

**Steps**:
1. Check provider status pages
2. Review error logs:
   ```bash
   gcloud logging read "jsonPayload.provider=openai AND severity=ERROR" --limit=20
   ```

3. Check model router health:
   ```bash
   gcloud firestore documents list \
     --database="(default)" \
     --collection=model_provider_health \
     --limit=10
   ```

4. Mitigate:
   - Model router should auto-fallback
   - If all providers down, pause job creation
   - Communicate to users if extended outage

### Scenario: Database Issues

**Detection**: Firestore errors in logs

**Steps**:
1. Check GCP status for Firestore
2. Check connection from service:
   ```bash
   gcloud logging read "resource.type=cloud_run_revision AND textPayload:firestore" --limit=20
   ```

3. Verify IAM permissions:
   ```bash
   gcloud projects get-iam-policy kitesforu-dev --filter="bindings.members:serviceAccount:kitesforu-worker"
   ```

4. Mitigate:
   - If GCP issue, wait for resolution
   - If permissions, fix IAM
   - If code issue, rollback or fix

## Runbook Quick Links

| Issue | Runbook |
|-------|---------|
| Deploy API | [deploy-api.md](./runbooks/deploy-api.md) |
| Deploy Workers | [deploy-workers.md](./runbooks/deploy-workers.md) |
| Rollback Service | [rollback-service.md](./runbooks/rollback-service.md) |
| Diagnose Job | [diagnose-job.md](./runbooks/diagnose-job.md) |
| Clear Dead Letters | [clear-dead-letters.md](./runbooks/clear-dead-letters.md) |

## Post-Incident Template

```markdown
# Post-Incident Report: [Title]

## Summary
- **Date**: YYYY-MM-DD
- **Duration**: X hours Y minutes
- **Severity**: SEV[1/2/3]
- **Impact**: [Users/jobs affected]

## Timeline
- HH:MM - Issue detected
- HH:MM - Investigation started
- HH:MM - Root cause identified
- HH:MM - Fix deployed
- HH:MM - Issue resolved

## Root Cause
[Technical description of what went wrong]

## Resolution
[What was done to fix it]

## Lessons Learned
- What went well
- What could be improved

## Action Items
- [ ] Action 1 - @owner - Due date
- [ ] Action 2 - @owner - Due date
```

## Emergency Contacts

| Role | Contact |
|------|---------|
| On-Call | Check #kitesforu-oncall |
| GCP Support | Cloud Console > Support |
| Clerk Support | dashboard.clerk.com |
| Stripe Support | dashboard.stripe.com |
