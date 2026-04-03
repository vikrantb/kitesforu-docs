> **Research Date**: April 2026. Verify current API documentation before acting on specific parameters or pricing.

# Pipeline Performance Optimization — Master Plan

**Date**: 2026-04-03
**Scope**: Podcast pipeline + Car Mode
**Baseline Job**: `78c4b6b1` — 211.8s (3.5 min) for a 5-min podcast
**Target**: <60s end-to-end

---

## Table of Contents

1. [Job Forensics — What's Actually Slow](#1-job-forensics)
2. [Critical Bugs (Fix Immediately)](#2-critical-bugs)
3. [P0 — Infrastructure (Hours, Biggest Impact)](#3-p0-infrastructure)
4. [P1 — Provider API Optimizations (Days)](#4-p1-provider-api)
5. [P2 — Pipeline Architecture (Weeks)](#5-p2-architecture)
6. [P3 — Advanced Optimizations (Future)](#6-p3-advanced)
7. [Complete File Inventory](#7-file-inventory)
8. [Implementation Roadmap](#8-roadmap)
9. [Projected Impact Summary](#9-impact-summary)

---

## 1. Job Forensics

### Job 78c4b6b1 Timeline

```
04:27:59 ─── Job Created ───────────────────────────────────
04:28:04 │ Initiate (5s setup)
04:28:04 ├─ Research Planner ─── 25.2s ── (2 LLM calls: discovery 8s + tasks 9.5s)
04:28:44 │ ~~~ 13.3s Pub/Sub gap ~~~ (cold start)
04:28:57 ├─ Execute Tools ───── 9.3s ─── (5 web searches in parallel)
04:29:06 │ ~~~ 14.5s Pub/Sub gap ~~~ (cold start)
04:29:21 ├─ Assimilator ────── 31.7s ── (single LLM synthesis call)
04:29:53 │ ~~~ 14.2s Pub/Sub gap ~~~ (cold start)
04:30:07 ├─ Script ──────────── 17.8s ── (LLM: 5391 tokens, 10 items)
04:30:24 │ ~~~ 11.9s Pub/Sub gap ~~~ (cold start)
04:30:36 ├─ Audio ──────────── 54.6s ── (ElevenLabs QUOTA=0 → Google fallback)
04:31:31 ─── Job Complete ──────────────────────────────────
```

### Where Time Goes

| Category | Time | % |
|----------|------|---|
| Audio stage | 54.6s | 26% |
| Pub/Sub gaps (4 cold starts) | 53.9s | 25% |
| Assimilator LLM | 31.7s | 15% |
| Research Planner (2 LLM calls) | 25.2s | 12% |
| Script LLM | 17.8s | 8% |
| Execute Tools | 9.3s | 4% |
| Other overhead | 19.3s | 9% |

### Cost: $0.164 total ($0.061 LLM + $0.102 TTS)

---

## 2. Critical Bugs (Fix Immediately)

### BUG-1: Inworld max_chunk_size exceeds API limit

**File**: `kitesforu-workers/src/workers/stages/audio/providers/inworld_provider.py:99`
**Problem**: `max_chunk_size = 5000` but Inworld API limit is **2,000 characters**
**Impact**: Likely truncated audio output on long segments
**Fix**:
```python
# Line 99: Change from 5000 to 1900
max_chunk_size = 1900  # Stays safely under 2,000 char API limit
```

### BUG-2: Inworld timestampType adds unused latency

**File**: `kitesforu-workers/src/workers/stages/audio/providers/inworld_provider.py:218`
**Problem**: `"timestampType": "WORD"` adds ~100ms per API call, but timestamps are never used
**Impact**: ~5s wasted on a 50-chunk podcast
**Fix**: Remove `"timestampType": "WORD"` from payload

### BUG-3: Inworld temperature range too wide

**File**: `kitesforu-workers/src/workers/stages/audio/providers/inworld_provider.py:151`
**Problem**: `max(0.1, min(1.0, request.intensity))` — values below 0.8 risk broken/monotone generation
**Fix**:
```python
# Better range per Inworld docs
params["temperature"] = max(0.8, min(1.3, 0.8 + (request.intensity * 0.5))
```

---

## 3. P0 — Infrastructure (Hours, Biggest Impact)

### P0-1: Cloud Run min_instance_count = 1

**Impact**: Eliminate ~54s of cold starts (25% of total time)
**Cost**: ~$80/month for all 7 workers, or ~$35/month for critical path only
**Effort**: 30 minutes (Terraform change)

**File**: `kitesforu-infrastructure/terraform/cloud_run_workers.tf`

Set `min_instance_count = 1` for critical-path workers:
- Lines 144: `worker_initiator` — 0 → 1
- Lines 281: `worker_research_planner` — 0 → 1
- Lines 420: `worker_tools` — 0 → 1
- Lines 568: `worker_research_assimilator` — 0 → 1
- Lines 712: `worker_script` — 0 → 1
- Lines 849: `worker_audio` — 0 → 1

```hcl
scaling {
  min_instance_count = 1   # was 0
  max_instance_count = 5
}
```

### P0-2: Cloud Run startup_cpu_boost = true

**Impact**: 30-50% faster remaining cold starts (free!)
**Cost**: $0
**Effort**: 30 minutes (Terraform change)

Add to ALL Cloud Run service templates:
```hcl
template {
  # Add this line to each service
  startup_cpu_boost = true

  containers { ... }
  scaling { ... }
}
```

**Files**: All `cloud_run_*.tf` files in `kitesforu-infrastructure/terraform/`

### P0-3: uvloop — 25-35% async performance boost

**Impact**: 25-35% faster on all async I/O operations
**Cost**: $0
**Effort**: 15 minutes

**File**: `kitesforu-workers/requirements.txt` — add `uvloop`
**File**: `kitesforu-workers/Dockerfile:38` — change CMD:
```dockerfile
CMD ["sh", "-c", "uvicorn workers.server:app --host 0.0.0.0 --port ${PORT} --loop uvloop"]
```

### P0-4: Persistent httpx clients (all TTS providers)

**Impact**: Save ~50-150ms per API call (2.5-7.5s total per podcast)
**Cost**: $0
**Effort**: 2 hours

Current (creates new client per call):
```python
async with httpx.AsyncClient(timeout=90.0) as client:
    response = await client.post(url, ...)
```

Fix (persistent singleton with HTTP/2):
```python
class InworldTTSProvider(BaseTTSProvider):
    def __init__(self):
        self._api_key = os.getenv("INWORLD_API_KEY", "")
        self._client = httpx.AsyncClient(
            timeout=90.0,
            http2=True,
            limits=httpx.Limits(
                max_connections=20,
                max_keepalive_connections=10,
                keepalive_expiry=30.0,
            ),
            headers={
                "Authorization": f"Basic {self._api_key}",
                "Content-Type": "application/json",
            },
        )
```

**Files** (same pattern for each):
- `providers/inworld_provider.py:226` — `async with httpx.AsyncClient(timeout=90.0)`
- `providers/elevenlabs_provider.py:279` — `async with httpx.AsyncClient(timeout=90.0)`
- `providers/elevenlabs_v3.py` — same pattern

### P0-5: Disable Pub/Sub publisher batching

**Impact**: Save 50-100ms per stage transition (350-700ms total)
**Cost**: $0
**Effort**: 15 minutes

**File**: `kitesforu-workers/src/workers/server.py:56`

Current:
```python
pubsub_publisher = pubsub_v1.PublisherClient()  # default batching
```

Fix:
```python
from google.cloud.pubsub_v1.types import BatchSettings

pubsub_publisher = pubsub_v1.PublisherClient(
    batch_settings=BatchSettings(
        max_messages=1,   # send immediately, no batching
        max_latency=0,    # zero wait
    ),
)
```

### P0-6: Docker optimization

**Impact**: 200-500ms faster cold starts
**Cost**: $0
**Effort**: 1 hour

**File**: `kitesforu-workers/Dockerfile`

```dockerfile
FROM python:3.12-slim
# ^^^ 3.12 has 15-20% faster startup than 3.11

ARG PIP_EXTRA_INDEX_URL=""

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir \
    ${PIP_EXTRA_INDEX_URL:+--extra-index-url "$PIP_EXTRA_INDEX_URL"} \
    -r requirements.txt

COPY src/ ./src
COPY config/ ./config
COPY config/ ./src/config

# Pre-compile bytecode (saves 50-200ms on cold start)
RUN python -m compileall -q -j 0 /app

ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app/src
ENV PORT=8080

EXPOSE 8080
CMD ["sh", "-c", "uvicorn workers.server:app --host 0.0.0.0 --port ${PORT} --loop uvloop"]
```

### P0 Combined Impact

| Optimization | Time Saved | Cost |
|-------------|-----------|------|
| min_instances=1 | **54s** | +$35-80/mo |
| startup_cpu_boost | **3-5s** | Free |
| uvloop | **5-10s** | Free |
| Persistent httpx | **2.5-7.5s** | Free |
| Pub/Sub no-batch | **0.5s** | Free |
| Docker + Python 3.12 | **0.5s** | Free |
| **P0 Total** | **~65-78s** | **+$35-80/mo** |

**Baseline 211.8s → ~135-150s after P0**

---

## 4. P1 — Provider API Optimizations (Days)

### P1-1: LLM System/User Message Separation (Enables Prompt Caching)

**Impact**: 50-90% input token cost reduction + 2-5s latency reduction
**Effort**: 3-5 days

The `load_prompt_parts()` function already returns `{system, user}` separately. But `generate_text_with_failover()` takes a single `prompt: str`, collapsing them.

**Step 1**: Change `generate_text_with_failover()` signature

**File**: `kitesforu-workers/src/workers/routing/integration.py:321`
```python
async def generate_text_with_failover(
    prompt: str,                          # Keep for backward compat
    job_id: str,
    task: str = "podcast_scripting",
    system_prompt: str | None = None,     # NEW
    user_prompt: str | None = None,       # NEW
    temperature: float = 0.7,
    max_tokens: int = 8000,
    budget_remaining: float = 1.0,
    response_format: dict | None = None,  # NEW: for JSON mode
) -> ProviderResponse:
```

**Step 2**: Update OpenAI provider to use system message

**File**: `kitesforu-workers/src/workers/routing/providers/openai_provider.py:79-84`
```python
# Before
messages=[{"role": "user", "content": prompt}]

# After
messages = []
if system_prompt:
    messages.append({"role": "system", "content": system_prompt})
messages.append({"role": "user", "content": user_prompt or prompt})
```

**Step 3**: Update Anthropic provider with cache_control

**File**: `kitesforu-workers/src/workers/routing/providers/anthropic_provider.py:148-153`
```python
# Before
messages=[{"role": "user", "content": prompt}]

# After
system_blocks = None
if system_prompt:
    system_blocks = [{
        "type": "text",
        "text": system_prompt,
        "cache_control": {"type": "ephemeral"},  # 90% discount on cache hit
    }]

response = await self.client.messages.create(
    model=model_id,
    system=system_blocks,
    messages=[{"role": "user", "content": user_prompt or prompt}],
    temperature=temperature,
    max_tokens=max_tokens,
    **kwargs,
)
```

**Step 4**: Update all callers (assimilator, script, planner)

Each caller already uses `load_prompt_parts()` — just pass `system` and `user` separately:
- `stages/assimilator/worker.py:341-362`
- `stages/script/worker.py:509+`
- `stages/planner/prompt_builder.py:159`

### P1-2: OpenAI reasoning_effort per stage

**Impact**: 2-10x LLM latency difference
**Effort**: 1 day

Add `reasoning_effort` to `generate_text()`:

**File**: `kitesforu-workers/src/workers/routing/providers/openai_provider.py:79`
```python
response = await self.client.chat.completions.create(
    model=model_id,
    messages=messages,
    temperature=temperature,
    max_tokens=max_tokens,
    reasoning_effort=kwargs.pop("reasoning_effort", None),  # NEW
    **kwargs,
)
```

Per-stage configuration:
| Stage | reasoning_effort | Why |
|-------|-----------------|-----|
| research_planner (task gen) | `"low"` | Simple query generation |
| assimilator (synthesis) | `"medium"` | Needs reasoning for dedup/themes |
| script (dialogue) | `"none"` or `"low"` | Creative writing, not analytical |

### P1-3: JSON mode for structured outputs

**Impact**: Eliminates parse failures, reduces output tokens 20-30%
**Effort**: 1 day

**File**: `kitesforu-workers/src/workers/routing/providers/openai_provider.py:79`
```python
response = await self.client.chat.completions.create(
    model=model_id,
    messages=messages,
    temperature=temperature,
    max_tokens=max_tokens,
    response_format=kwargs.pop("response_format", None),  # NEW
    **kwargs,
)
```

Callers pass `response_format={"type": "json_object"}`:
- `stages/assimilator/worker.py` — synthesis output is JSON
- `stages/script/worker.py` — dialogue output is JSON
- `stages/planner/` — research tasks are JSON

### P1-4: TTS pre-flight quota check (ElevenLabs)

**Impact**: Save 10-15s when quota is exhausted
**Effort**: 1 day

**File**: `kitesforu-workers/src/workers/stages/audio/providers/elevenlabs_provider.py`

Add at class level:
```python
async def check_quota(self, chars_needed: int) -> bool:
    """Pre-flight check — returns False if insufficient quota."""
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get(
                "https://api.elevenlabs.io/v1/user/subscription",
                headers={"xi-api-key": self._api_key},
            )
            data = response.json()
            remaining = data["character_limit"] - data["character_count"]
            return remaining >= chars_needed
    except Exception:
        return True  # Assume available on check failure
```

**File**: `kitesforu-workers/src/workers/stages/audio/tts_orchestrator.py`

Before generating, check quota:
```python
# In _generate_parallel(), before batch loop:
if provider_type == ProviderType.ELEVENLABS:
    total_chars = sum(len(seg.text) for seg in segments)
    provider = get_provider(ProviderType.ELEVENLABS)
    if not await provider.check_quota(total_chars):
        down_providers.add("elevenlabs")
        self.logger.info("ElevenLabs quota insufficient, skipping")
```

### P1-5: ElevenLabs optimize_streaming_latency + previous_text

**Impact**: 75% latency improvement per segment + better prosody
**Effort**: 2 hours

**File**: `kitesforu-workers/src/workers/stages/audio/providers/elevenlabs_provider.py:269-287`

```python
payload: Dict = {
    "text": text,
    "model_id": model,
    "voice_settings": voice_settings,
    "optimize_streaming_latency": 2,  # NEW: sweet spot
}

# Add context for better prosody at chunk boundaries
if previous_text:
    payload["previous_text"] = previous_text[-1000:]  # last 1000 chars
if next_text:
    payload["next_text"] = next_text[:1000]  # first 1000 chars
```

### P1-6: Parallel TTS chunk processing within providers

**Impact**: 5-10x speedup for multi-chunk segments
**Effort**: 1 day

Currently chunks are processed sequentially in each provider's `generate()`:
```python
for i, chunk in enumerate(chunks):
    audio = await self.generate_with_retry(...)
    audio_parts.append(audio)
```

Fix (parallel with semaphore):
```python
sem = asyncio.Semaphore(10)

async def gen_chunk(i, chunk):
    async with sem:
        return await self.generate_with_retry(
            lambda c=chunk: self._call_api(c, voice_id, model, voice_params, language),
            description=f"TTS chunk {i+1}/{len(chunks)}",
        )

tasks = [gen_chunk(i, c) for i, c in enumerate(chunks)]
audio_parts = list(await asyncio.gather(*tasks))
```

**Files**: Apply to all providers:
- `inworld_provider.py:180-187`
- `elevenlabs_provider.py` (equivalent location)
- `openai_provider.py` (equivalent location)

### P1-7: Firestore async client + batch writes

**Impact**: 20-55% faster Firestore operations
**Effort**: 3-5 days

Replace sync Firestore with native async:

**File**: `kitesforu-workers/src/workers/server.py`
```python
# Before
from google.cloud import firestore
db = firestore.Client()

# After
from google.cloud.firestore_v1 import AsyncClient
db = AsyncClient()
```

Batch stage lifecycle writes:

**File**: `kitesforu-workers/src/workers/base.py:321-384`
```python
async def _complete_stage(self, job_ref, stage, result, duration_ms):
    """Batch all completion writes into one Firestore call."""
    batch = self.db.batch()
    batch.update(job_ref, {
        f"stages.{stage}.status": "completed",
        f"stages.{stage}.completed_at": firestore.SERVER_TIMESTAMP,
        f"stages.{stage}.result": result,
        f"stage_durations.{stage}_ms": duration_ms,
        f"stage_durations.{stage}_success": True,
    })
    await batch.commit()  # One network call instead of 3-5
```

### P1-8: Anthropic model routing — Haiku for simple stages

**Impact**: 67% cost reduction on mechanical stages, faster responses
**Effort**: 1 day

Use claude-haiku-4-5 for:
- Research planner task generation (simple JSON output)
- SSML/formatting tasks
- Query generation

Keep claude-sonnet-4-6 for:
- Assimilator synthesis (needs reasoning)
- Script dialogue (needs quality)

**File**: `kitesforu-workers/src/workers/routing/router.py` — add stage-aware model selection

### P1-9: orjson for JSON serialization

**Impact**: 5-8x faster JSON ops
**Effort**: 30 minutes

**File**: `requirements.txt` — add `orjson`

Replace throughout:
```python
# Before
import json
data = json.dumps(research_results)

# After
import orjson
data = orjson.dumps(research_results).decode()
```

### P1 Combined Impact

| Optimization | Time Saved | Cost Saved |
|-------------|-----------|-----------|
| Prompt caching (system/user split) | 2-5s | 50-90% input tokens |
| reasoning_effort per stage | 5-15s | 30-50% LLM cost |
| JSON mode | 1-2s (no retries) | 20-30% output tokens |
| TTS quota pre-check | 10-15s (when down) | $0 |
| ElevenLabs optimize_streaming_latency | 3-5s | $0 |
| Parallel TTS chunks | 10-20s | $0 |
| Firestore async + batch | 3-8s | $0 |
| Haiku for simple stages | 2-5s | 67% on those stages |
| orjson | 0.02s | $0 |
| **P1 Total** | **~35-75s** | **~50% overall** |

**~135s after P0 → ~60-100s after P0+P1**

---

## 5. P2 — Pipeline Architecture (Weeks)

### P2-1: Combine Plan + Execute Tools workers

**Impact**: Eliminate 1 Pub/Sub hop (13s) + reduce Firestore overhead
**Effort**: 1 week

After planner generates research tasks, execute them in the same process. The tasks are lightweight web searches.

### P2-2: Streaming Script → Audio pipeline

**Impact**: Overlap script generation (17.8s) with audio generation (54.6s)
**Effort**: 2 weeks

`StreamingScriptAudioWorker` already exists at `stages/combined/streaming_script_audio_worker.py`. Wire it as the default path for podcast generation.

### P2-3: Cloud Workflows orchestration

**Impact**: Replace 5 Pub/Sub hops with centralized orchestration
**Effort**: 2-3 weeks
**Cost**: $0.00035 per pipeline run (negligible)

Cloud Workflows supports parallel step execution, built-in retry, and execution visibility in Cloud Console.

### P2-4: Assimilator prompt optimization

**Impact**: Reduce assimilator from 31.7s to ~15-20s
**Effort**: 3 days

- Pre-compress research results (JSON minification saves 30-40% tokens)
- Reduce max_tokens from 4000 to 2000
- Use structured output for guaranteed schema

### P2-5: Async GCS uploads

**Impact**: 50-80% faster audio uploads
**Effort**: 2 days

Replace sync `google-cloud-storage` with `gcloud-aio-storage`:
```python
from gcloud.aio.storage import Storage
```

---

## 6. P3 — Advanced Optimizations (Future)

### P3-1: Gemini context caching for multi-episode batches
- Explicit cache for shared system prompts across episodes
- 75% input token discount

### P3-2: OpenAI Batch API for overnight generation
- 50% cost discount on all tokens
- For non-time-sensitive "queue" mode

### P3-3: Anthropic Batch + Cache stacking
- 50% batch + 90% cache = 95% off input tokens
- For Pro/Ultimate tier batch runs

### P3-4: Connection pre-warming on worker startup
- HEAD requests to API endpoints during init
- Saves 100-300ms per provider on first real request

### P3-5: Google Cloud Profiler integration
- Continuous production profiling
- 3 lines of code, identifies real bottlenecks

### P3-6: Parallel script generation for long episodes
- Generate intro/body/outro in parallel
- ~2x speedup for 15+ min episodes at 1.8x token cost

---

## 7. Complete File Inventory

### Files that need changes

| File | Changes | Priority |
|------|---------|----------|
| **Infrastructure** | | |
| `terraform/cloud_run_workers.tf` | min_instances, startup_cpu_boost (lines 144,281,420,568,712,849,1038) | P0 |
| `terraform/cloud_run_car_mode.tf` | startup_cpu_boost | P0 |
| `terraform/cloud_run_course_workers.tf` | startup_cpu_boost | P0 |
| **Workers — Docker** | | |
| `Dockerfile` | Python 3.12, compileall, uvloop CMD | P0 |
| `requirements.txt` | Add uvloop, orjson, h2, gcloud-aio-storage | P0/P1 |
| **Workers — TTS Providers** | | |
| `providers/inworld_provider.py:99` | max_chunk_size 5000→1900 | BUG |
| `providers/inworld_provider.py:151` | Temperature range fix | BUG |
| `providers/inworld_provider.py:218` | Remove timestampType: WORD | BUG |
| `providers/inworld_provider.py:226` | Persistent httpx client | P0 |
| `providers/inworld_provider.py:180` | Parallel chunk generation | P1 |
| `providers/elevenlabs_provider.py:279` | Persistent httpx client | P0 |
| `providers/elevenlabs_provider.py:269` | optimize_streaming_latency, previous_text | P1 |
| `providers/elevenlabs_provider.py` | check_quota() method | P1 |
| **Workers — TTS Orchestrator** | | |
| `tts_orchestrator.py:300` | Pre-flight quota check | P1 |
| **Workers — LLM Providers** | | |
| `routing/providers/openai_provider.py:79` | System/user messages, reasoning_effort, response_format | P1 |
| `routing/providers/anthropic_provider.py:148` | System with cache_control, system blocks | P1 |
| `routing/providers/google_provider.py:100` | thinking_budget per stage | P1 |
| **Workers — Integration** | | |
| `routing/integration.py:321` | system_prompt/user_prompt params, response_format | P1 |
| **Workers — Pipeline** | | |
| `server.py:56` | PublisherClient batch_settings | P0 |
| `server.py` | Firestore AsyncClient | P1 |
| `base.py:321-384` | Batch Firestore writes | P1 |
| **Workers — Stages** | | |
| `stages/assimilator/worker.py:341-362` | Pass system/user separately | P1 |
| `stages/script/worker.py:509` | Pass system/user separately | P1 |
| `stages/planner/prompt_builder.py:159` | Pass system/user separately | P1 |

---

## 8. Implementation Roadmap

### Sprint 1: P0 Infrastructure (1-2 days)

**Day 1 Morning**: Terraform + Docker
- [ ] Add `min_instance_count = 1` to 6 critical workers
- [ ] Add `startup_cpu_boost = true` to ALL Cloud Run services
- [ ] Update Dockerfile: Python 3.12, compileall, uvloop CMD
- [ ] Add `uvloop`, `h2` to requirements.txt
- [ ] PR → merge → terraform apply → verify Cloud Run revisions

**Day 1 Afternoon**: Code quick wins
- [ ] Fix Inworld bugs (max_chunk_size, timestampType, temperature)
- [ ] Persistent httpx clients (Inworld, ElevenLabs)
- [ ] Disable Pub/Sub publisher batching
- [ ] PR → merge → verify deployment

**Day 2**: Verify & measure
- [ ] Run test podcast, compare timing
- [ ] Expected: 211s → ~135-150s

### Sprint 2: P1 Provider Optimizations (1-2 weeks)

**Week 1**:
- [ ] System/user message separation in LLM providers
- [ ] Anthropic cache_control on system prompts
- [ ] OpenAI reasoning_effort per stage
- [ ] JSON mode for structured outputs
- [ ] ElevenLabs quota pre-check
- [ ] ElevenLabs optimize_streaming_latency + previous_text

**Week 2**:
- [ ] Parallel TTS chunk processing in all providers
- [ ] Firestore AsyncClient migration
- [ ] Batch Firestore writes in base worker
- [ ] orjson integration
- [ ] Haiku routing for simple stages

**Expected after Sprint 2**: 211s → ~60-90s

### Sprint 3: P2 Architecture (2-4 weeks)

- [ ] Combine Plan + Execute Tools workers
- [ ] Wire StreamingScriptAudioWorker as default
- [ ] Assimilator prompt optimization
- [ ] Async GCS uploads
- [ ] Evaluate Cloud Workflows migration

**Expected after Sprint 3**: ~45-60s

---

## 9. Projected Impact Summary

```
CURRENT STATE
═══════════════════════════════════════════════════
Job 78c4b6b1: 211.8 seconds (3.5 minutes)
Cost: $0.164/episode

AFTER P0 (Infrastructure — 2 days)
═══════════════════════════════════════════════════
Target: ~135-150 seconds (2.2-2.5 min)
Speedup: 30-36%
Cost delta: +$35-80/month infrastructure

AFTER P0 + P1 (Provider APIs — 2 weeks)
═══════════════════════════════════════════════════
Target: ~60-90 seconds (1.0-1.5 min)
Speedup: 57-72%
Cost delta: -50% per episode ($0.08/episode)

AFTER P0 + P1 + P2 (Architecture — 6 weeks)
═══════════════════════════════════════════════════
Target: ~45-60 seconds (<1 min)
Speedup: 72-79%
Cost delta: -60% per episode ($0.065/episode)

ALL OPTIMIZATIONS APPLY TO CAR MODE
```

---

## Research Sources

Full research reports from 8 deep-dive agents are in `claudedocs/`:
- `openai-api-optimization-research.md`
- `anthropic-api-optimization.md`
- `elevenlabs-api-optimization-research.md`
- `gcp-tts-gemini-optimization-research.md`
- `pipeline-optimization-research.md`
- `prompt-optimization-research.md`

Key references:
- Inworld TTS docs: 2,000 char API limit, 100 req/s rate limit, audio markup English-only
- OpenAI: reasoning_effort `none`/`low`/`medium`/`high`, auto prompt caching ≥1024 tokens
- Anthropic: cache_control ephemeral (90% discount, 5-min TTL), Haiku 4.5 = old Sonnet 4
- ElevenLabs: optimize_streaming_latency=2 sweet spot, previous_text/next_text for prosody
- Cloud Run: startup_cpu_boost is free, min_instance idle = 10% vCPU cost
- uvloop: 25-35% improvement, 1 line to enable
- Firestore: native async since v2.1.0, 20-55% faster with AsyncClient
