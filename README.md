# KitesForU Documentation

**Living documentation for the KitesForU podcast generation platform.**

Last Updated: 2026-01-07

---

## Documentation Structure

This repository is organized into two main sections optimized for different audiences:

### For AI Agents: `llm/`

Machine-readable documentation optimized for LLM parsing:
- **YAML reference files** with structured data
- **Explicit file paths** and configurations
- **Service specifications** in parseable format

```
llm/
├── README.md           # Navigation for AI agents
├── SYSTEM_STATE.md     # Core system state
├── reference/          # Machine-readable YAML files
├── services/           # Service specifications
└── operations/         # Operational commands
```

### For Humans: `human/`

Human-readable documentation with narratives and visualizations:
- **Mermaid diagrams** for visual understanding
- **Step-by-step guides** and runbooks
- **Explanatory narratives** for context

```
human/
├── README.md           # Navigation for humans
├── architecture/       # System architecture docs
├── diagrams/           # Mermaid visualizations
├── services/           # Service documentation
├── infrastructure/     # GCP resource docs
├── operations/         # Runbooks and procedures
├── development/        # Developer guides
├── integrations/       # External service docs
└── glossary/           # Domain terminology
```

---

## Quick Links

### LLM Resources
| Resource | Path |
|----------|------|
| File Index | [llm/reference/file-index.yaml](llm/reference/file-index.yaml) |
| API Endpoints | [llm/reference/endpoints.yaml](llm/reference/endpoints.yaml) |
| Environment Variables | [llm/reference/environment-vars.yaml](llm/reference/environment-vars.yaml) |
| GCP Resources | [llm/reference/gcp-resources.yaml](llm/reference/gcp-resources.yaml) |
| Job States | [llm/reference/job-states.yaml](llm/reference/job-states.yaml) |

### Human Resources
| Resource | Path |
|----------|------|
| System Overview | [human/architecture/OVERVIEW.md](human/architecture/OVERVIEW.md) |
| System Diagram | [human/diagrams/system-context.md](human/diagrams/system-context.md) |
| Local Setup | [human/development/LOCAL_SETUP.md](human/development/LOCAL_SETUP.md) |
| Deployment Guide | [human/operations/DEPLOYMENT.md](human/operations/DEPLOYMENT.md) |
| Troubleshooting | [human/operations/TROUBLESHOOTING.md](human/operations/TROUBLESHOOTING.md) |

---

## System Overview

**KitesForU** is an AI-powered podcast generation platform that transforms topics into polished audio content.

### Tech Stack
- **Frontend**: Next.js 14, Clerk Auth, Tailwind CSS
- **API**: FastAPI (Python 3.11)
- **Workers**: 7 Python Cloud Run services
- **Infrastructure**: GCP (Cloud Run, Firestore, Pub/Sub, Cloud Storage)

### Architecture
```
User → Frontend → API → Pub/Sub → Workers → Audio Output
                    ↓
              Firestore (State)
```

### Worker Pipeline
```
Initiator → Research Planner → Tools Executor → Script → Audio → Complete
```

---

## Source Repositories

| Repository | Purpose | Path |
|------------|---------|------|
| kitesforu-frontend | Next.js frontend | `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend` |
| kitesforu-api | FastAPI backend | `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-api` |
| kitesforu-workers | Python workers | `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-workers` |
| kitesforu-schemas | Shared Pydantic models | `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-schemas` |
| kitesforu-infrastructure | Terraform IaC | `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-infrastructure` |
| kitesforu-support | Operational scripts | `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-support` |

---

## GCP Project

- **Project ID**: `kitesforu-dev`
- **Region**: `us-central1`
- **Domain**: `beta.kitesforu.com`

---

## Contributing

1. Update relevant files in both `llm/` and `human/` sections
2. Keep YAML files in sync with codebase changes
3. Update diagrams when architecture changes
4. Test documentation links before committing
