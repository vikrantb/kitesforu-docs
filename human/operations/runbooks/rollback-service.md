# Runbook: Rollback Service

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Rollback any Cloud Run service to a previous revision.

## When to Rollback

- New deployment causes errors
- Performance degradation after deploy
- User-reported issues after deploy
- Failed health checks

## Quick Rollback

### Step 1: Identify Service

```bash
# List all services
gcloud run services list --region=us-central1
```

### Step 2: List Revisions

```bash
# Replace SERVICE_NAME with actual service
gcloud run revisions list --service=SERVICE_NAME --region=us-central1 --limit=5
```

Output example:
```
REVISION                        ACTIVE  SERVICE          DEPLOYED
kitesforu-api-00045-abc         yes     kitesforu-api    2024-01-15 10:30:00
kitesforu-api-00044-def               kitesforu-api    2024-01-14 15:20:00
kitesforu-api-00043-ghi               kitesforu-api    2024-01-13 09:00:00
```

### Step 3: Execute Rollback

```bash
# Route all traffic to previous revision
gcloud run services update-traffic SERVICE_NAME \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION=100
```

### Step 4: Verify

```bash
# Confirm traffic routing
gcloud run services describe SERVICE_NAME --region=us-central1 --format="value(status.traffic)"

# Test service
curl https://SERVICE_URL/health
```

## Service-Specific Rollback Commands

### API

```bash
# List API revisions
gcloud run revisions list --service=kitesforu-api --region=us-central1 --limit=5

# Rollback API
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=kitesforu-api-XXXXX-xxx=100
```

### Frontend

```bash
# List frontend revisions
gcloud run revisions list --service=kitesforu-frontend --region=us-central1 --limit=5

# Rollback frontend
gcloud run services update-traffic kitesforu-frontend \
  --region=us-central1 \
  --to-revisions=kitesforu-frontend-XXXXX-xxx=100
```

### Workers

```bash
# List worker revisions (example: script worker)
gcloud run revisions list --service=kitesforu-worker-script --region=us-central1 --limit=5

# Rollback worker
gcloud run services update-traffic kitesforu-worker-script \
  --region=us-central1 \
  --to-revisions=kitesforu-worker-script-XXXXX-xxx=100
```

## Gradual Rollback

For less critical issues, use traffic splitting:

```bash
# Split 50/50 between old and new
gcloud run services update-traffic SERVICE_NAME \
  --region=us-central1 \
  --to-revisions=OLD_REV=50,NEW_REV=50

# Shift to 90/10 if old is better
gcloud run services update-traffic SERVICE_NAME \
  --region=us-central1 \
  --to-revisions=OLD_REV=90,NEW_REV=10

# Complete rollback
gcloud run services update-traffic SERVICE_NAME \
  --region=us-central1 \
  --to-revisions=OLD_REV=100
```

## Roll Forward

After fixing the issue:

```bash
# Deploy new fixed version
./infra/deploy-api.sh  # or appropriate deploy script

# Or route traffic to latest revision
gcloud run services update-traffic SERVICE_NAME \
  --region=us-central1 \
  --to-latest
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Can't find revision | Revision may have been deleted, check older ones |
| Rollback didn't help | Issue might be in data/config, not code |
| Traffic still going to new | Check traffic split with describe command |

## Post-Rollback Actions

1. **Document**: Note what was rolled back and why
2. **Investigate**: Determine root cause of failed deployment
3. **Fix**: Create fix and test thoroughly
4. **Redeploy**: Deploy fixed version when ready
5. **Monitor**: Watch for issues after redeployment

## Related

- [ROLLBACK.md](../ROLLBACK.md)
- [INCIDENT_RESPONSE.md](../INCIDENT_RESPONSE.md)
