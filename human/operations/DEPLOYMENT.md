# Deployment Guide

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses GitHub Actions for CI/CD with Cloud Build for container builds. All services deploy to Cloud Run in us-central1.

## Prerequisites

1. **Access Requirements**:
   - GCP project access (kitesforu-dev)
   - GitHub repository write access
   - gcloud CLI authenticated

2. **Environment Setup**:
   ```bash
   gcloud auth login
   gcloud config set project kitesforu-dev
   ```

## Deployment Methods

### 1. Automated (Recommended)

Push to `main` branch triggers automatic deployment:

```bash
git checkout main
git pull
git merge feature/my-feature
git push origin main
```

GitHub Actions will:
1. Run tests
2. Build Docker image
3. Push to Artifact Registry
4. Deploy to Cloud Run

### 2. Manual Deployment

Use deploy scripts in each repository:

```bash
# API
./infra/deploy-api.sh

# Workers
./infra/deploy-workers.sh

# Frontend
./infra/deploy-frontend.sh
```

## Service-Specific Deployment

### API Service

| Attribute | Value |
|-----------|-------|
| Repository | kitesforu-api |
| Image | us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/kitesforu-api |
| Deploy Script | infra/deploy-api.sh |
| Min Instances | 0 |
| Max Instances | 10 |

```bash
cd kitesforu-api
./infra/deploy-api.sh
```

### Workers

| Worker | Deploy Script |
|--------|---------------|
| All Workers | infra/deploy-workers.sh |

Workers share a single Docker image but deploy as separate Cloud Run services:

```bash
cd kitesforu-workers
./infra/deploy-workers.sh
```

### Frontend

| Attribute | Value |
|-----------|-------|
| Repository | kitesforu-frontend |
| Image | us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/frontend |
| Deploy Script | infra/deploy-frontend.sh |

```bash
cd kitesforu-frontend
./infra/deploy-frontend.sh
```

## Deployment Verification

### Check Service Status

```bash
# List all services
gcloud run services list --region=us-central1

# Check specific service
gcloud run services describe kitesforu-api --region=us-central1
```

### View Recent Revisions

```bash
gcloud run revisions list --service=kitesforu-api --region=us-central1 --limit=5
```

### Check Logs

```bash
gcloud run services logs read kitesforu-api --region=us-central1 --limit=50
```

### Health Check

```bash
# API health
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/health

# Frontend
curl https://kitesforu-frontend-m6zqve5yda-uc.a.run.app/
```

## Deployment Checklist

### Before Deployment

- [ ] All tests passing locally
- [ ] Code reviewed and approved
- [ ] Environment variables configured
- [ ] Secrets updated if needed
- [ ] Database migrations applied (if any)

### After Deployment

- [ ] Service is healthy
- [ ] Logs show no errors
- [ ] Test critical user flows
- [ ] Monitor error rates
- [ ] Verify Pub/Sub message processing

## Environment Variables

### Updating Environment Variables

1. **Add to Secret Manager** (for secrets):
   ```bash
   echo -n "value" | gcloud secrets versions add SECRET_NAME --data-file=-
   ```

2. **Update Cloud Run Service**:
   ```bash
   gcloud run services update kitesforu-api \
     --region=us-central1 \
     --set-env-vars="NEW_VAR=value"
   ```

3. **For secrets**:
   ```bash
   gcloud run services update kitesforu-api \
     --region=us-central1 \
     --set-secrets="ENV_VAR=secret-name:latest"
   ```

## Troubleshooting Deployments

### Build Failures

1. Check Cloud Build logs:
   ```bash
   gcloud builds list --limit=5
   gcloud builds log BUILD_ID
   ```

2. Common issues:
   - Dockerfile syntax errors
   - Missing dependencies
   - Test failures

### Deployment Failures

1. Check service status:
   ```bash
   gcloud run services describe SERVICE_NAME --region=us-central1
   ```

2. Check revision logs:
   ```bash
   gcloud run services logs read SERVICE_NAME --region=us-central1
   ```

3. Common issues:
   - Missing secrets
   - Invalid environment variables
   - Container startup failures

### Rollback

If deployment causes issues, rollback immediately:

```bash
# List revisions
gcloud run revisions list --service=kitesforu-api --region=us-central1

# Rollback to previous revision
gcloud run services update-traffic kitesforu-api \
  --region=us-central1 \
  --to-revisions=REVISION_NAME=100
```

See [ROLLBACK.md](./ROLLBACK.md) for detailed rollback procedures.
