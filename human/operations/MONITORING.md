# Monitoring Guide

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses GCP's built-in monitoring capabilities including Cloud Logging, Cloud Monitoring, and service-specific metrics.

## Key Metrics

### Cloud Run Services

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Request count | Total HTTP requests | N/A |
| Request latency | Response time (p50, p95, p99) | p99 > 10s |
| Container instance count | Active instances | > 8 instances |
| CPU utilization | Container CPU usage | > 80% sustained |
| Memory utilization | Container memory usage | > 85% |
| Error rate | 4xx/5xx responses | > 5% |

### Pub/Sub

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Unacked messages | Pending messages | > 100 |
| Oldest unacked message | Message age | > 5 minutes |
| Dead letter messages | Failed messages | > 10 |
| Publish latency | Message publish time | > 1s |

### Firestore

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Read operations | Document reads | Monitor trends |
| Write operations | Document writes | Monitor trends |
| Active connections | Client connections | Monitor trends |

## Logging

### View Logs

```bash
# API logs
gcloud run services logs read kitesforu-api --region=us-central1 --limit=100

# Worker logs
gcloud run services logs read kitesforu-worker-script --region=us-central1 --limit=100

# With filters
gcloud logging read "resource.type=cloud_run_revision AND severity>=ERROR" --limit=50
```

### Log Queries (Cloud Console)

```
# All errors in last hour
resource.type="cloud_run_revision"
severity>=ERROR
timestamp>="-1h"

# Specific job
resource.type="cloud_run_revision"
jsonPayload.job_id="JOB_ID"

# Worker failures
resource.type="cloud_run_revision"
resource.labels.service_name=~"kitesforu-worker"
severity=ERROR

# Pub/Sub delivery failures
resource.type="pubsub_subscription"
severity=ERROR
```

### Structured Logging

Workers use structured JSON logging:

```json
{
  "severity": "INFO",
  "message": "Processing job",
  "job_id": "abc-123",
  "stage": "script",
  "duration_ms": 1234
}
```

Query by job:
```
jsonPayload.job_id="abc-123"
```

## Dashboards

### GCP Console Dashboards

1. **Cloud Run Dashboard**: https://console.cloud.google.com/run
   - Request metrics
   - Instance counts
   - Error rates

2. **Pub/Sub Dashboard**: https://console.cloud.google.com/cloudpubsub
   - Message throughput
   - Unacked messages
   - Dead letter queue

3. **Firestore Dashboard**: https://console.cloud.google.com/firestore
   - Read/write operations
   - Storage usage

### Key Views

#### Service Health
```bash
# Quick health check
for service in kitesforu-api kitesforu-frontend kitesforu-worker-initiator; do
  echo "=== $service ==="
  gcloud run services describe $service --region=us-central1 --format="value(status.url)"
  curl -s -o /dev/null -w "%{http_code}\n" $(gcloud run services describe $service --region=us-central1 --format="value(status.url)")/health 2>/dev/null || echo "N/A"
done
```

#### Pub/Sub Health
```bash
# Check unacked messages
for sub in job-initiate-sub job-research-planner-sub job-execute-tools-sub job-script-sub job-audio-sub; do
  echo "$sub: $(gcloud pubsub subscriptions describe $sub --format='value(messageRetentionDuration)')"
done
```

## Alerts

### Setting Up Alerts

1. Go to Cloud Monitoring > Alerting
2. Create alerting policy
3. Configure notification channels (email, Slack)

### Recommended Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| High Error Rate | 5xx errors > 5% for 5 min | Critical |
| High Latency | p99 > 10s for 5 min | Warning |
| Dead Letter Messages | count > 10 | Warning |
| Worker Timeout | unacked > 100 for 10 min | Critical |
| Instance Scaling | instances > 8 | Info |

### Alert Response

1. **High Error Rate**:
   - Check recent deployments
   - Review error logs
   - Consider rollback

2. **High Latency**:
   - Check external API status (OpenAI, etc.)
   - Review resource usage
   - Check for cold starts

3. **Dead Letter Messages**:
   - Inspect failed messages
   - Check worker logs
   - Fix and reprocess

## Health Checks

### API Health Endpoint

```bash
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/health
```

Expected response:
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Worker Health

Workers don't have HTTP health endpoints. Monitor via:
- Pub/Sub message processing rate
- Error logs
- Job status progression

## Debugging Commands

### Recent Errors

```bash
gcloud logging read "resource.type=cloud_run_revision AND severity>=ERROR" \
  --limit=20 \
  --format="table(timestamp,resource.labels.service_name,jsonPayload.message)"
```

### Job Trace

```bash
# Find all logs for a specific job
gcloud logging read "jsonPayload.job_id=\"JOB_ID\"" \
  --limit=100 \
  --format="table(timestamp,resource.labels.service_name,jsonPayload.message)"
```

### Performance Analysis

```bash
# Slow requests
gcloud logging read "resource.type=cloud_run_revision AND httpRequest.latency>5s" \
  --limit=20 \
  --format="table(timestamp,httpRequest.requestUrl,httpRequest.latency)"
```

## External Dependencies

Monitor external API health:

| Service | Status Page |
|---------|-------------|
| OpenAI | https://status.openai.com |
| Anthropic | https://status.anthropic.com |
| Clerk | https://status.clerk.com |
| Stripe | https://status.stripe.com |
| ElevenLabs | https://status.elevenlabs.io |

## Runbook Links

- [diagnose-job.md](./runbooks/diagnose-job.md) - Job debugging
- [clear-dead-letters.md](./runbooks/clear-dead-letters.md) - Dead letter handling
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues
