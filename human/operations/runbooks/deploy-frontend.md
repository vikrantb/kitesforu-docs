# Runbook: Deploy Frontend

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Deploy the Next.js frontend to Cloud Run.

## ⚠️ Critical: Build-Time Variables

**Next.js `NEXT_PUBLIC_*` variables are inlined at BUILD TIME, not runtime.**

Do NOT use `gcloud run deploy --source` with `--set-build-env-vars` - this sets ENV vars, not Docker ARGs, and will break authentication. Always use the deploy script or build Docker image with `--build-arg`.

If you see `Error: @clerk/nextjs: Missing publishableKey` after deployment, immediately rollback to previous revision.

## Prerequisites

- [ ] GCP access configured
- [ ] Access to kitesforu-frontend repository
- [ ] Node.js 18+ installed locally

## Procedure

### Step 1: Verify Current State

```bash
# Check current revision
gcloud run services describe kitesforu-frontend --region=us-central1 --format="value(status.latestReadyRevisionName)"

# Check frontend health
curl -I https://kitesforu-frontend-m6zqve5yda-uc.a.run.app/
```

### Step 2: Pull Latest Changes

```bash
cd /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend
git checkout main
git pull origin main
```

### Step 3: Test Locally

```bash
# Install dependencies
npm install

# Run build
npm run build

# Run tests
npm test

# Test locally (optional)
npm run dev
# Visit http://localhost:3000
```

### Step 4: Deploy

**Option A: Using deploy script (recommended)**
```bash
./infra/deploy-frontend.sh
```

**Option B: Manual deployment**
```bash
# Build image
gcloud builds submit --tag us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/frontend:latest

# Deploy to Cloud Run
gcloud run deploy kitesforu-frontend \
  --image us-central1-docker.pkg.dev/kitesforu-dev/kitesforu-docker/frontend:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated
```

### Step 5: Verify Deployment

```bash
# Check new revision
gcloud run services describe kitesforu-frontend --region=us-central1 --format="value(status.latestReadyRevisionName)"

# Check frontend loads
curl -I https://kitesforu-frontend-m6zqve5yda-uc.a.run.app/

# Manual verification
# Open https://kitesforu-frontend-m6zqve5yda-uc.a.run.app in browser
# Test: Login flow, create job form, job history
```

### Step 6: Monitor

```bash
# Watch logs
gcloud run services logs read kitesforu-frontend --region=us-central1 --limit=50

# Check for client-side errors in browser console
```

## Rollback

If issues detected:

```bash
# List revisions
gcloud run revisions list --service=kitesforu-frontend --region=us-central1 --limit=5

# Rollback
gcloud run services update-traffic kitesforu-frontend \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION=100
```

## Environment Variables

Frontend requires these environment variables:

| Variable | Description |
|----------|-------------|
| NEXT_PUBLIC_API_URL | Backend API URL |
| NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY | Clerk public key |
| CLERK_SECRET_KEY | Clerk secret (from Secret Manager) |

Update if needed:
```bash
gcloud run services update kitesforu-frontend \
  --region=us-central1 \
  --set-env-vars="NEXT_PUBLIC_API_URL=https://kitesforu-api-m6zqve5yda-uc.a.run.app"
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Blank page | Check browser console for JS errors |
| API errors | Verify NEXT_PUBLIC_API_URL is correct |
| Auth not working | Check Clerk keys configuration |
| Build fails | Check Node version, dependencies |
| Hydration errors | Check for SSR/client mismatch |

## Manual Verification Checklist

After deployment, verify:

- [ ] Homepage loads
- [ ] Login/logout works
- [ ] Create job form displays
- [ ] Job history loads
- [ ] Job progress updates
- [ ] Audio player works

## Related

- [DEPLOYMENT.md](../DEPLOYMENT.md)
- [rollback-service.md](./rollback-service.md)
