# Worker Pipeline

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Pipeline Overview

The worker pipeline transforms a user's topic into a finished podcast through 5 sequential stages.

```
Initiator → Research Planner → Tools Executor → Script Generator → Audio Worker
```

## Stage 1: Initiator

**Service**: `kitesforu-worker-initiator`
**Topic**: `job-initiate`
**Timeout**: 120 seconds
**Budget**: 0%

### Responsibilities
- Validate job inputs
- Detect topic ambiguity
- Generate clarifier questions if needed
- Manage clarifier Q&A flow

### Input
```json
{
  "job_id": "uuid",
  "user_id": "clerk_user_id"
}
```

### Output
- Clarifier questions (optional)
- Publish to next stage

### Clarifier Questions
When a topic is ambiguous (e.g., "AI"), the initiator generates questions like:
- "What aspect of AI interests you most?"
- "Who is your target audience?"
- "What's your experience level with this topic?"

## Stage 2: Research Planner

**Service**: `kitesforu-worker-research-planner`
**Topic**: `job-research-planner`
**Timeout**: 180 seconds
**Budget**: 15%

### Responsibilities
- Analyze topic and clarifier answers
- Generate research plan
- Estimate costs and duration
- Wait for user approval

### Research Tasks Generated
1. **Web Search**: 2-3 searches on key aspects
2. **URL Extraction**: 1-2 authoritative sources

### Output Format
```json
{
  "tasks": [
    {
      "task_id": "t1",
      "task_type": "web_search",
      "query": "quantum computing basics 2024",
      "estimated_credits": 5,
      "estimated_duration_seconds": 30
    }
  ],
  "user_facing_summary": "I'll research the basics...",
  "total_estimated_credits": 15
}
```

### User Approval
- User can approve, edit (once), or cancel
- Approval triggers next stage

## Stage 3: Tools Executor

**Service**: `kitesforu-worker-tools`
**Topic**: `job-execute-tools`
**Timeout**: 300 seconds
**Budget**: 30%

### Responsibilities
- Execute approved research tasks
- Aggregate findings
- Handle API failures gracefully

### External APIs
- **Tavily**: Web search
- **Custom**: URL content extraction

### Error Handling
- Skip failed tasks, continue with available data
- Log partial results
- Never block on single task failure

## Stage 4: Script Generator

**Service**: `kitesforu-worker-script`
**Topic**: `job-script`
**Timeout**: 480 seconds
**Budget**: 40%

### Responsibilities
- Process research findings
- Generate podcast script
- Match requested style and duration
- Format as dialogue

### LLM Usage
- **Primary**: OpenAI GPT-4o
- **Fallback**: Anthropic Claude 3.5 Sonnet

### Script Format
```json
{
  "segments": [
    {
      "speaker": "host",
      "text": "Welcome to today's episode...",
      "duration_estimate": 15
    },
    {
      "speaker": "guest",
      "text": "Thanks for having me...",
      "duration_estimate": 10
    }
  ],
  "metadata": {
    "total_duration": 600,
    "word_count": 1500
  }
}
```

## Stage 5: Audio Worker

**Service**: `kitesforu-worker-audio`
**Topic**: `job-audio`
**Timeout**: 300 seconds
**Budget**: 25%

### Responsibilities
- TTS synthesis per segment
- Voice assignment (host vs guest)
- Audio mixing
- Waveform generation
- Upload to Cloud Storage
- Mark job complete

### TTS Providers
| Quality | Provider | Model |
|---------|----------|-------|
| Standard | OpenAI | tts-1 |
| HD | OpenAI | tts-1-hd |
| Premium | ElevenLabs | eleven_multilingual_v2 |

### Voice Assignment
- **Host**: Consistent voice (e.g., "alloy")
- **Guest**: Different voice (e.g., "echo")

### Output
- MP3 file uploaded to GCS
- Waveform PNG (optional)
- Job status updated to COMPLETED

## Error Handling

### Retry Policy
- Transient errors: NACK, Pub/Sub retries
- Max 5 retries with exponential backoff
- Min backoff: 10s, Max backoff: 600s

### Dead Letter Queue
- Failed messages go to `workers-dead-letter`
- Retained for 7 days
- Manual investigation required

### Failure Recovery
1. Check dead-letter queue
2. Identify root cause
3. Fix issue
4. Republish message or mark job failed

## Monitoring

### Key Metrics
- Processing time per stage
- Success/failure rates
- Credit usage per job
- LLM/TTS API latency

### Logging
- Structured JSON logs
- Cloud Logging integration
- Trace IDs for request tracking
