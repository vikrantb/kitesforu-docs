# P1: Cloud Run Cold Start Latency

## Problem
First podcast after idle takes 5-15s per stage waiting for containers. With 4-7 stages, 30-100s overhead.

## Root Cause
All 11 Cloud Run services have min-instances=0. Containers scale to zero after ~15 min idle.

## Fix
```bash
# Option A: Hot-path services only (~$15-30/month)
gcloud run services update kitesforu-worker-initiator --min-instances=1 --region=us-central1
gcloud run services update kitesforu-worker-audio --min-instances=1 --region=us-central1
```

## Code Paths
- Cloud Run configs: kitesforu-infrastructure/terraform/
- Docker image: kitesforu-workers/Dockerfile
- Pub/Sub subscriptions: GCP Console

## Priority: P1 | Impact: High | Effort: 10 min | Cost: ~$15-30/month
