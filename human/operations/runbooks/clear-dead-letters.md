# Runbook: Clear Dead Letter Queue

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Process and clear messages from the dead letter queue (DLQ). Messages end up here after exceeding retry attempts.

## Prerequisites

- GCP access configured
- Understanding of why messages failed

## Procedure

### Step 1: View Dead Letter Messages

```bash
# View messages without acknowledging
gcloud pubsub subscriptions pull workers-dead-letter-sub \
  --limit=10 \
  --auto-ack=false

# View message contents
gcloud pubsub subscriptions pull workers-dead-letter-sub \
  --limit=10 \
  --auto-ack=false \
  --format="table(message.data.decode('base64').decode('utf-8'),message.attributes)"
```

### Step 2: Analyze Message Content

Each message contains:
- `job_id`: The failed job
- Original topic: Which worker failed
- Error attributes: Why it failed

```bash
# Decode and analyze
gcloud pubsub subscriptions pull workers-dead-letter-sub \
  --limit=5 \
  --auto-ack=false \
  --format=json | jq '.[].message.data | @base64d | fromjson'
```

### Step 3: Determine Action

| Scenario | Action |
|----------|--------|
| Bug in code | Fix code, then reprocess |
| External API timeout | Check provider, then reprocess |
| Invalid job data | Mark job as failed, clear message |
| Quota exceeded | Wait for quota reset, then reprocess |

### Step 4A: Reprocess Messages

If the underlying issue is fixed:

```bash
# For each job in DLQ, re-trigger processing
# Option 1: Update job and republish

python3 << 'EOF'
from google.cloud import pubsub_v1
import json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("kitesforu-dev", "job-initiate")

# Get job_ids from DLQ messages first
job_ids = ["job-id-1", "job-id-2"]  # Replace with actual IDs

for job_id in job_ids:
    message = json.dumps({"job_id": job_id}).encode()
    publisher.publish(topic_path, message)
    print(f"Republished {job_id}")
EOF
```

### Step 4B: Clear Without Reprocessing

If messages should be discarded:

```bash
# Acknowledge and clear messages
gcloud pubsub subscriptions pull workers-dead-letter-sub \
  --limit=100 \
  --auto-ack

# For large quantities, pull in batches
while true; do
  COUNT=$(gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=100 --auto-ack --format="value(message.messageId)" | wc -l)
  echo "Cleared $COUNT messages"
  if [ "$COUNT" -lt 100 ]; then break; fi
done
```

### Step 5: Update Failed Jobs

Mark jobs as failed if they won't be retried:

```bash
python3 << 'EOF'
from google.cloud import firestore
from datetime import datetime

db = firestore.Client(project="kitesforu-dev")

# Job IDs that won't be retried
job_ids = ["job-id-1", "job-id-2"]

for job_id in job_ids:
    job_ref = db.collection("podcast_jobs").document(job_id)
    job_ref.update({
        "status": "failed",
        "progress.stage": "failed",
        "error_details": {
            "error_type": "DEAD_LETTER",
            "message": "Exceeded maximum retry attempts",
            "timestamp": datetime.utcnow().isoformat()
        },
        "updated_at": firestore.SERVER_TIMESTAMP
    })
    print(f"Marked {job_id} as failed")
EOF
```

### Step 6: Verify Queue is Clear

```bash
# Check remaining messages
gcloud pubsub subscriptions describe workers-dead-letter-sub \
  --format="value(numUndeliveredMessages)"

# Should return 0 or empty
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=1 --auto-ack=false
```

## Bulk Operations

### Clear All Messages (Dangerous)

Only use when all messages should be discarded:

```bash
# Seek to end (clears all unacked messages)
gcloud pubsub subscriptions seek workers-dead-letter-sub \
  --time=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

### Export Messages Before Clearing

```bash
# Save messages to file
gcloud pubsub subscriptions pull workers-dead-letter-sub \
  --limit=1000 \
  --auto-ack=false \
  --format=json > dead_letters_backup.json

# Then clear
gcloud pubsub subscriptions pull workers-dead-letter-sub \
  --limit=1000 \
  --auto-ack
```

## Monitoring

### Check DLQ Size

```bash
# Current undelivered count
gcloud pubsub subscriptions describe workers-dead-letter-sub \
  --format="value(numUndeliveredMessages)"
```

### Set Up Alert

In Cloud Console > Monitoring:
- Metric: pubsub.googleapis.com/subscription/num_undelivered_messages
- Resource: workers-dead-letter-sub
- Threshold: > 10 for 10 minutes

## Prevention

To reduce DLQ messages:

1. **Increase retries**: Adjust subscription retry policy
2. **Increase timeout**: Extend ack deadline for slow workers
3. **Add error handling**: Catch and handle errors in workers
4. **Monitor proactively**: Alert on increasing DLQ size

## Related

- [diagnose-job.md](./diagnose-job.md)
- [TROUBLESHOOTING.md](../TROUBLESHOOTING.md)
- [MONITORING.md](../MONITORING.md)
