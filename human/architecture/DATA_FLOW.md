# Data Flow

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Request Lifecycle

This document describes the complete flow of a podcast creation request.

## Phase 1: Job Creation

```
User → Frontend → API → Firestore → Pub/Sub
```

### Steps

1. **User Action**: User fills out podcast creation form
   - Topic (required)
   - Duration (0.5 - 180 minutes)
   - Style (Explainer, Storytelling, Interview, Motivational)
   - Audience (optional)

2. **Frontend Submission**
   - Clerk token attached to request
   - POST to `/v1/podcasts`

3. **API Processing**
   - Validate JWT token
   - Calculate credit cost
   - Verify user has sufficient credits
   - Create job document in Firestore
   - Deduct credits
   - Publish message to `job-initiate` topic

4. **Response to User**
   - Return `job_id`
   - Frontend starts polling for status

## Phase 2: Research Planning

```
Pub/Sub → Initiator → Research Planner → Wait for User
```

### Initiator Worker
1. Receive message from `job-initiate`
2. Load job from Firestore
3. Analyze topic for ambiguity
4. If ambiguous, generate clarifier questions
5. Wait for user answers (if needed)
6. Publish to `job-research-planner`

### Research Planner Worker
1. Receive message from `job-research-planner`
2. Analyze topic and any clarifier answers
3. Generate research plan (3-4 tasks)
4. Estimate credits and duration
5. Save plan to Firestore
6. Set `awaiting_user_action = "research_plan_approval"`
7. **PAUSE** - Wait for user approval

### User Approval
- User views research plan in frontend
- Options: Approve, Edit (max 1), Cancel
- On approval: API publishes to `job-execute-tools`

## Phase 3: Research Execution

```
Pub/Sub → Tools Executor → Firestore
```

### Tools Executor Worker
1. Receive message from `job-execute-tools`
2. Load approved research plan
3. Execute each task:
   - **Web Search**: Use Tavily API
   - **URL Extraction**: Scrape content from URLs
4. Aggregate results
5. Save research results to Firestore
6. Publish to `job-script`

## Phase 4: Script Generation

```
Pub/Sub → Script Generator → Firestore
```

### Script Generator Worker
1. Receive message from `job-script`
2. Load research results
3. Construct LLM prompt with:
   - Topic
   - Research findings
   - Style preference
   - Target duration
   - Audience
4. Generate podcast script via LLM
5. Validate script structure
6. Save script to Firestore
7. Publish to `job-audio`

## Phase 5: Audio Production

```
Pub/Sub → Audio Worker → GCS → Firestore
```

### Audio Worker
1. Receive message from `job-audio`
2. Load script
3. For each dialogue segment:
   - Select appropriate voice
   - Call TTS API
   - Collect audio bytes
4. Mix audio segments
5. Generate waveform visualization
6. Upload to Cloud Storage
7. Update job document:
   - Set `audio_url`
   - Set `status = COMPLETED`

## Phase 6: Delivery

```
Frontend (polling) ← API ← Firestore
```

### Frontend Polling
1. Poll `/v1/podcasts/{job_id}/status` every 3 seconds
2. When `status = completed`:
   - Fetch result from `/v1/podcasts/{job_id}/result`
   - Display audio player
   - Enable download

## Error Handling

### Transient Errors
- Worker NACKs message
- Pub/Sub retries with backoff
- Max 5 retries

### Permanent Errors
- Job marked as FAILED
- Error details stored
- Message sent to dead-letter queue
- Credits refunded

## Data Storage

| Data | Storage | Retention |
|------|---------|-----------|
| Job metadata | Firestore | Indefinite |
| Research results | Firestore | Indefinite |
| Script | Firestore | Indefinite |
| Audio file | GCS | 2 days (auto-delete) |
| Credit transactions | Firestore | Indefinite |
