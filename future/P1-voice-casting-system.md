# P1: Voice Casting & Format Selection System

## Status: Foundation Laid, Full System Pending

### What's Done (Apr 7-8, 2026)
- Workers read `job_state["speakers"]` and `job_state["format"]` (PR #231)
- When provided, skip auto-detection entirely (user intent is law)
- Debate/comedy/discussion route correctly to dialogue format (PR #230, API #205)
- Debate keyword + pattern detection in content_format_planner (PR #230)
- Format override at streaming_worker:1073 removed (PR #231)

### What's Next

#### Phase 2: Persona API + Previews (3-4 days)
- New /v1/personas endpoint (list personas with metadata)
- Generate audio previews for all 21 personas
- Generate illustrated avatars (Flux Pro)
- Store in GCS public bucket

#### Phase 3: Schema + Format Selection (2-3 days)
- Add PodcastFormat enum to kitesforu-schemas
- Add SpeakerConfig model to kitesforu-schemas
- Extend CreateJobRequest with format + speakers fields
- Format cards in PlanSection.tsx

#### Phase 4: Persona Browser + Chat Casting (4-5 days)
- PersonaBrowser.tsx component (modal with search, filter, preview)
- Speaker slots in PlanSection.tsx (swap, preview, role/stance)
- Chat system prompt for casting questions
- speakers field in plan tool schema

#### Phase 5: Workers Integration (2-3 days)
- Per-speaker persona resolution (not just primary)
- Debate-specific prompt template
- Panel discussion (3+ speakers) support

## Code Paths
- Workers: src/workers/stages/combined/streaming_script_audio_worker.py (speakers[] reading)
- API: src/api/services/smart_create/executor.py (STYLE_NAME_MAP, plan execution)
- Schemas: kitesforu-schemas (PodcastFormat, SpeakerConfig, CreateJobRequest)
- Frontend: components/debug/ (persona browser), app/ (plan section)

## Priority: P1 | Impact: Transformative | Effort: 2-3 weeks total
