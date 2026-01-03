# Worker Pipeline Details

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Stage 1: Initiator

**Topic**: `job-initiate`
**Timeout**: 120 seconds
**Budget**: 0% (no LLM cost)

### What It Does
1. Loads job from Firestore
2. Validates job inputs
3. Analyzes topic for ambiguity
4. Generates clarifier questions if needed

### Clarifier Flow
When a topic like "AI" is detected:

```
Topic "AI" → Ambiguity detected → Generate questions:
  - "What aspect of AI interests you?"
  - "Who is your target audience?"
  - "What depth of technical detail?"
```

User answers are collected via API, then processing continues.

### Output
- Updates job with clarifier questions
- Or publishes to `job-research-planner`

---

## Stage 2: Research Planner

**Topic**: `job-research-planner`
**Timeout**: 180 seconds
**Budget**: 15%

### What It Does
1. Combines topic with clarifier answers
2. Uses LLM to generate research plan
3. Creates 3-4 specific research tasks
4. Estimates credits and time per task
5. Waits for user approval

### Research Task Types

| Type | Description | Example |
|------|-------------|---------|
| web_search | Tavily search query | "quantum computing applications 2024" |
| url_extraction | Scrape specific URL | "https://arxiv.org/abs/..." |

### Plan Format
```json
{
  "tasks": [
    {
      "task_id": "t1",
      "task_type": "web_search",
      "query": "quantum computing basics explained",
      "estimated_credits": 5,
      "estimated_duration_seconds": 30,
      "priority": 1,
      "rationale": "Cover fundamental concepts"
    }
  ],
  "user_facing_summary": "I'll research the fundamentals of quantum computing, focusing on practical applications and recent developments.",
  "total_estimated_credits": 15
}
```

### User Approval
- Plan displayed in frontend
- User can: Approve, Edit (once), Cancel
- API publishes to next stage on approval

---

## Stage 3: Tools Executor

**Topic**: `job-execute-tools`
**Timeout**: 300 seconds
**Budget**: 30%

### What It Does
1. Loads approved research plan
2. Executes each enabled task
3. Aggregates results
4. Handles failures gracefully

### Execution Strategy
```
For each task:
  1. Check if enabled
  2. Execute (web_search or url_extraction)
  3. Store result
  4. Track credit usage
  5. Continue even if task fails
```

### Web Search (Tavily)
```python
results = tavily.search(
    query=task.query,
    max_results=5,
    include_answer=True
)
```

### URL Extraction
```python
content = await extract_url_content(task.url)
# Handles JavaScript rendering if needed
# Extracts main article content
```

### Output
- Research findings saved to Firestore
- Publishes to `job-script`

---

## Stage 4: Script Generator

**Topic**: `job-script`
**Timeout**: 480 seconds
**Budget**: 40%

### What It Does
1. Loads research findings
2. Constructs detailed prompt
3. Generates podcast script via LLM
4. Validates script structure

### Prompt Components
- Topic and clarifier context
- Research findings
- Style preference (Explainer, Storytelling, etc.)
- Target duration
- Audience level

### Script Structure
```json
{
  "segments": [
    {
      "speaker": "host",
      "text": "Welcome to our deep dive into quantum computing...",
      "type": "intro",
      "duration_estimate": 15
    },
    {
      "speaker": "guest",
      "text": "Thanks for having me. Let's start with the basics...",
      "type": "content",
      "duration_estimate": 45
    }
  ],
  "metadata": {
    "total_duration_estimate": 600,
    "word_count": 1500,
    "style": "Explainer"
  }
}
```

### LLM Selection
- **Primary**: GPT-4o (best quality)
- **Fallback**: Claude 3.5 Sonnet
- Model router handles selection

---

## Stage 5: Audio Worker

**Topic**: `job-audio`
**Timeout**: 300 seconds
**Budget**: 25%

### What It Does
1. Loads script
2. Synthesizes speech per segment
3. Assigns different voices
4. Mixes audio
5. Generates waveform
6. Uploads to GCS
7. Marks job complete

### Voice Assignment
```
Host → "alloy" voice
Guest → "echo" voice
```

### TTS Process
```python
for segment in script.segments:
    voice = "alloy" if segment.speaker == "host" else "echo"
    audio = await tts_service.synthesize(
        text=segment.text,
        voice=voice,
        model="tts-1"  # or "tts-1-hd" for HD quality
    )
    audio_segments.append(audio)
```

### Audio Mixing
```python
final_audio = mix_segments(audio_segments)
# Add brief pauses between speakers
# Normalize volume levels
# Export as MP3
```

### Upload
```python
gcs_url = await storage.upload(
    bucket="kitesforu-podcasts",
    path=f"{user_id}/{job_id}/podcast.mp3",
    data=final_audio
)
```

### Completion
```python
await firestore.update_job(job_id, {
    "status": "completed",
    "outputs.audio_url": gcs_url,
    "updated_at": datetime.utcnow()
})
```

---

## Error Handling

### Transient Errors
- Network timeouts
- Rate limits
- Temporary API failures

**Action**: NACK message, Pub/Sub retries

### Permanent Errors
- Invalid job data
- Content policy violations
- Unrecoverable API errors

**Action**: Mark job failed, send to dead-letter queue

### Dead Letter Queue
Messages that fail 5 times go to `workers-dead-letter`:
- Retained 7 days
- Manual investigation needed
- Use debug scripts to analyze
