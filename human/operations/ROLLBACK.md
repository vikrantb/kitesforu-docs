# Rollback Procedures

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Cloud Run maintains revision history, enabling quick rollbacks without rebuilding. Each deployment creates a new revision that can be rolled back to.

## Quick Rollback Commands

### Immediate Rollback

```bash
# 1. List recent revisions
gcloud run revisions list --service=SERVICE_NAME --region=us-central1 --limit=5

# 2. Route all traffic to previous revision
gcloud run services update-traffic SERVICE_NAME \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION=100
```

### Service-Specific Commands

```bash
# API Rollback
gcloud run revisions list --service=kitesforu-api --region=us-central1 --limit=5
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=kitesforu-api-XXXXXX=100

# Frontend Rollback
gcloud run revisions list --service=kitesforu-frontend --region=us-central1 --limit=5
gcloud run services update-traffic kitesforu-frontend \
  --region=us-central1 \
  --to-revisions=kitesforu-frontend-XXXXXX=100

# Worker Rollback (example: initiator)
gcloud run revisions list --service=kitesforu-worker-initiator --region=us-central1 --limit=5
gcloud run services update-traffic kitesforu-worker-initiator \
  --region=us-central1 \
  --to-revisions=kitesforu-worker-initiator-XXXXXX=100
```

## Rollback Scenarios

### Scenario 1: API Breaking Change

**Symptoms**:
- 5xx errors from API
- Frontend showing errors
- Jobs not being created

**Actions**:
1. Check API logs for errors
2. Identify last working revision
3. Rollback API service
4. Verify functionality
5. Investigate and fix code

```bash
# Check logs
gcloud run services logs read kitesforu-api --region=us-central1 --limit=100

# Get revisions
gcloud run revisions list --service=kitesforu-api --region=us-central1

# Rollback
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=kitesforu-api-00042-xyz=100
```

### Scenario 2: Worker Pipeline Failure

**Symptoms**:
- Jobs stuck in "running" status
- Dead letter queue filling up
- Timeout errors in logs

**Actions**:
1. Identify failing worker
2. Check worker logs
3. Rollback specific worker
4. Process stuck jobs

```bash
# Check dead letter queue
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=10 --auto-ack

# Rollback worker
gcloud run services update-traffic kitesforu-worker-script \
  --region=us-central1 \
  --to-revisions=kitesforu-worker-script-00015-abc=100
```

### Scenario 3: Frontend Display Issues

**Symptoms**:
- UI not rendering correctly
- Console errors in browser
- Broken user flows

**Actions**:
1. Verify API is healthy
2. Check frontend logs
3. Rollback frontend

```bash
# Check frontend
curl -I https://kitesforu-frontend-m6zqve5yda-uc.a.run.app/

# Rollback
gcloud run services update-traffic kitesforu-frontend \
  --region=us-central1 \
  --to-revisions=kitesforu-frontend-00008-def=100
```

## Gradual Rollback

For less critical issues, use gradual traffic shifting:

```bash
# Split traffic 50/50 between old and new
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=OLD_REVISION=50,NEW_REVISION=50

# Shift more traffic to old if needed
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=OLD_REVISION=90,NEW_REVISION=10

# Complete rollback
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=OLD_REVISION=100
```

## Rollback Checklist

### Before Rollback

- [ ] Confirm the issue is deployment-related
- [ ] Identify the last known good revision
- [ ] Notify team of rollback

### During Rollback

- [ ] Execute rollback command
- [ ] Verify revision traffic routing
- [ ] Check service health

### After Rollback

- [ ] Confirm issue is resolved
- [ ] Test critical user flows
- [ ] Document incident
- [ ] Plan fix for failed deployment

## Revision Management

### List All Revisions

```bash
gcloud run revisions list --service=kitesforu-api --region=us-central1
```

### Describe a Revision

```bash
gcloud run revisions describe REVISION_NAME --region=us-central1
```

### Delete Old Revisions

Clean up old revisions (keep recent 5):

```bash
# List revisions
gcloud run revisions list --service=kitesforu-api --region=us-central1 --format="value(name)"

# Delete specific revision
gcloud run revisions delete REVISION_NAME --region=us-central1 --quiet
```

## Emergency Contacts

For critical production issues:
1. Check #kitesforu-alerts Slack channel
2. Review recent deployments in GitHub Actions
3. Check GCP Console for service health

## Related Documentation

- [DEPLOYMENT.md](./DEPLOYMENT.md) - Deployment procedures
- [INCIDENT_RESPONSE.md](./INCIDENT_RESPONSE.md) - Incident handling
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues
