<!--
================================================================================
FOR HUMAN CONSUMPTION ONLY - AI AGENTS SHOULD SKIP THIS FOLDER

This documentation is optimized for human readability with:
- Narrative explanations
- Visual diagrams (Mermaid)
- Step-by-step guides
- Context and reasoning

For machine-readable data, use: ../llm/
================================================================================
-->

# Human Documentation

Welcome to the human-readable documentation for KitesForU.

## Getting Started

### New to the project?
1. [System Overview](architecture/OVERVIEW.md) - Understand the big picture
2. [System Diagram](diagrams/system-context.md) - Visual architecture
3. [Local Setup](development/LOCAL_SETUP.md) - Set up your dev environment

### Need to deploy?
1. [Deployment Guide](operations/DEPLOYMENT.md) - Full deployment procedures
2. [Runbooks](operations/runbooks/) - Step-by-step operational tasks

### Something broken?
1. [Troubleshooting](operations/TROUBLESHOOTING.md) - Common issues and fixes
2. [Incident Response](operations/INCIDENT_RESPONSE.md) - When things go really wrong

---

## Documentation Map

### Architecture
Understanding how the system works:
- [OVERVIEW.md](architecture/OVERVIEW.md) - High-level system view
- [DATA_FLOW.md](architecture/DATA_FLOW.md) - Request-to-response flow
- [WORKER_PIPELINE.md](architecture/WORKER_PIPELINE.md) - Worker stage details

### Diagrams
Visual representations:
- [System Context](diagrams/system-context.md) - System architecture
- [Data Flow](diagrams/data-flow.md) - Request lifecycle
- [Worker Pipeline](diagrams/worker-pipeline.md) - Processing stages
- [Job State Machine](diagrams/job-state-machine.md) - Status transitions
- [Authentication Flow](diagrams/authentication-flow.md) - Clerk JWT
- [Deployment Flow](diagrams/deployment-flow.md) - CI/CD pipeline

### Services
Individual service documentation:
- **API**: [Overview](services/api/OVERVIEW.md) | [Endpoints](services/api/ENDPOINTS.md) | [Auth](services/api/AUTHENTICATION.md)
- **Workers**: [Overview](services/workers/OVERVIEW.md) | [Pipeline](services/workers/PIPELINE.md) | [Model Router](services/workers/MODEL_ROUTER.md)
- **Frontend**: [Overview](services/frontend/OVERVIEW.md)
- **Schemas**: [Overview](services/schemas/OVERVIEW.md)

### Infrastructure
GCP and Terraform:
- [GCP Resources](infrastructure/GCP_RESOURCES.md) - All cloud resources
- [Terraform](infrastructure/TERRAFORM.md) - Infrastructure as code
- [Secrets](infrastructure/SECRETS.md) - Secret management
- [Firestore](infrastructure/FIRESTORE.md) - Database design

### Operations
Running the system:
- [Deployment](operations/DEPLOYMENT.md) - Deploy procedures
- [Rollback](operations/ROLLBACK.md) - Rollback procedures
- [Monitoring](operations/MONITORING.md) - Observability
- [Troubleshooting](operations/TROUBLESHOOTING.md) - Problem solving
- [Incident Response](operations/INCIDENT_RESPONSE.md) - Emergency procedures

#### Runbooks
Step-by-step procedures:
- [Deploy API](operations/runbooks/deploy-api.md)
- [Deploy Workers](operations/runbooks/deploy-workers.md)
- [Deploy Frontend](operations/runbooks/deploy-frontend.md)
- [Rollback Service](operations/runbooks/rollback-service.md)
- [Diagnose Job](operations/runbooks/diagnose-job.md)
- [Clear Dead Letters](operations/runbooks/clear-dead-letters.md)

### Development
For developers:
- [Local Setup](development/LOCAL_SETUP.md) - Environment setup
- [Testing](development/TESTING.md) - Test strategy
- [CI/CD](development/CI_CD.md) - Pipeline docs

### Integrations
External services:
- [Clerk](integrations/CLERK.md) - Authentication
- [Stripe](integrations/STRIPE.md) - Payments
- [LLM Providers](integrations/LLM_PROVIDERS.md) - AI services

### Glossary
- [Terms](glossary/TERMS.md) - Domain terminology

---

## Quick Reference

### Key URLs
| Service | URL |
|---------|-----|
| Frontend | https://beta.kitesforu.com |
| API | https://kitesforu-api-m6zqve5yda-uc.a.run.app |
| GCP Console | https://console.cloud.google.com/home/dashboard?project=kitesforu-dev |

### Key Commands
```bash
# Check deployed versions
gcloud run services list --project=kitesforu-dev --region=us-central1

# View API logs
gcloud logging read 'resource.type="cloud_run_revision" resource.labels.service_name="kitesforu-api"' --project=kitesforu-dev --limit=50

# Run E2E tests
cd /Users/vikrantbhosale/gitprojects/kitesforu/kitetest
python3 scripts/run_beta_e2e_test.py
```

---

*This documentation is for humans. AI agents should use `../llm/` instead.*
