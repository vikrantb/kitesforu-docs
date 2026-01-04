# Model Router

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

The Model Router intelligently selects AI providers based on health, quota availability, cost, and task requirements. It ensures reliable service even when individual providers have issues.

## How It Works

```
Request → Load Catalog → Check Health → Check Quota → Score Models → Select Best
                ↓                                            ↓
     (Google Sheets or CSV)                          (If fails)
                                                          ↓
                                                    Try Fallback
```

## Supported Providers

### LLM Providers
| Provider | Models | Primary Use |
|----------|--------|-------------|
| OpenAI | gpt-4o, gpt-4o-mini | Script generation |
| Anthropic | claude-3-5-sonnet, claude-3-haiku | Fallback, research |
| Google | gemini-1.5-pro | Secondary fallback |

### TTS Providers
| Provider | Models | Quality |
|----------|--------|---------|
| OpenAI | tts-1, tts-1-hd | Standard, HD |
| Google | cloud-tts | Standard |
| ElevenLabs | eleven_multilingual_v2 | Premium |

## Selection Criteria

### 1. Provider Health
```
HEALTHY (100%) → Fully operational
DEGRADED (50%) → Higher latency or error rate
UNAVAILABLE (0%) → Not responding
```

### 2. Quota Availability
```
AVAILABLE (100%) → Sufficient quota
WARNING (50%) → <20% remaining
CRITICAL (25%) → <5% remaining
EXHAUSTED (0%) → No quota
```

### 3. Task Fit
```
- Does model support this task type?
- Is input within context limits?
- Does quality match requirements?
```

### 4. Cost
```
Lower cost preferred when quality is equal
```

## Scoring Formula

```
score = (health_weight × health_score)
      + (quota_weight × quota_score)
      + (fit_weight × fit_score)
      - (cost_weight × normalized_cost)
```

Default weights:
- Health: 0.4
- Quota: 0.3
- Fit: 0.2
- Cost: 0.1

## Fallback Chains

### LLM Tasks
```
Primary: OpenAI GPT-4o
  ↓ (if unavailable)
Fallback 1: Anthropic Claude 3.5 Sonnet
  ↓ (if unavailable)
Fallback 2: Google Gemini 1.5 Pro
```

### TTS Tasks
```
Standard Quality:
  Primary: OpenAI tts-1
    ↓
  Fallback: Google Cloud TTS

HD Quality:
  Primary: OpenAI tts-1-hd
    ↓
  Fallback: Google Cloud TTS (HD)

Premium (if allowed):
  Primary: ElevenLabs
    ↓
  Fallback: OpenAI tts-1-hd
```

## Health Monitoring

### Health Checks
- Run every 60 seconds
- Check latency and error rate
- Update status in Firestore

### Circuit Breaker
- Opens after 5 consecutive failures
- Recovery attempt after 5 minutes
- Gradual traffic increase on recovery

### Firestore Collection
```
model_provider_health/{provider}
{
  "provider": "openai",
  "status": "healthy",
  "last_check": "2024-01-15T10:30:00Z",
  "latency_ms": 250,
  "error_rate": 0.01
}
```

## Quota Tracking

### Tracked Metrics
- Requests per minute
- Tokens per day
- Cost per period

### Quota Collection
```
model_quotas/{provider}_{model}
{
  "provider": "openai",
  "model_id": "gpt-4o",
  "tokens_used": 450000,
  "tokens_limit": 500000,
  "reset_at": "2024-01-16T00:00:00Z"
}
```

## Alerts

### Alert Types
| Type | Severity | Trigger |
|------|----------|---------|
| QUOTA_WARNING | Warning | <20% remaining |
| QUOTA_CRITICAL | Critical | <5% remaining |
| QUOTA_EXHAUSTED | Emergency | 0% remaining |
| PROVIDER_UNHEALTHY | Warning | Degraded status |
| PROVIDER_UNAVAILABLE | Critical | Not responding |
| FAILOVER_EXECUTED | Info | Switched provider |

### Alert Collection
```
model_router_alerts/{alert_id}
{
  "type": "quota_warning",
  "severity": "warning",
  "provider": "openai",
  "message": "OpenAI quota below 20%",
  "acknowledged": false,
  "created_at": "2024-01-15T10:30:00Z"
}
```

## Audit Trail

Every routing decision is logged:

```
routing_decisions/{decision_id}
{
  "job_id": "...",
  "task_type": "llm",
  "selected_provider": "openai",
  "selected_model": "gpt-4o",
  "score": 95.5,
  "selection_reason": "Best health and quota",
  "alternatives_considered": [
    {"provider": "anthropic", "model": "claude-3-5-sonnet", "score": 88.2}
  ],
  "decision_time_ms": 12,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

## Configuration

### Environment Variables
```env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_API_KEY=...
ELEVENLABS_API_KEY=...
```

### Health Check Settings
```python
HEALTH_CHECK_INTERVAL = 60  # seconds
CIRCUIT_BREAKER_THRESHOLD = 5  # failures
CIRCUIT_BREAKER_RECOVERY = 300  # seconds
```

## Debugging

### Check Provider Health
```python
from google.cloud import firestore
db = firestore.Client()

for doc in db.collection("model_provider_health").stream():
    print(f"{doc.id}: {doc.to_dict()['status']}")
```

### Check Recent Routing Decisions
```bash
gcloud firestore documents list \
  --collection=routing_decisions \
  --order-by="timestamp desc" \
  --limit=10
```

## Related Documentation

- [Model Catalog](./MODEL_CATALOG.md) - Model catalog management and Google Sheets integration
- [Pipeline](./PIPELINE.md) - Worker pipeline overview
