# Workers Service Overview

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Summary

The workers service consists of 5 Cloud Run services that process podcast jobs through a sequential pipeline. They're built from a single codebase but deployed as separate services, each handling a specific stage.

## Quick Reference

| Worker | Topic | Purpose |
|--------|-------|---------|
| kitesforu-worker-initiator | job-initiate | Initialize job, clarifier |
| kitesforu-worker-research-planner | job-research-planner | Plan research tasks |
| kitesforu-worker-tools | job-execute-tools | Execute research |
| kitesforu-worker-script | job-script | Generate script |
| kitesforu-worker-audio | job-audio | TTS and audio mixing |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pub/Sub Topics                            │
├──────────┬────────────────┬────────────┬──────────┬─────────────┤
│ initiate │ research-plan  │ exec-tools │  script  │    audio    │
└────┬─────┴───────┬────────┴─────┬──────┴────┬─────┴──────┬──────┘
     │             │              │           │            │
     v             v              v           v            v
┌─────────┐  ┌───────────┐  ┌──────────┐  ┌────────┐  ┌─────────┐
│Initiator│─>│ Planner   │─>│  Tools   │─>│ Script │─>│  Audio  │
└─────────┘  └───────────┘  └──────────┘  └────────┘  └─────────┘
     │             │              │           │            │
     └─────────────┴──────────────┴───────────┴────────────┘
                                  │
                                  v
                            ┌──────────┐
                            │ Firestore │
                            └──────────┘
```

## Project Structure

```
kitesforu-workers/
├── src/workers/
│   ├── main.py              # FastAPI app with stage routing
│   ├── handlers/
│   │   ├── initiator.py
│   │   ├── research_planner.py
│   │   ├── tools_executor.py
│   │   ├── script_generator.py
│   │   └── audio_worker.py
│   ├── services/
│   │   ├── firestore_service.py
│   │   ├── pubsub_service.py
│   │   ├── llm_service.py
│   │   ├── tts_service.py
│   │   └── storage_service.py
│   ├── routing/
│   │   ├── model_router.py
│   │   ├── provider_health.py
│   │   └── quota_manager.py
│   └── prompts/
│       ├── clarifier.py
│       ├── research_planner.py
│       └── script_generator.py
├── tests/
├── Dockerfile
└── requirements.txt
```

## How It Works

### Message Handling

Each worker exposes a POST endpoint that receives Pub/Sub push messages:

```python
@app.post("/")
async def handle_message(request: Request):
    # 1. Verify OIDC token
    # 2. Parse Pub/Sub message
    # 3. Route to appropriate handler based on WORKER_STAGE env var
    # 4. ACK on success, NACK on failure
```

### Stage Routing

The `WORKER_STAGE` environment variable determines which handler runs:

```python
handlers = {
    "job-initiate": initiator_handler,
    "job-research-planner": research_planner_handler,
    "job-execute-tools": tools_executor_handler,
    "job-script": script_generator_handler,
    "job-audio": audio_worker_handler,
}

handler = handlers[os.environ["WORKER_STAGE"]]
await handler(job_id, user_id)
```

## Resource Configuration

| Worker | Memory | CPU | Max Instances | Timeout |
|--------|--------|-----|---------------|---------|
| Initiator | 512Mi | 1 | 3 | 120s |
| Research Planner | 512Mi | 1 | 3 | 180s |
| Tools Executor | 1Gi | 1 | 5 | 300s |
| Script Generator | 512Mi | 1 | 3 | 480s |
| Audio Worker | 1Gi | 1 | 3 | 300s |

## External Integrations

### LLM Providers
- OpenAI (GPT-4o, GPT-4o-mini)
- Anthropic (Claude 3.5 Sonnet)
- Google (Gemini 1.5 Pro)

### TTS Providers
- OpenAI (tts-1, tts-1-hd)
- Google Cloud TTS
- ElevenLabs (premium)

### Research APIs
- Tavily (web search)

## Deployment

### Deploy All Workers
```bash
cd kitesforu-workers
./infra/deploy-workers.sh
```

### Check Status
```bash
gcloud run services list --region=us-central1 --filter="name~worker"
```

## Related Documentation

- [Pipeline Details](./PIPELINE.md) - Stage-by-stage breakdown
- [Model Router](./MODEL_ROUTER.md) - AI provider selection
