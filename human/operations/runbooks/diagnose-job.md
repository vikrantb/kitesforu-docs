# Runbook: Diagnose Job

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Debug a job that is stuck, failed, or behaving unexpectedly.

## Prerequisites

- Job ID from user or logs
- GCP access configured

## Diagnosis Steps

### Step 1: Get Job Details

```bash
# Get job document from Firestore
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document=JOB_ID
```

Key fields to check:
- `status`: Current job status
- `progress.stage`: Current pipeline stage
- `progress.pct`: Completion percentage
- `error_details`: Error information if failed
- `updated_at`: Last update timestamp

### Step 2: Check Job Status

| Status | Meaning | Next Step |
|--------|---------|-----------|
| `queued` | Waiting to start | Check initiator worker |
| `clarifying` | Waiting for user input | Check research plan approval |
| `running` | Currently processing | Check current stage worker |
| `completed` | Finished successfully | N/A |
| `failed` | Error occurred | Check error_details |

### Step 3: Trace Job Through Pipeline

```bash
# Find all logs for this job
gcloud logging read "jsonPayload.job_id=\"JOB_ID\"" \
  --limit=100 \
  --format="table(timestamp,resource.labels.service_name,jsonPayload.message)"
```

### Step 4: Check Specific Stage

Based on `progress.stage`:

**Stage: queued**
```bash
# Check initiator worker
gcloud run services logs read kitesforu-worker-initiator --region=us-central1 --limit=50
```

**Stage: research**
```bash
# Check research planner
gcloud run services logs read kitesforu-worker-research-planner --region=us-central1 --limit=50

# Or tools executor
gcloud run services logs read kitesforu-worker-tools --region=us-central1 --limit=50
```

**Stage: script**
```bash
gcloud run services logs read kitesforu-worker-script --region=us-central1 --limit=50
```

**Stage: voice/mix**
```bash
gcloud run services logs read kitesforu-worker-audio --region=us-central1 --limit=50
```

### Step 5: Check for Dead Letters

```bash
# Check if message failed and went to dead letter
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=20 --auto-ack=false

# Look for job ID in messages
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=20 --auto-ack=false | grep JOB_ID
```

### Step 6: Check External API Status

If job failed with EXTERNAL_API_ERROR:

| Provider | Status Page |
|----------|-------------|
| OpenAI | https://status.openai.com |
| Anthropic | https://status.anthropic.com |
| ElevenLabs | https://status.elevenlabs.io |

```bash
# Check model router health records
gcloud firestore documents list \
  --database="(default)" \
  --collection=model_provider_health \
  --limit=10
```

## Common Issues

### Job Stuck in "queued"

**Causes**:
1. Initiator worker not processing messages
2. Pub/Sub push endpoint misconfigured
3. Worker crashed during startup

**Fixes**:
```bash
# Check initiator is running
gcloud run services describe kitesforu-worker-initiator --region=us-central1

# Check Pub/Sub subscription
gcloud pubsub subscriptions describe job-initiate-sub

# Manually trigger reprocessing
# (Requires re-publishing the message to the topic)
```

### Job Stuck in "running"

**Causes**:
1. Worker timeout
2. External API taking too long
3. Message lost

**Fixes**:
```bash
# Check for timeout errors
gcloud logging read "jsonPayload.job_id=\"JOB_ID\" AND severity=ERROR" --limit=20

# Check if message is in dead letter
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=10 --auto-ack=false
```

### Job Failed with Error

**Get error details**:
```bash
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document=JOB_ID \
  --format="json" | jq '.fields.error_details'
```

**Common errors**:
| Error Type | Cause | Solution |
|------------|-------|----------|
| EXTERNAL_API_ERROR | Provider API failed | Retry or check provider status |
| TTS_ERROR | Text-to-speech failed | Check ElevenLabs quota |
| TIMEOUT | Operation too slow | Check job complexity |
| INVALID_INPUT | Bad user input | Check topic/duration |
| QUOTA_EXCEEDED | API quota hit | Wait or use alternate provider |

## Retry a Failed Job

### Option 1: User Retries via UI

User can click "Retry" button in the frontend.

### Option 2: Manual Retry

```bash
# Update job status to queued
# (This requires using Firestore SDK or console)
# Then republish to job-initiate topic

# Using Python:
python3 << 'EOF'
from google.cloud import firestore
from google.cloud import pubsub_v1
import json

db = firestore.Client(project="kitesforu-dev")
publisher = pubsub_v1.PublisherClient()

job_id = "JOB_ID"
job_ref = db.collection("podcast_jobs").document(job_id)
job_ref.update({
    "status": "queued",
    "progress.stage": "queued",
    "error_details": None
})

topic_path = publisher.topic_path("kitesforu-dev", "job-initiate")
message = json.dumps({"job_id": job_id}).encode()
publisher.publish(topic_path, message)
print(f"Retried job {job_id}")
EOF
```

## Monitoring Dashboard

Check job metrics in Cloud Console:
- https://console.cloud.google.com/run (Cloud Run metrics)
- https://console.cloud.google.com/cloudpubsub (Pub/Sub metrics)
- https://console.cloud.google.com/firestore (Firestore data)

## Related

- [TROUBLESHOOTING.md](../TROUBLESHOOTING.md)
- [clear-dead-letters.md](./clear-dead-letters.md)
