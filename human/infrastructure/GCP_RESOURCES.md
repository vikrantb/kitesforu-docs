# GCP Resources

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU runs on Google Cloud Platform in the `kitesforu-dev` project. All infrastructure is managed with Terraform.

## Project Details

| Attribute | Value |
|-----------|-------|
| Project ID | kitesforu-dev |
| Region | us-central1 |
| Organization | Personal |

## Cloud Run Services

### Frontend
| Service | URL |
|---------|-----|
| kitesforu-frontend | https://kitesforu-frontend-m6zqve5yda-uc.a.run.app |

### API
| Service | URL |
|---------|-----|
| kitesforu-api | https://kitesforu-api-m6zqve5yda-uc.a.run.app |

### Workers
| Service | Pub/Sub Topic |
|---------|---------------|
| kitesforu-worker-initiator | job-initiate |
| kitesforu-worker-research-planner | job-research-planner |
| kitesforu-worker-tools | job-execute-tools |
| kitesforu-worker-script | job-script |
| kitesforu-worker-audio | job-audio |

## Firestore

| Attribute | Value |
|-----------|-------|
| Database ID | (default) |
| Mode | Native |
| Location | us-central1 |

### Collections
- podcast_jobs
- users
- credit_transactions
- model_provider_health
- model_quotas
- model_router_alerts
- routing_decisions

## Pub/Sub

### Topics
| Topic | Subscription | Worker |
|-------|--------------|--------|
| job-initiate | job-initiate-sub | initiator |
| job-research-planner | job-research-planner-sub | research-planner |
| job-execute-tools | job-execute-tools-sub | tools |
| job-script | job-script-sub | script |
| job-audio | job-audio-sub | audio |
| workers-dead-letter | workers-dead-letter-sub | (none) |

### Configuration
- Message retention: 30 minutes (topics), 7 days (dead-letter)
- Ack deadline: 120-480 seconds (varies by worker)
- Max retries: 5
- Push authentication: OIDC

## Cloud Storage

### Buckets
| Bucket | Purpose | Lifecycle |
|--------|---------|-----------|
| kitesforu-podcasts | Production audio | 2 days |
| kitesforu-dev-podcasts | Development audio | 2 days |
| kitesforu-dev-worker-logs | Debug logs | - |

## Artifact Registry

### Docker Repository
- **Name**: kitesforu-docker
- **Location**: us-central1
- **Images**: frontend, kitesforu-api, kitesforu-workers

### Python Repository
- **Name**: kitesforu-python
- **Location**: us-central1
- **Packages**: kitesforu-schemas

## Secret Manager

### Secrets
| Secret | Used By |
|--------|---------|
| openai-api-key | API, Workers |
| anthropic-api-key | API, Workers |
| google-ai-api-key | Workers |
| elevenlabs-api-key | API, Workers |
| deepgram-api-key | Workers |
| clerk-secret-key | API, Frontend |
| zoho-smtp-user | API |
| zoho-smtp-pass | API |
| pubsub-verification-token | Workers |

## Service Accounts

### kitesforu-worker
**Purpose**: Worker service identity

**Roles**:
- roles/datastore.user
- roles/pubsub.publisher
- roles/pubsub.subscriber
- roles/storage.objectAdmin
- roles/secretmanager.secretAccessor
- roles/logging.logWriter

### github-actions
**Purpose**: CI/CD deployment

**Roles**:
- roles/run.admin
- roles/cloudbuild.builds.editor
- roles/iam.serviceAccountUser
- roles/storage.admin
- roles/artifactregistry.writer

## APIs Enabled

- Cloud Build
- Cloud Run
- Firestore
- Cloud Storage
- Pub/Sub
- Text-to-Speech
- Secret Manager

## Resource Commands

### List Cloud Run Services
```bash
gcloud run services list --region=us-central1
```

### List Pub/Sub Topics
```bash
gcloud pubsub topics list
```

### List Storage Buckets
```bash
gsutil ls
```

### View Service Logs
```bash
gcloud run services logs read kitesforu-api --region=us-central1 --limit=100
```
