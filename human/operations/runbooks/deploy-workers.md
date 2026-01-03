# Runbook: Deploy Workers

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Deploy all worker services to Cloud Run. Workers share a single Docker image but run as separate services.

## Worker Services

| Service | Pub/Sub Topic | Description |
|---------|---------------|-------------|
| kitesforu-worker-initiator | job-initiate | Initializes job processing |
| kitesforu-worker-research-planner | job-research-planner | Creates research plans |
| kitesforu-worker-tools | job-execute-tools | Executes research tools |
| kitesforu-worker-script | job-script | Generates podcast scripts |
| kitesforu-worker-audio | job-audio | Generates audio files |

## Prerequisites

- [ ] GCP access configured
- [ ] Access to kitesforu-workers repository
- [ ] All tests passing

## Procedure

### Step 1: Verify Current State

```bash
# Check all worker revisions
for worker in kitesforu-worker-initiator kitesforu-worker-research-planner kitesforu-worker-tools kitesforu-worker-script kitesforu-worker-audio; do
  echo "$worker: $(gcloud run services describe $worker --region=us-central1 --format='value(status.latestReadyRevisionName)' 2>/dev/null || echo 'NOT FOUND')"
done
```

### Step 2: Pull Latest Changes

```bash
cd /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-workers
git checkout main
git pull origin main
```

### Step 3: Run Tests

```bash
# Run unit tests
pytest tests/ -v

# Type checking
mypy src/
```

### Step 4: Deploy All Workers

**Option A: Using deploy script (recommended)**
```bash
./infra/deploy-workers.sh
```

**Option B: Deploy single worker**
```bash
# Build image
gcloud builds submit --tag us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/kitesforu-workers:latest

# Deploy specific worker
gcloud run deploy kitesforu-worker-initiator \
  --image us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/kitesforu-workers:latest \
  --region us-central1 \
  --platform managed \
  --set-env-vars="WORKER_TYPE=initiator"
```

### Step 5: Verify Deployment

```bash
# Check all workers deployed
gcloud run services list --region=us-central1 --filter="SERVICE:kitesforu-worker"

# Check logs for each worker
for worker in kitesforu-worker-initiator kitesforu-worker-research-planner kitesforu-worker-tools kitesforu-worker-script kitesforu-worker-audio; do
  echo "=== $worker ==="
  gcloud run services logs read $worker --region=us-central1 --limit=5
done
```

### Step 6: Test Message Processing

```bash
# Create a test job via API
curl -X POST https://kitesforu-api-m6zqve5yda-uc.a.run.app/api/v1/jobs \
  -H "Authorization: Bearer $TEST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"topic": "Test topic", "duration_min": 1}'

# Monitor job progression
watch -n 5 'gcloud firestore documents describe --database="(default)" --collection=podcast_jobs --document=JOB_ID'
```

### Step 7: Check Pub/Sub Health

```bash
# Check for unacked messages
for sub in job-initiate-sub job-research-planner-sub job-execute-tools-sub job-script-sub job-audio-sub; do
  echo "$sub:"
  gcloud pubsub subscriptions describe $sub --format="value(pushConfig.pushEndpoint)"
done

# Check dead letter queue
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=5 --auto-ack=false
```

## Rollback

### Rollback Single Worker

```bash
# List revisions
gcloud run revisions list --service=kitesforu-worker-script --region=us-central1 --limit=5

# Rollback
gcloud run services update-traffic kitesforu-worker-script \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION=100
```

### Rollback All Workers

```bash
# For each worker, rollback to previous revision
for worker in kitesforu-worker-initiator kitesforu-worker-research-planner kitesforu-worker-tools kitesforu-worker-script kitesforu-worker-audio; do
  PREV_REV=$(gcloud run revisions list --service=$worker --region=us-central1 --limit=2 --format="value(name)" | tail -1)
  echo "Rolling back $worker to $PREV_REV"
  gcloud run services update-traffic $worker --region=us-central1 --to-revisions=$PREV_REV=100
done
```

## Troubleshooting

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| Workers not receiving messages | Check Pub/Sub push config | Verify push endpoint URLs |
| Jobs stuck in queue | Check initiator logs | Restart initiator or check auth |
| Timeout errors | Check worker timeout settings | Increase Cloud Run timeout |
| Dead letters accumulating | Check error messages | Fix worker code, reprocess messages |

## Related

- [DEPLOYMENT.md](../DEPLOYMENT.md)
- [diagnose-job.md](./diagnose-job.md)
- [clear-dead-letters.md](./clear-dead-letters.md)
