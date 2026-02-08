# KitesForU Architecture Overview
<!-- last_verified: 2026-02-07 -->

## System Map
KitesForU is an AI-powered audio content platform. It transforms topics, resumes, and documents into professional audio courses, podcasts, and interview prep materials.

## Repositories

### kitesforu-frontend
- **Tech**: Next.js (App Router), TypeScript, Tailwind CSS, Radix UI
- **Location**: `kitesforu-frontend/`
- **Deploy**: Cloud Run via GH Actions CI (auto on main push)
- **Package manager**: pnpm (CRITICAL: Docker uses `pnpm install --frozen-lockfile`)
- **Key directories**: `app/` (pages), `components/` (UI), `hooks/`, `lib/`, `e2e/` (Playwright tests), `__tests__/` (Jest)
- **Key routes**: `/create` (podcast), `/courses/create` (course), `/interview-prep/create` (interview prep), `/debug/[jobId]` (podcast debug), `/debug/course/[courseId]` (course debug), `/admin/model-router` (model admin)

### kitesforu-api
- **Tech**: Python, FastAPI, Pydantic
- **Location**: `kitesforu-api/`
- **Deploy**: Cloud Run via GH Actions CI (auto on main push)
- **Key directories**: `src/api/routes/` (endpoints), `src/api/services/` (business logic), `tests/`
- **Key services**: credit_manager.py, stripe.py, model_router, provider_health_monitor, extraction/prompts/
- **Auth**: Clerk JWT
- **Key routes**: `/v1/podcasts/`, `/v1/courses/`, `/v1/interview-prep/`, `/v1/me/credits`, `/v1/admin/`

### kitesforu-workers (Podcast Workers)
- **Tech**: Python
- **Location**: `kitesforu-workers/`
- **Deploy**: Cloud Run via GH Actions (11 separate services, auto on main push)
- **Key directories**: `src/workers/stages/` (pipeline stages), `src/workers/routing/` (model router), `src/workers/prompting/` (prompt templates), `config/` (model_catalog.csv)
- **Pipeline**: initiate → plan → research → execute-tools → assimilate → script → audio
- **Prompts**: Hard-coded Python dicts in `src/workers/prompting/templates.py`
- **Model router**: Full routing system in `src/workers/routing/` (router.py, health.py, quota.py, failover.py, alerts.py)

### kitesforu-course-workers
- **Tech**: Python
- **Location**: `kitesforu-course-workers/`
- **Deploy**: Cloud Run via Cloud Build (LEGACY — should migrate to GH Actions)
- **Key directories**: `src/workers/stages/` (initiator, planner, syllabus, orchestrator), `src/workers/prompts/` (YAML prompt files)
- **Pipeline**: initiate → [planner] → syllabus → orchestrate → (triggers podcast workers per episode)
- **Prompts**: YAML files loaded by `src/workers/prompts/loader.py`
- **Deploy command**: `gcloud builds submit --config=cloudbuild.yaml --project=kitesforu-dev`

### kitesforu-schemas
- **Tech**: Python package
- **Location**: `kitesforu-schemas/`
- **Deploy**: GH Actions → Artifact Registry (auto version check + publish)
- **Key exports**: `format_duration()`, `SupportedLanguage` (Literal type — use cast(), not direct str), duration utilities
- **Consumed by**: API, workers, course-workers (shared dependency)

### kitesforu-infrastructure
- **Tech**: Terraform, Bash
- **Location**: `kitesforu-infrastructure/`
- **Deploy**: GH Actions (plan on PR, apply on main)
- **Manages**: Cloud Run services, Firestore, networking, IAM, Pub/Sub, Artifact Registry
- **Structure**: `terraform/` with 27 subdirectories

### Other Repos
- `kitesforu-docs/` — Project documentation
- `kitesforu-qa/` — Quality assurance
- `kitesforu-business/` — Business strategy docs
- `kitesforu-support/` — Customer support

## Inter-Service Communication
- **API → Workers**: Via Pub/Sub topics (course-initiate, course-syllabus, course-orchestrate, course-planner)
- **Workers → Workers**: Via Pub/Sub topics
- **Orchestrator → Podcast API**: HTTP calls to create podcast jobs per episode
- **All → Firestore**: Shared state via Firestore collections
- **Frontend → API**: REST via fetch/SWR

## Environment
- **GCP Project**: kitesforu-dev
- **Region**: us-central1
- **14+ Cloud Run services** running in production
- **Auth**: Clerk (frontend), JWT verification (API)
- **Payments**: Stripe (subscriptions + PAYG)
- **Domain**: beta.kitesforu.com

## Critical Gotchas
1. Frontend MUST use pnpm (not npm) — Docker build fails with stale pnpm-lock.yaml
2. SupportedLanguage is a Literal type — use `cast(SupportedLanguage, value)` not direct assignment
3. Duration: seconds internally, minutes (float) at API boundary
4. Course workers Cloud Build needs explicit `cd` to correct directory + `SHORT_SHA` substitution
5. Always verify Cloud Run revisions after deploy: `gcloud run revisions list --service=<name> --region=us-central1`
6. pipeline_config in Firestore toggles features at RUNTIME (planner_enabled, etc.)
7. Code on main != deployed — always verify CI passed AND new revision exists
