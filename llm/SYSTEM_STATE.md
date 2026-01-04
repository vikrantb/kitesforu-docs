# KitesForU System State

<!-- METADATA
file: llm/SYSTEM_STATE.md
purpose: Core system state for LLM agents
updated: 2026-01-04
-->

## Overview

KitesForU is an AI-powered podcast generation platform. Users provide a topic, and the system researches, scripts, and synthesizes a podcast automatically.

## Quick Reference

| Aspect | Value |
|--------|-------|
| Environment | kitesforu-dev (GCP) |
| Region | us-central1 |
| Services | 9 Cloud Run services |
| Database | Firestore (native mode) |
| Messaging | Pub/Sub (6 topics) |
| Storage | Cloud Storage |
| Auth | Clerk JWT |
| Payments | Stripe |

## Architecture

```
User → Frontend (Next.js) → API (FastAPI) → Pub/Sub → Workers → GCS
                                  ↓
                            Firestore
```

## Services

### Frontend
- **Service**: kitesforu-frontend
- **URL**: https://kitesforu-frontend-m6zqve5yda-uc.a.run.app
- **Stack**: Next.js 14, TypeScript, Tailwind, Clerk

### API
- **Service**: kitesforu-api
- **URL**: https://kitesforu-api-m6zqve5yda-uc.a.run.app
- **Stack**: FastAPI, Python 3.11+

### Workers (5 services)
1. **kitesforu-worker-initiator** → job-initiate
2. **kitesforu-worker-research-planner** → job-research-planner
3. **kitesforu-worker-tools** → job-execute-tools
4. **kitesforu-worker-script** → job-script
5. **kitesforu-worker-audio** → job-audio

## Pipeline Flow

```
1. API creates job → Firestore
2. API publishes → job-initiate topic
3. Initiator → clarifier questions (optional)
4. Research Planner → generates plan, waits for approval
5. Tools Executor → web search, URL extraction
6. Script Generator → LLM creates podcast script
7. Audio Worker → TTS synthesis, upload to GCS
8. Job marked COMPLETED
```

## Job States

### JobStatus
- `queued` → Job waiting
- `clarifying` → Needs user input
- `running` → Processing
- `completed` → Success
- `failed` → Error

### JobStage
`queued` → `research` → `script` → `voice` → `mix` → `publish` → `completed`

## Key Collections (Firestore)

- **podcast_jobs**: Job state and metadata
- **users**: User data and credits
- **credit_transactions**: Credit history
- **model_provider_health**: LLM provider status
- **routing_decisions**: Model selection audit

## Pub/Sub Topics

| Topic | Worker | Purpose |
|-------|--------|---------|
| job-initiate | initiator | Start job |
| job-research-planner | research-planner | Create plan |
| job-execute-tools | tools | Execute research |
| job-script | script | Generate script |
| job-audio | audio | TTS synthesis |
| workers-dead-letter | - | Failed messages |

## LLM Providers

| Provider | Models | Usage |
|----------|--------|-------|
| OpenAI | gpt-4o, gpt-4o-mini, tts-1 | Primary |
| Anthropic | claude-3-5-sonnet | Fallback |
| Google | gemini-1.5-pro, gemini-2.0-flash, tts | Fallback |
| ElevenLabs | multilingual-v2 | Premium TTS |

### Model Selection

Models are selected via the **Model Router** based on:
- User tier (trial, enthusiast, pro_creator, ultimate_creator)
- Task type and complexity
- Model capabilities from `config/model_catalog.csv`

### Complexity Routing

Non-premium tiers get complexity-based routing:
- **SIMPLE**: Cheapest enabled model
- **MODERATE**: Standard tier model
- **COMPLEX**: Higher tier model

Premium tiers (pro_creator, ultimate_creator) always get the best available model.

### Research Execution

| User Tier | Executor | Description |
|-----------|----------|-------------|
| trial, enthusiast | DirectToolExecutor | Sequential execution |
| pro_creator, ultimate_creator | AgenticExecutor | LLM-driven with tool calling |

## Key Endpoints

```
POST   /v1/podcasts                         # Create job
GET    /v1/podcasts/{id}/status             # Get status
GET    /v1/podcasts/{id}/result             # Get result
PATCH  /v1/podcasts/{id}/answers            # Submit clarifier
GET    /v1/podcasts/{id}/research-plan      # Get plan
POST   /v1/podcasts/{id}/research-plan/approve  # Approve plan
GET    /v1/me/credits                       # Get balance
POST   /v1/checkout                         # Buy credits
```

## File Paths

| Repository | Path |
|------------|------|
| Monorepo root | /Users/vikrantbhosale/gitprojects/kitesforu |
| API | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-api |
| Workers | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-workers |
| Frontend | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend |
| Schemas | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-schemas |
| Infrastructure | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-infrastructure |
| Support | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-support |
| Docs | /Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-docs |

## Environment Variables

### Critical
- `OPENAI_API_KEY` - LLM and TTS
- `ANTHROPIC_API_KEY` - Fallback LLM
- `CLERK_JWKS_URL` - Auth verification
- `STRIPE_SECRET_KEY` - Payments

### GCP
- `GOOGLE_PROJECT_ID`: kitesforu-dev
- `GCS_BUCKET`: kitesforu-podcasts

## Deploy Commands

```bash
# API
cd kitesforu-api && ./infra/deploy-api.sh

# Workers
cd kitesforu-workers && ./infra/deploy-workers.sh

# Frontend
cd kitesforu-frontend && ./infra/deploy-frontend.sh
```

## Debug Commands

```bash
# Check job in Firestore
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document={job_id}

# View worker logs
gcloud run services logs read kitesforu-worker-script \
  --region=us-central1 --limit=100

# Check dead-letter queue
gcloud pubsub subscriptions pull workers-dead-letter-sub --limit=10
```

## Related Documentation

- [Reference Files](./reference/) - YAML specifications
- [Service Specs](./services/) - Service details
- [Operations](./operations/) - Deploy and debug commands
