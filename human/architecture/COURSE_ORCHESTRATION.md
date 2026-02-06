# Course Orchestration Architecture

This document describes the course creation and orchestration system that leverages the existing podcast pipeline.

## Overview

The course system orchestrates content creation by calling the podcast API for each module, ensuring any improvements to the podcast pipeline automatically benefit courses.

```
┌─────────────────────────────────────────────────────────────────┐
│                    kitesforu-course-workers                     │
│                    (Separate Repository)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐    ┌─────────────────────────┐      │
│  │ CourseInitiatorWorker │───►│ CoursePlannerWorker     │      │
│  │ (Validate & create)   │    │ (Optional, runtime cfg) │      │
│  └──────────────────────┘    └───────────┬─────────────┘      │
│          │ (if planner disabled)          │ (if planner enabled)│
│          ▼                               ▼                     │
│  ┌──────────────────────────────┐                              │
│  │ CourseSyllabusWorker         │                              │
│  │ (LLM curriculum generation)  │                              │
│  └──────────────┬───────────────┘                              │
│                 ▼                                               │
│  ┌──────────────────────────────┐                              │
│  │ CourseModuleOrchestratorWorker│                              │
│  │ (Creates podcasts via API)   │                              │
│  └──────────────┬───────────────┘                              │
│                                              │                  │
│              ┌───────────────┬───────────────┼───────────────┐ │
│              ▼               ▼               ▼               ▼ │
│         ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐│
│         │Module 1│     │Module 2│     │Module 3│     │Module N││
│         │Podcast │     │Podcast │     │Podcast │     │Podcast ││
│         └───┬────┘     └───┬────┘     └───┬────┘     └───┬────┘│
│             │              │               │               │    │
└─────────────┼──────────────┼───────────────┼───────────────┼────┘
              │              │               │               │
              ▼              ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                       kitesforu-api                             │
│                                                                 │
│  POST /v1/podcasts  →  Existing Podcast Pipeline                │
│  GET /v1/podcasts/{id}/status  →  Job status polling            │
│  GET /v1/podcasts/{id}/result  →  Audio URL + script            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
              │              │               │               │
              ▼              ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    kitesforu-workers                            │
│  (Existing - NO CHANGES NEEDED)                                 │
│                                                                 │
│  InitiatorWorker → ResearchPlannerWorker → ToolsWorker          │
│       → ScriptWorker → AudioWorker                              │
│                                                                 │
│  Capabilities: Research, PDF reading, Script generation,        │
│                Multi-language TTS, Tier-based quality           │
└─────────────────────────────────────────────────────────────────┘
```

## Design Principles

### Win-Win Architecture

| Approach | Duplicates Code | Benefits from Podcast Improvements | Maintenance |
|----------|-----------------|-----------------------------------|-------------|
| ❌ Old: Generate content directly | Yes | No | 2x maintenance |
| ✅ New: Orchestrate via API calls | No | Yes, automatically | Single codebase |

**Benefits**:
1. Any research/script/audio improvements benefit both podcasts AND courses
2. Single source of truth for content generation
3. Course workers focus on orchestration logic only
4. Easier to maintain and evolve
5. Natural separation of concerns

## Pipeline Stages

### Stage 1: Course Initiation

**Topic**: `course-initiate`
**Worker**: `CourseInitiatorWorker`

**Responsibilities**:
- Validate input parameters
- Create course document in Firestore
- Route to planner or syllabus based on runtime config

**Input**:
```json
{
    "course_id": "uuid",
    "user_id": "user-id",
    "topic": "Introduction to AI",
    "num_modules": 5,
    "module_duration_min": 10,
    "style": "Explainer",
    "language": "en-US",
    "audience": "Beginners",
    "context": "Optional context..."
}
```

### Stage 1.5: Workflow Planning (Optional)

**Topic**: `course-planner`
**Worker**: `CoursePlannerWorker`

**Controlled by**: Firestore `pipeline_config/course_workers` → `planner_enabled`

**Responsibilities**:
- Determine workflow plan using deterministic routing (or LLM in Phase 2)
- Validate plan against guardrails (step limits, credit caps, dependency checks)
- Execute plan by publishing to downstream workers

When `planner_enabled=false` (default), courses skip this stage and go directly from initiate to syllabus. Config changes take effect within 60 seconds (TTL cache), no redeploy needed.

### Stage 2: Syllabus Generation

**Topic**: `course-syllabus`
**Worker**: `CourseSyllabusWorker`

**Responsibilities**:
- Use LLM to generate structured curriculum
- Ensure modules are incremental (concept 1 → concept 2 → etc.)
- Each module has unique, specific focus
- Save curriculum to Firestore

**Curriculum Output**:
```json
{
    "course_summary": "Overview of the course",
    "target_audience": "Who this is for",
    "prerequisites": ["Prerequisite 1"],
    "learning_outcomes": ["Outcome 1"],
    "modules": [
        {
            "module_number": 1,
            "title": "Module Title",
            "description": "What this module covers",
            "objectives": ["Objective 1"],
            "key_topics": ["Topic A", "Topic B"],
            "builds_on": null,
            "prepares_for": 2
        }
    ]
}
```

### Stage 3: Module Orchestration

**Topic**: `course-orchestrate`
**Worker**: `CourseModuleOrchestratorWorker`

**Responsibilities**:
- Read curriculum from Firestore
- For each module, call `POST /v1/podcasts`
- Poll job status until completion
- Update module with results (audio_url, script)
- Track overall course progress

**Processing**:
- Processes modules in parallel batches (default: 3 concurrent)
- Each module creates a podcast job via API
- Polls for completion with configurable timeout
- Aggregates results and updates course status

## Firestore Schema

### courses/{course_id}

```javascript
{
  // Identity
  course_id: "uuid",
  user_id: "user-id",
  
  // Inputs
  topic: "Course topic",
  num_modules: 5,
  module_duration_min: 10,
  style: "Explainer",
  language: "en-US",
  audience: "Target audience",
  context: "Additional context",
  
  // Status
  status: "completed", // draft | syllabus_generation | curriculum_ready | generating | completed | partial | failed
  
  // Curriculum (generated by CourseSyllabusWorker)
  curriculum: {
    course_summary: "...",
    modules: [
      {
        module_number: 1,
        title: "...",
        status: "completed",
        podcast_job_id: "job-uuid",
        audio_url: "gs://...",
        script: {...}
      }
    ]
  },
  
  // Progress
  progress: {
    completed_modules: 5,
    failed_modules: 0,
    total_modules: 5,
    percentage: 100
  },
  
  // Timestamps
  created_at: "ISO timestamp",
  updated_at: "ISO timestamp",
  completed_at: "ISO timestamp"
}
```

## API Integration

### Service Account Authentication

Course workers use service account with user impersonation:

```python
headers = {
    "Authorization": f"Bearer {service_account_token}",
    "X-On-Behalf-Of": user_id
}
```

### Podcast API Calls

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/podcasts` | POST | Create podcast job for module |
| `/v1/podcasts/{id}/status` | GET | Poll job status |
| `/v1/podcasts/{id}/result` | GET | Get completed audio URL |

## Infrastructure

### Pub/Sub Topics

| Topic | Worker | Description |
|-------|--------|-------------|
| `course-initiate` | CourseInitiatorWorker | Validates input, creates course doc |
| `course-planner` | CoursePlannerWorker | Optional workflow planning |
| `course-syllabus` | CourseSyllabusWorker | LLM generates curriculum |
| `course-orchestrate` | CourseModuleOrchestratorWorker | Creates podcast jobs via API |

### Cloud Run Services

- `course-initiate-worker`: Low resource (512Mi), high concurrency
- `course-planner-worker`: Low resource (512Mi), long timeout (1 hour)
- `course-syllabus-worker`: Medium resource (1Gi), LLM calls
- `course-orchestrate-worker`: Higher resource, long timeout (1 hour)

### Runtime Pipeline Config

Stored in Firestore `pipeline_config/course_workers`:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `planner_enabled` | bool | `false` | Route initiate → planner instead of → syllabus |
| `planner_mode` | string | `deterministic_only` | Routing strategy |
| `max_workflow_steps` | int | `5` | Maximum steps in a workflow plan |
| `max_credits_per_workflow` | float | `50.0` | Maximum credits per workflow |

## Repository Structure

```
kitesforu-course-workers/
├── src/
│   ├── workers/
│   │   ├── base.py              # BaseWorker class
│   │   ├── server.py            # FastAPI Pub/Sub dispatcher
│   │   ├── exceptions.py        # Custom exceptions
│   │   ├── config/              # Runtime pipeline config
│   │   │   ├── pipeline_config.py  # Firestore-backed config with TTL cache
│   │   │   └── stage_resolver.py   # Routing abstraction
│   │   └── stages/
│   │       ├── initiator/       # CourseInitiatorWorker
│   │       ├── planner/         # CoursePlannerWorker (optional)
│   │       ├── syllabus/        # CourseSyllabusWorker
│   │       └── orchestrator/    # CourseModuleOrchestratorWorker
│   └── services/
│       └── podcast_api_client.py # HTTP client for API
├── scripts/
│   └── seed_pipeline_config.py  # Seed Firestore config doc
├── tests/
├── Dockerfile
├── cloudbuild.yaml
└── requirements.txt
```

## Related Documentation

- [Worker Pipeline](./WORKER_PIPELINE.md) - Podcast worker pipeline
- [Data Flow](./DATA_FLOW.md) - System data flow
- [TTS Voice System](./tts-voice-system.md) - Audio generation
