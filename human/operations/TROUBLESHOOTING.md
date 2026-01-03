# Troubleshooting Guide

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Quick Diagnosis

### Check System Health

```bash
# All services status
gcloud run services list --region=us-central1 --format="table(SERVICE,URL,LAST_DEPLOYED_BY)"

# Recent errors
gcloud logging read "severity>=ERROR" --limit=20 --format="table(timestamp,resource.labels.service_name,textPayload)"
```

## Common Issues

### Jobs

#### Job Stuck in "queued" Status

**Symptoms**: Job created but never starts processing

**Diagnosis**:
```bash
# Check Pub/Sub delivery
gcloud pubsub subscriptions describe job-initiate-sub --format="table(ackDeadlineSeconds,messageRetentionDuration)"

# Check initiator logs
gcloud run services logs read kitesforu-worker-initiator --region=us-central1 --limit=20
```

**Causes & Solutions**:
1. **Pub/Sub push not configured**: Verify push endpoint URL
2. **Worker not deployed**: Check service exists
3. **Authentication failure**: Verify OIDC service account

#### Job Stuck in "running" Status

**Symptoms**: Job started but never completes

**Diagnosis**:
```bash
# Get job details
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document=JOB_ID

# Check worker logs
gcloud logging read "jsonPayload.job_id=\"JOB_ID\"" --limit=50
```

**Causes & Solutions**:
1. **Worker timeout**: Check for long-running operations
2. **External API failure**: Check OpenAI/Anthropic status
3. **Message lost**: Check dead letter queue

#### Job Failed with Error

**Diagnosis**:
```bash
# Get error details from job
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document=JOB_ID \
  --format="json" | jq '.fields.error_details'
```

**Common Errors**:
| Error | Cause | Solution |
|-------|-------|----------|
| EXTERNAL_API_ERROR | OpenAI/Anthropic failure | Check provider status, retry |
| TTS_ERROR | Text-to-speech failed | Check ElevenLabs quota |
| TIMEOUT | Operation exceeded limit | Check job complexity |
| INVALID_INPUT | Bad user input | Check topic/duration values |

### API Issues

#### 401 Unauthorized

**Diagnosis**:
```bash
# Check Clerk secret
gcloud secrets versions access latest --secret=clerk-secret-key | head -c 20
```

**Causes & Solutions**:
1. **Invalid JWT**: User needs to re-authenticate
2. **Clerk secret mismatch**: Update secret in Secret Manager
3. **Clock skew**: JWT expired due to time difference

#### 500 Internal Server Error

**Diagnosis**:
```bash
# Check API logs
gcloud run services logs read kitesforu-api --region=us-central1 --limit=50

# Check for Python exceptions
gcloud logging read "resource.labels.service_name=kitesforu-api AND severity=ERROR" --limit=20
```

**Causes & Solutions**:
1. **Database connection**: Check Firestore connectivity
2. **Missing secret**: Verify all secrets are mounted
3. **Code bug**: Review recent deployments

### Worker Issues

#### Workers Not Processing Messages

**Diagnosis**:
```bash
# Check unacked messages
gcloud pubsub subscriptions describe job-initiate-sub

# Check worker instances
gcloud run services describe kitesforu-worker-initiator --region=us-central1
```

**Causes & Solutions**:
1. **Push endpoint misconfigured**: Verify URL matches service
2. **Service account permissions**: Check IAM roles
3. **Worker crashed**: Check logs for startup errors

#### Dead Letter Queue Filling Up

**Diagnosis**:
```bash
# View dead letter messages
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=5 --auto-ack=false
```

**Causes & Solutions**:
1. **Repeated failures**: Check worker error logs
2. **Timeout**: Increase ack deadline
3. **Bug in worker**: Fix code and redeploy

### External API Issues

#### OpenAI API Errors

**Symptoms**: Research or script generation fails

**Diagnosis**:
```bash
gcloud logging read "jsonPayload.provider=\"openai\" AND severity=ERROR" --limit=20
```

**Common Issues**:
| Error | Solution |
|-------|----------|
| Rate limited | Wait and retry, check usage |
| Invalid API key | Update secret |
| Context too long | Reduce input size |
| Service unavailable | Check status.openai.com |

#### ElevenLabs TTS Errors

**Symptoms**: Audio generation fails

**Diagnosis**:
```bash
gcloud logging read "jsonPayload.stage=\"audio\" AND severity=ERROR" --limit=20
```

**Common Issues**:
| Error | Solution |
|-------|----------|
| Quota exceeded | Check subscription limits |
| Invalid voice ID | Verify voice configuration |
| Text too long | Split into chunks |

### Database Issues

#### Firestore Connection Errors

**Diagnosis**:
```bash
# Check Firestore status
gcloud firestore operations list --database="(default)"

# Test connection
python3 -c "from google.cloud import firestore; db = firestore.Client(); print('Connected')"
```

**Solutions**:
1. Check IAM permissions for service account
2. Verify project configuration
3. Check for quota limits

#### Slow Queries

**Diagnosis**:
```bash
# Check for missing indexes
gcloud logging read "resource.type=datastore_database AND textPayload:\"missing index\"" --limit=10
```

**Solutions**:
1. Add composite indexes
2. Optimize query filters
3. Use pagination for large result sets

### Deployment Issues

#### Build Failures

**Diagnosis**:
```bash
gcloud builds list --limit=5
gcloud builds log BUILD_ID
```

**Common Issues**:
| Error | Solution |
|-------|----------|
| Dockerfile not found | Check file path |
| Dependencies failed | Check requirements.txt |
| Tests failed | Fix failing tests |
| Out of memory | Increase build machine type |

#### Service Won't Start

**Diagnosis**:
```bash
gcloud run services describe SERVICE_NAME --region=us-central1
gcloud run services logs read SERVICE_NAME --region=us-central1 --limit=50
```

**Common Issues**:
| Error | Solution |
|-------|----------|
| Port binding failed | Ensure PORT env var is used |
| Secret not found | Check secret name and version |
| Import error | Check dependencies installed |

## Debug Commands Reference

### Firestore Queries

```bash
# Get job by ID
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document=JOB_ID

# List recent jobs for user
gcloud firestore documents list \
  --database="(default)" \
  --collection=podcast_jobs \
  --filter="user_id=USER_ID" \
  --order-by="created_at desc" \
  --limit=10
```

### Log Queries

```bash
# All logs for a job
gcloud logging read "jsonPayload.job_id=\"JOB_ID\"" --limit=100

# Errors in last hour
gcloud logging read "severity>=ERROR AND timestamp>=\"$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)\"" --limit=50

# Specific service errors
gcloud logging read "resource.labels.service_name=kitesforu-api AND severity=ERROR" --limit=20
```

### Service Commands

```bash
# Restart service (redeploy same image)
gcloud run services update SERVICE_NAME --region=us-central1 --no-traffic

# Force new instance
gcloud run services update SERVICE_NAME --region=us-central1 --clear-env-vars && \
gcloud run services update SERVICE_NAME --region=us-central1 --set-env-vars="FORCE_RESTART=$(date +%s)"
```

## Escalation Path

1. **Self-service**: Use this guide + runbooks
2. **Team channel**: Post in #kitesforu-dev
3. **On-call**: Contact on-call engineer
4. **External**: Check provider status pages

## Related Documentation

- [MONITORING.md](./MONITORING.md) - Monitoring setup
- [INCIDENT_RESPONSE.md](./INCIDENT_RESPONSE.md) - Incident handling
- [runbooks/diagnose-job.md](./runbooks/diagnose-job.md) - Job debugging
