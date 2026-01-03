# CI/CD Pipeline

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses GitHub Actions for CI/CD, with Cloud Build for Docker image builds and Cloud Run for deployments.

## Pipeline Architecture

```
Push to main
    │
    ▼
┌─────────────┐
│  Run Tests  │
└─────────────┘
    │
    ▼
┌─────────────┐
│ Build Image │
│(Cloud Build)│
└─────────────┘
    │
    ▼
┌─────────────┐
│Push to      │
│Artifact Reg │
└─────────────┘
    │
    ▼
┌─────────────┐
│Deploy to    │
│Cloud Run    │
└─────────────┘
```

## GitHub Actions Workflows

### API Workflow

```yaml
# kitesforu-api/.github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run tests
        run: pytest tests/ -v --cov=src/

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy kitesforu-api \
            --source . \
            --region us-central1 \
            --allow-unauthenticated
```

### Workers Workflow

```yaml
# kitesforu-workers/.github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/ -v

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy workers
        run: ./infra/deploy-workers.sh
```

### Frontend Workflow

```yaml
# kitesforu-frontend/.github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Deploy to Cloud Run
        run: ./infra/deploy-frontend.sh
```

## GitHub Secrets

Required secrets in each repository:

| Secret | Description |
|--------|-------------|
| GCP_SA_KEY | Service account JSON key for deployments |
| CODECOV_TOKEN | (Optional) For coverage reports |

### Setting Up GCP Service Account

```bash
# Create service account
gcloud iam service-accounts create github-actions \
  --display-name="GitHub Actions"

# Grant roles
gcloud projects add-iam-policy-binding kitesforu-dev \
  --member="serviceAccount:github-actions@kitesforu-dev.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding kitesforu-dev \
  --member="serviceAccount:github-actions@kitesforu-dev.iam.gserviceaccount.com" \
  --role="roles/cloudbuild.builds.editor"

gcloud projects add-iam-policy-binding kitesforu-dev \
  --member="serviceAccount:github-actions@kitesforu-dev.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding kitesforu-dev \
  --member="serviceAccount:github-actions@kitesforu-dev.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

gcloud projects add-iam-policy-binding kitesforu-dev \
  --member="serviceAccount:github-actions@kitesforu-dev.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Create key
gcloud iam service-accounts keys create key.json \
  --iam-account=github-actions@kitesforu-dev.iam.gserviceaccount.com

# Add to GitHub secrets as GCP_SA_KEY
cat key.json | base64
```

## Branch Protection

Recommended settings for `main` branch:

- [x] Require pull request reviews
- [x] Require status checks to pass
- [x] Require branches to be up to date
- [x] Include administrators

## Deployment Flow

### Standard Deployment

1. Create feature branch
2. Push changes, tests run automatically
3. Create PR, tests run again
4. Merge to main
5. CI/CD deploys automatically

### Manual Deployment

For emergency or controlled deployments:

```bash
# API
cd kitesforu-api
./infra/deploy-api.sh

# Workers
cd kitesforu-workers
./infra/deploy-workers.sh

# Frontend
cd kitesforu-frontend
./infra/deploy-frontend.sh
```

## Monitoring Deployments

### GitHub Actions

Check workflow runs:
- https://github.com/vikrantb/kitesforu-api/actions
- https://github.com/vikrantb/kitesforu-frontend/actions
- https://github.com/vikrantb/kitesforu-workers/actions

### Cloud Build

```bash
# List recent builds
gcloud builds list --limit=10

# View build logs
gcloud builds log BUILD_ID
```

### Cloud Run Revisions

```bash
# List revisions
gcloud run revisions list --service=kitesforu-api --region=us-central1
```

## Rollback in CI/CD

If deployment fails or causes issues:

1. **Automatic**: GitHub Actions will show failure
2. **Manual rollback**: See [rollback-service.md](../operations/runbooks/rollback-service.md)

## Environment Variables in CI

Environment variables for deployments are:
1. Defined in deploy scripts
2. Stored in Secret Manager
3. Mounted to Cloud Run services

Updating secrets requires redeployment.

## Related

- [DEPLOYMENT.md](../operations/DEPLOYMENT.md)
- [LOCAL_SETUP.md](./LOCAL_SETUP.md)
- [TESTING.md](./TESTING.md)
