# LLM Documentation Index

Machine-readable documentation for AI agents working with KitesForU.

## Navigation

### Reference Files (YAML)
Structured data for quick lookups:

| File | Purpose | Use When |
|------|---------|----------|
| [reference/file-index.yaml](reference/file-index.yaml) | All source files with paths | Finding code locations |
| [reference/endpoints.yaml](reference/endpoints.yaml) | API endpoint specifications | Working with API |
| [reference/environment-vars.yaml](reference/environment-vars.yaml) | Environment variables | Configuration tasks |
| [reference/gcp-resources.yaml](reference/gcp-resources.yaml) | GCP resource inventory | Infrastructure queries |
| [reference/pubsub-topics.yaml](reference/pubsub-topics.yaml) | Pub/Sub topic mappings | Worker pipeline work |
| [reference/firestore-schema.yaml](reference/firestore-schema.yaml) | Database collections | Data model queries |
| [reference/job-states.yaml](reference/job-states.yaml) | Job state machine | Status transitions |
| [reference/worker-stages.yaml](reference/worker-stages.yaml) | Worker pipeline spec | Worker development |
| [reference/error-codes.yaml](reference/error-codes.yaml) | Error types | Debugging |

### Service Specifications (YAML)
Detailed service configurations:

| File | Purpose |
|------|---------|
| [services/api.yaml](services/api.yaml) | API service spec |
| [services/workers.yaml](services/workers.yaml) | Workers service spec |
| [services/frontend.yaml](services/frontend.yaml) | Frontend service spec |
| [services/schemas.yaml](services/schemas.yaml) | Schemas package spec |

### Operations (YAML)
Runbook commands in structured format:

| File | Purpose |
|------|---------|
| [operations/deployment.yaml](operations/deployment.yaml) | Deploy commands |
| [operations/rollback.yaml](operations/rollback.yaml) | Rollback commands |
| [operations/troubleshooting.yaml](operations/troubleshooting.yaml) | Debug commands |

### System State
| File | Purpose |
|------|---------|
| [SYSTEM_STATE.md](SYSTEM_STATE.md) | Core system state document |

## Usage Pattern

```yaml
# To find a file location:
read: llm/reference/file-index.yaml
lookup: repository → key_files → target

# To understand an API endpoint:
read: llm/reference/endpoints.yaml
lookup: endpoints → path → method

# To deploy a service:
read: llm/operations/deployment.yaml
execute: services → target → commands
```

## Base Path
All file paths are relative to: `/Users/vikrantbhosale/gitprojects/kitesforu/`
