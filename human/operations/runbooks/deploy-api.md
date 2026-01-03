# Runbook: Deploy API

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Deploy the kitesforu-api service to Cloud Run.

## Prerequisites

- [ ] GCP access configured (`gcloud auth login`)
- [ ] Access to kitesforu-api repository
- [ ] All tests passing

## Procedure

### Step 1: Verify Current State

```bash
# Check current revision
gcloud run services describe kitesforu-api --region=us-central1 --format="value(status.latestReadyRevisionName)"

# Check service health
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/health
```

### Step 2: Pull Latest Changes

```bash
cd /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-api
git checkout main
git pull origin main
```

### Step 3: Run Tests Locally

```bash
# Run unit tests
pytest tests/ -v

# Run type checking
mypy src/
```

### Step 4: Deploy

**Option A: Using deploy script (recommended)**
```bash
./infra/deploy-api.sh
```

**Option B: Manual deployment**
```bash
# Build image
gcloud builds submit --tag us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/kitesforu-api:latest

# Deploy to Cloud Run
gcloud run deploy kitesforu-api \
  --image us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/kitesforu-api:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated
```

### Step 5: Verify Deployment

```bash
# Check new revision
gcloud run services describe kitesforu-api --region=us-central1 --format="value(status.latestReadyRevisionName)"

# Health check
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/health

# Test an endpoint
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/api/v1/health
```

### Step 6: Monitor

```bash
# Watch logs for errors
gcloud run services logs read kitesforu-api --region=us-central1 --limit=50

# Check error rate in Cloud Console
# https://console.cloud.google.com/run/detail/us-central1/kitesforu-api/metrics
```

## Rollback

If issues detected:

```bash
# List revisions
gcloud run revisions list --service=kitesforu-api --region=us-central1 --limit=5

# Rollback to previous
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION_NAME=100
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Build fails | Check Dockerfile, dependencies |
| Deploy fails | Check IAM permissions, secrets |
| Health check fails | Check logs for startup errors |
| 5xx errors | Check logs, consider rollback |

## Related

- [DEPLOYMENT.md](../DEPLOYMENT.md)
- [rollback-service.md](./rollback-service.md)
