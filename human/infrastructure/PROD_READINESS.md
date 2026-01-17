# Production Environment Readiness

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

This document outlines the requirements for deploying KitesForU to a production environment (`kitesforu-prod` project).

## Current State

| Component | Dev Ready | Prod Ready | Notes |
|-----------|-----------|------------|-------|
| Terraform | Yes | Partial | `prod.tfvars` exists but needs review |
| API CI/CD | Yes | No | Hardcoded to `kitesforu-dev` |
| Frontend CI/CD | Yes | No | Hardcoded to `kitesforu-dev` |
| Workers CI/CD | Yes | No | Hardcoded to `kitesforu-dev` |
| Secrets | Yes | No | Need prod Clerk, API keys |

## Terraform Configuration

### Files Status

| File | Prod Support |
|------|--------------|
| `variables.tf` | Yes - Uses variables for project_id |
| `prod.tfvars` | Partial - Needs Clerk URL update |
| `dev.tfvars` | Yes - Complete |
| `main.tf` | Yes - Uses var.project_id |
| `cloud_run_workers.tf` | Yes - Uses var.project_id |

### prod.tfvars Issues

```hcl
# Current (needs update)
clerk_jwks_url = "https://prompt-caribou-48.clerk.accounts.dev/.well-known/jwks.json"

# This is the DEV Clerk instance. Production requires a separate Clerk app.
```

**Action Required**: Before prod deployment, create a Clerk production application and update `clerk_jwks_url`.

### Applying to Production

```bash
cd kitesforu-infrastructure/terraform

# Plan
terraform plan -var-file=prod.tfvars -out=prod.tfplan

# Apply (with caution!)
terraform apply prod.tfplan
```

## CI/CD Requirements

### GitHub Secrets Needed for Prod

| Secret | Purpose |
|--------|---------|
| `GCP_SERVICE_ACCOUNT_KEY_PROD` | GCP auth for prod project |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY_PROD` | Clerk prod public key |
| `CLERK_SECRET_KEY_PROD` | Clerk prod secret key |
| `OPENAI_API_KEY` | Can share with dev |
| `ANTHROPIC_API_KEY` | Can share with dev |

### Workflow Modifications

#### Option 1: Environment-Based Workflows (Recommended)

```yaml
# Example: kitesforu-frontend/.github/workflows/ci.yml
env:
  GCP_PROJECT_ID: ${{ github.ref == 'refs/heads/main' && 'kitesforu-prod' || 'kitesforu-dev' }}
  CLERK_KEY: ${{ github.ref == 'refs/heads/main' && secrets.CLERK_PROD || secrets.CLERK_DEV }}
```

#### Option 2: Separate Workflows

Create `ci-prod.yml` triggered on release tags:
```yaml
on:
  push:
    tags:
      - 'v*'
```

### Infrastructure CI/CD

The `kitesforu-infrastructure` workflow currently only applies to dev:

```yaml
# Line 127: terraform apply -var-file=dev.tfvars
```

**Recommendation**: Add manual prod deployment job with approval:

```yaml
apply-prod:
  needs: plan
  if: github.event_name == 'workflow_dispatch'
  environment: production  # Requires approval
  steps:
    - name: Terraform Apply Prod
      run: terraform apply -var-file=prod.tfvars -auto-approve
```

## Secrets Management

### Secret Manager Secrets (Per Environment)

Secrets in Secret Manager are project-scoped, so prod will have its own:

| Secret | Dev | Prod |
|--------|-----|------|
| `openai-api-key` | Same | Same |
| `anthropic-api-key` | Same | Same |
| `clerk-secret-key` | Dev app | Prod app |
| `elevenlabs-api-key` | Same | Same |
| `tavily-api-key` | Same | Same |

### Frontend Environment Variables

| Variable | Build-Time | Runtime | Notes |
|----------|------------|---------|-------|
| `NEXT_PUBLIC_API_BASE` | Yes | No | Must match prod API URL |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Yes | No | Must be prod Clerk key |

## Checklist Before Prod Deployment

### GCP Project Setup
- [ ] Create `kitesforu-prod` GCP project
- [ ] Enable required APIs (same as dev)
- [ ] Set up billing alerts
- [ ] Configure IAM for production service accounts

### Clerk Setup
- [ ] Create Clerk production application
- [ ] Configure allowed origins for prod domain
- [ ] Update `prod.tfvars` with prod Clerk JWKS URL

### GitHub Setup
- [ ] Add prod secrets to all repositories
- [ ] Set up environment protection rules
- [ ] Configure required reviewers for prod deployments

### Terraform
- [ ] Review `prod.tfvars` values
- [ ] Plan and review changes
- [ ] Apply infrastructure

### DNS/Domains
- [ ] Configure custom domain (if not using Cloud Run URLs)
- [ ] Set up SSL certificates
- [ ] Update Clerk allowed origins

### Monitoring
- [ ] Set up Cloud Monitoring alerts
- [ ] Configure error reporting
- [ ] Set up uptime checks

## Post-Deployment Verification

```bash
# 1. Check all services are running
gcloud run services list --project=kitesforu-prod --region=us-central1

# 2. Verify API health
curl https://kitesforu-api-<hash>.run.app/health

# 3. Test frontend loads
curl -I https://kitesforu-frontend-<hash>.run.app/

# 4. Test authentication flow (manually in browser)

# 5. Create a test job and verify pipeline works
```

## Rollback Plan

If prod deployment fails:

1. **Cloud Run**: Route traffic to previous revision
2. **Terraform**: `terraform apply -var-file=prod.tfvars` with previous state
3. **Secrets**: Version control allows rollback via Secret Manager UI

## Related

- [DEPLOYMENT.md](../operations/DEPLOYMENT.md)
- [Terraform Configuration](./TERRAFORM.md)
- [CI_CD.md](../development/CI_CD.md)
