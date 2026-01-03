# LLM Provider Integration

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU integrates with multiple LLM and TTS providers through a Model Router that intelligently selects providers based on availability, performance, and cost.

## Providers

### LLM Providers

| Provider | Models Used | Use Cases |
|----------|-------------|-----------|
| OpenAI | gpt-4o, gpt-4o-mini | Research, script generation |
| Anthropic | claude-3-5-sonnet, claude-3-5-haiku | Research, script generation |
| Google | gemini-2.0-flash | Research, script generation |

### TTS Providers

| Provider | Voices | Use Cases |
|----------|--------|-----------|
| OpenAI TTS | alloy, echo, fable, onyx, nova, shimmer | Podcast audio |
| Google TTS | Various neural voices | Podcast audio |
| ElevenLabs | Custom voices | Premium podcast audio |

## API Keys

### Storage

All API keys stored in Secret Manager:

```bash
# List secrets
gcloud secrets list

# View specific key (careful!)
gcloud secrets versions access latest --secret=openai-api-key
```

### Required Secrets

| Secret Name | Provider |
|-------------|----------|
| openai-api-key | OpenAI |
| anthropic-api-key | Anthropic |
| google-ai-api-key | Google AI |
| elevenlabs-api-key | ElevenLabs |
| deepgram-api-key | Deepgram |

## Model Router

### Architecture

```
                    ┌─────────────────┐
                    │  Model Router   │
                    │   (Selection)   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   OpenAI     │    │  Anthropic   │    │   Google     │
│   Provider   │    │   Provider   │    │   Provider   │
└──────────────┘    └──────────────┘    └──────────────┘
```

### Selection Criteria

1. **Health Status**: Only select healthy providers
2. **Latency**: Prefer lower latency providers
3. **Cost**: Consider cost for budget optimization
4. **Capability**: Match model capabilities to task
5. **Quota**: Avoid providers near quota limits

### Configuration

```python
# src/workers/routing/config.py
MODEL_CONFIG = {
    "research": {
        "providers": ["openai", "anthropic", "google"],
        "models": {
            "openai": "gpt-4o",
            "anthropic": "claude-3-5-sonnet-20241022",
            "google": "gemini-2.0-flash"
        },
        "fallback_order": ["openai", "anthropic", "google"]
    },
    "script": {
        "providers": ["anthropic", "openai"],
        "models": {
            "anthropic": "claude-3-5-sonnet-20241022",
            "openai": "gpt-4o"
        },
        "fallback_order": ["anthropic", "openai"]
    }
}
```

## OpenAI Integration

### Setup

```python
import openai

client = openai.OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
```

### Usage

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a research assistant."},
        {"role": "user", "content": "Research topic: AI in healthcare"}
    ],
    temperature=0.7,
    max_tokens=4000
)

content = response.choices[0].message.content
```

### Error Handling

```python
try:
    response = client.chat.completions.create(...)
except openai.RateLimitError:
    # Fallback to another provider
    return await use_fallback_provider(...)
except openai.APIError as e:
    # Log and retry or fallback
    logger.error(f"OpenAI API error: {e}")
```

## Anthropic Integration

### Setup

```python
import anthropic

client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))
```

### Usage

```python
message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=4000,
    messages=[
        {"role": "user", "content": "Generate a podcast script about AI"}
    ]
)

content = message.content[0].text
```

### Streaming

```python
with client.messages.stream(
    model="claude-3-5-sonnet-20241022",
    max_tokens=4000,
    messages=[...]
) as stream:
    for text in stream.text_stream:
        yield text
```

## Google AI Integration

### Setup

```python
import google.generativeai as genai

genai.configure(api_key=os.environ.get("GOOGLE_AI_API_KEY"))
model = genai.GenerativeModel("gemini-2.0-flash")
```

### Usage

```python
response = model.generate_content(
    "Research topic: quantum computing basics",
    generation_config={
        "temperature": 0.7,
        "max_output_tokens": 4000
    }
)

content = response.text
```

## ElevenLabs TTS Integration

### Setup

```python
from elevenlabs import ElevenLabs

client = ElevenLabs(api_key=os.environ.get("ELEVENLABS_API_KEY"))
```

### Usage

```python
audio = client.generate(
    text="Welcome to today's episode...",
    voice="Rachel",
    model="eleven_multilingual_v2"
)

# Save audio
with open("output.mp3", "wb") as f:
    for chunk in audio:
        f.write(chunk)
```

### Available Voices

| Voice ID | Name | Style |
|----------|------|-------|
| 21m00Tcm4TlvDq8ikWAM | Rachel | Calm, professional |
| AZnzlk1XvdvUeBnXmlld | Domi | Energetic |
| EXAVITQu4vr4xnSDxMaL | Bella | Warm, friendly |

## Health Monitoring

### Provider Health Checks

```python
async def check_provider_health(provider: str) -> HealthStatus:
    """Check if provider is healthy"""
    try:
        start = time.time()

        if provider == "openai":
            # Quick API call
            client.models.list()
        elif provider == "anthropic":
            # Minimal request
            ...

        latency = (time.time() - start) * 1000

        return HealthStatus(
            provider=provider,
            status="healthy",
            latency_ms=latency,
            checked_at=datetime.utcnow()
        )
    except Exception as e:
        return HealthStatus(
            provider=provider,
            status="unavailable",
            error=str(e)
        )
```

### Health Storage

Health status stored in Firestore:

```python
# Collection: model_provider_health
{
    "provider": "openai",
    "status": "healthy",
    "latency_ms": 150,
    "error_rate": 0.01,
    "last_check": "2024-01-15T10:30:00Z"
}
```

## Quota Management

### Tracking Usage

```python
async def track_usage(provider: str, model: str, tokens: int):
    """Track token usage for quota management"""
    quota_ref = db.collection("model_quotas").document(f"{provider}_{model}")
    quota_ref.update({
        "tokens_used": firestore.Increment(tokens),
        "last_used": firestore.SERVER_TIMESTAMP
    })
```

### Quota Limits

| Provider | Daily Limit | Rate Limit |
|----------|-------------|------------|
| OpenAI | 1M tokens | 10K TPM |
| Anthropic | 1M tokens | 8K TPM |
| Google | 500K tokens | 60 RPM |

## Fallback Logic

```python
async def call_llm_with_fallback(
    task_type: str,
    prompt: str,
    job_id: str
) -> str:
    """Call LLM with automatic fallback"""
    config = MODEL_CONFIG[task_type]

    for provider in config["fallback_order"]:
        try:
            health = await get_provider_health(provider)
            if health.status != "healthy":
                continue

            model = config["models"][provider]
            response = await call_provider(provider, model, prompt)

            # Record successful routing
            await record_routing_decision(job_id, task_type, provider, model)

            return response

        except Exception as e:
            logger.warning(f"Provider {provider} failed: {e}")
            continue

    raise AllProvidersFailedError("No available providers")
```

## Cost Optimization

### Pricing (approximate)

| Provider | Model | Input | Output |
|----------|-------|-------|--------|
| OpenAI | gpt-4o | $5/1M | $15/1M |
| OpenAI | gpt-4o-mini | $0.15/1M | $0.60/1M |
| Anthropic | claude-3-5-sonnet | $3/1M | $15/1M |
| Anthropic | claude-3-5-haiku | $0.25/1M | $1.25/1M |

### Cost-Aware Routing

```python
def select_model_with_budget(task_type: str, budget_priority: bool = False):
    """Select model considering cost"""
    if budget_priority:
        # Prefer cheaper models
        return "gpt-4o-mini" or "claude-3-5-haiku"
    else:
        # Prefer quality
        return "gpt-4o" or "claude-3-5-sonnet"
```

## Related

- [MODEL_ROUTER.md](../services/workers/MODEL_ROUTER.md)
- [TROUBLESHOOTING.md](../operations/TROUBLESHOOTING.md)
