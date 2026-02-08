# KitesForU Deployment Guide
<!-- last_verified: 2026-02-07 -->

## Service Deployment Matrix

| Service | Repo | CI/CD | Trigger | Region |
|---------|------|-------|---------|--------|
| kitesforu-frontend | kitesforu-frontend | GH Actions (ci.yml) | Auto on main push | us-central1 |
| kitesforu-api | kitesforu-api | GH Actions (ci.yml) | Auto on main push | us-central1 |
| 11 podcast workers | kitesforu-workers | GH Actions | Auto on main push | us-central1 |
| course-initiate-worker | kitesforu-course-workers | Cloud Build (LEGACY) | Manual submit | us-central1 |
| course-syllabus-worker | kitesforu-course-workers | Cloud Build (LEGACY) | Manual submit | us-central1 |
| course-orchestrate-worker | kitesforu-course-workers | Cloud Build (LEGACY) | Manual submit | us-central1 |
| course-planner-worker | kitesforu-course-workers | Cloud Build (LEGACY) | Manual submit | us-central1 |
| kitesforu-schemas | kitesforu-schemas | GH Actions | Auto on main push | Artifact Registry |
| Infrastructure | kitesforu-infrastructure | GH Actions | Plan on PR, Apply on main | - |

## Deployment Procedures

### Frontend & API (Auto-deploy)
1. Merge PR to main
2. GH Actions CI triggers automatically
3. Docker build → push to Container Registry → deploy to Cloud Run
4. Verify: `gcloud run revisions list --service=kitesforu-frontend --region=us-central1`
5. Frontend CRITICAL: Include pnpm-lock.yaml in commits (Docker uses --frozen-lockfile)

### Podcast Workers (Auto-deploy)
1. Merge PR to main in kitesforu-workers
2. GH Actions deploys all 11 services automatically
3. Services: worker-audio, worker-course-initiate, worker-course-orchestrate, worker-course-syllabus, worker-execute-tools, worker-initiator, worker-planner, worker-research-assimilator, and more
4. Verify each: `gcloud run revisions list --service=kitesforu-worker-{name} --region=us-central1`

### Course Workers (Manual Cloud Build — LEGACY)
```bash
cd kitesforu-course-workers
gcloud builds submit --config=cloudbuild.yaml --project=kitesforu-dev
```
- Deploys: course-initiate-worker, course-syllabus-worker, course-orchestrate-worker, course-planner-worker
- GOTCHA: Cloud Build needs explicit `cd` to correct directory
- GOTCHA: Uses `SHORT_SHA=$(git rev-parse --short HEAD)` for image tagging
- SHOULD MIGRATE: To GH Actions for consistency

### Schemas Package
1. Merge PR to main in kitesforu-schemas
2. GH Actions checks version, publishes to Artifact Registry if bumped
3. After publish: update version pin in consumers (API, workers, course-workers)
4. GOTCHA: Consumers won't get new version until their next deploy with updated requirements

### Infrastructure
1. Create PR in kitesforu-infrastructure
2. GH Actions runs `terraform plan` on PR
3. Review plan output
4. Merge to main → GH Actions runs `terraform apply`

## Verification Checklist

After ANY deployment:
1. Check CI/CD passed: `gh run list --repo vikrantb/{repo} --limit 5`
2. Check new revision exists: `gcloud run revisions list --service={name} --region=us-central1`
3. Check revision timestamp matches expected deploy time
4. Check service health: `gcloud run services describe {name} --region=us-central1`
5. Check logs for errors: `gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name={name} AND severity>=ERROR" --limit=10 --project=kitesforu-dev`

## Common Deployment Issues

### Frontend build fails
- **Cause**: stale pnpm-lock.yaml
- **Fix**: Run `pnpm install` locally, commit pnpm-lock.yaml
- **Prevention**: Always `pnpm install` (not npm) when adding deps

### Course workers not updating
- **Cause**: Cloud Build not triggered (manual process)
- **Fix**: Run `gcloud builds submit --config=cloudbuild.yaml --project=kitesforu-dev`

### Code on main but not deployed
- **Cause**: CI might have failed silently
- **Check**: `gh run list --repo vikrantb/{repo}` and `gcloud run revisions list`

### Schema version mismatch
- **Cause**: Schemas updated but consumers not redeployed
- **Fix**: Bump schema version in consumer's requirements, redeploy

## GCP Useful Commands
```bash
# List all Cloud Run services
gcloud run services list --region=us-central1 --project=kitesforu-dev

# Check specific service revisions
gcloud run revisions list --service={name} --region=us-central1

# Read recent logs for a service
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name={name}" --limit=20 --project=kitesforu-dev --format=json

# Read error logs only
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name={name} AND severity>=ERROR" --limit=10 --project=kitesforu-dev

# Check Cloud Build status
gcloud builds list --project=kitesforu-dev --limit=5
```

## PR Workflow (Mandatory for ALL changes)
1. Create feature branch from main
2. Commit changes to feature branch
3. Push and create PR via `gh pr create`
4. Wait for Copilot review (poll every 60s, up to 10 min)
5. Address Copilot comments (fix security/bugs, defer style)
6. Merge PR: `gh pr merge --squash`
7. Verify CI passes on main
8. Verify Cloud Run gets new revision
- NEVER push directly to main
- NEVER commit without PR, even for "small changes"
