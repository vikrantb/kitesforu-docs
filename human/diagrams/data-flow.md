# Data Flow Diagram

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Request-to-Response Flow

Complete data flow from user request to podcast delivery.

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant API as API
    participant FS as Firestore
    participant PS as Pub/Sub
    participant W1 as Initiator
    participant W2 as Research Planner
    participant W3 as Tools Executor
    participant W4 as Script Generator
    participant W5 as Audio Worker
    participant GCS as Cloud Storage

    U->>FE: Create Podcast Request
    FE->>API: POST /v1/podcasts
    API->>FS: Create Job Document
    API->>PS: Publish to job-initiate
    API-->>FE: job_id + status

    PS->>W1: Push Message
    W1->>FS: Update Job Status
    W1->>PS: Publish to job-research-planner

    PS->>W2: Push Message
    W2->>W2: Generate Research Plan
    W2->>FS: Save Plan, Set awaiting_user_action

    Note over U,FE: User Reviews Research Plan

    U->>FE: Approve Plan
    FE->>API: POST /v1/podcasts/{id}/research-plan/approve
    API->>FS: Update Plan Status
    API->>PS: Publish to job-execute-tools

    PS->>W3: Push Message
    W3->>W3: Execute Web Searches
    W3->>FS: Save Research Results
    W3->>PS: Publish to job-script

    PS->>W4: Push Message
    W4->>W4: Generate Script via LLM
    W4->>FS: Save Script
    W4->>PS: Publish to job-audio

    PS->>W5: Push Message
    W5->>W5: TTS Synthesis
    W5->>W5: Audio Mixing
    W5->>GCS: Upload Audio File
    W5->>FS: Update Job COMPLETED

    FE->>API: GET /v1/podcasts/{id}/status (polling)
    API->>FS: Read Job
    API-->>FE: status: completed, audio_url

    U->>FE: Play/Download Audio
    FE->>GCS: Fetch Audio
```

## Stage Details

### 1. Job Creation
- Frontend submits topic, duration, style
- API validates input
- Job created in Firestore with status=QUEUED
- Message published to job-initiate topic

### 2. Initialization
- Initiator receives message
- May generate clarifier questions
- If clarifying, waits for user answers

### 3. Research Planning
- Research Planner generates 3-4 tasks
- Creates user-facing summary
- Estimates credits and duration
- Waits for user approval

### 4. Research Execution
- Tools Executor runs approved tasks
- Web searches via Tavily API
- URL content extraction
- Aggregates findings

### 5. Script Generation
- Script Generator uses LLM
- Creates host/guest dialogue
- Matches requested style
- Targets specified duration

### 6. Audio Production
- Audio Worker synthesizes speech
- Different voices for speakers
- Mixes segments
- Uploads to Cloud Storage

### 7. Completion
- Job marked COMPLETED
- Audio URL saved
- User can play/download
