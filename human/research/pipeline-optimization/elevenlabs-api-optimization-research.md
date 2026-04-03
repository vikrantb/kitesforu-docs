> **Research Date**: April 2026. Verify current API documentation before acting on specific parameters or pricing.

# ElevenLabs API Optimization for Podcast Generation Pipeline

Deep research report covering all optimization vectors for the KitesForU podcast pipeline using `eleven_v3` and `eleven_multilingual_v2`.

**Date**: 2026-04-02
**Current implementation**: `src/workers/stages/audio/providers/elevenlabs_provider.py`

---

## Table of Contents

1. [WebSocket Streaming vs REST](#1-websocket-streaming-vs-rest)
2. [optimize_streaming_latency Parameter](#2-optimize_streaming_latency-parameter)
3. [output_format Parameter](#3-output_format-parameter)
4. [Model Comparison: Turbo v2.5 vs v3 vs Multilingual v2 vs Flash v2.5](#4-model-comparison)
5. [Voice Latency Optimization](#5-voice-latency-optimization)
6. [Chunking Strategy](#6-chunking-strategy)
7. [Concurrency Limits](#7-concurrency-limits)
8. [previous_text and next_text Parameters](#8-previous_text-and-next_text-parameters)
9. [Pronunciation Dictionaries](#9-pronunciation-dictionaries)
10. [Cost Optimization](#10-cost-optimization)
11. [Quota Management](#11-quota-management)

---

## 1. WebSocket Streaming vs REST

### Three Endpoint Types

ElevenLabs provides three distinct TTS endpoint types:

**Regular REST** (`POST /v1/text-to-speech/{voice_id}`)
- Waits for full synthesis, returns complete audio file
- Simplest integration, no connection management
- Current implementation in our pipeline

**Streaming REST** (`POST /v1/text-to-speech/{voice_id}/stream`)
- Returns audio chunks progressively via chunked transfer encoding
- Reduces time-to-first-byte (TTFB)
- Text must be provided upfront (not incremental)

**WebSocket** (`wss://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream-input`)
- Bidirectional persistent connection
- Send text incrementally, receive audio chunks as they generate
- Best for real-time LLM-output-to-speech pipelines
- Supports `flush` messages to force remaining buffer output

### Latency Benchmarks (Real-World 2026)

From India (relevant for our GCP us-central1 deployment):

| Method | Format | TTFB Average | Total Time Average |
|---|---|---|---|
| Streaming REST | pcm_22050 | 478ms | 515ms |
| Regular REST | mp3_44100 | 500ms | 521ms |
| Streaming REST | mp3_44100 | 590ms | 608ms |
| WebSocket | pcm_16000 | 711ms | 718ms |
| WebSocket | pcm_22050 | 866ms | 907ms |

**Key finding**: Streaming REST outperforms WebSocket from high-latency regions. The WebSocket handshake adds ~233ms overhead. WebSocket is faster only from regions near ElevenLabs servers (US, EU) where the persistent connection amortizes over many requests.

### Relevance to Our Pipeline

For **batch podcast generation** (not real-time), WebSocket has limited benefit because:
- We have all text upfront (not streaming from an LLM)
- We process segments sequentially per chunk anyway
- The connection overhead does not amortize for single-segment calls

**However**, WebSocket can help if we keep a connection open and send multiple segments through it, avoiding per-request TCP/TLS overhead. The WebSocket protocol supports sending multiple text chunks through a single connection.

### WebSocket Chunk Protocol

```python
import websockets
import json

async def stream_via_websocket(voice_id: str, texts: list[str], api_key: str):
    uri = f"wss://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream-input"
    uri += "?model_id=eleven_v3&output_format=mp3_44100_128"

    async with websockets.connect(uri, extra_headers={"xi-api-key": api_key}) as ws:
        # Send initial config
        await ws.send(json.dumps({
            "text": " ",  # Initial space to open stream
            "voice_settings": {"stability": 0.45, "similarity_boost": 0.75},
            "xi_api_key": api_key,
        }))

        for text in texts:
            await ws.send(json.dumps({"text": text}))
            # Collect audio chunks
            while True:
                response = await ws.recv()
                if isinstance(response, bytes):
                    yield response  # Audio chunk
                else:
                    data = json.loads(response)
                    if data.get("isFinal"):
                        break

        # Send empty string to close
        await ws.send(json.dumps({"text": ""}))
```

### Recommendation for Our Pipeline

**Stay with REST for now.** Our podcast generation is batch-mode from Cloud Run. The streaming REST endpoint would help if we needed progressive playback, but we concatenate all audio at the end. The ~20ms difference between REST and streaming REST is negligible for batch. If we later need to reduce overall wall-clock time for many segments, consider a persistent WebSocket connection that sends all segments sequentially through one connection.

**Expected improvement**: Negligible for batch. ~50-100ms per request if switching to WebSocket from US-based infrastructure for multi-segment streams.

---

## 2. optimize_streaming_latency Parameter

### Parameter Values

| Value | Description | Quality Impact |
|---|---|---|
| 0 | Default mode (no optimizations) | Full quality |
| 1 | Normal optimizations (~50% of max latency improvement) | Minimal quality loss |
| 2 | Strong optimizations (~75% of max latency improvement) | Slight quality loss |
| 3 | Max latency optimizations | Noticeable quality loss |
| 4 | Max + text normalizer disabled | Worst quality; mispronounces numbers/dates |

### How It Works

This parameter trades audio quality for reduced time-to-first-byte. It affects:
- Internal model buffer sizes (smaller buffers = faster first chunk, more artifacts)
- Text preprocessing pipeline (levels 3-4 skip normalization steps)
- Audio encoding pipeline optimization

### Does It Affect Non-Streaming (Batch) Use?

**No.** This parameter is a **query parameter** on the streaming and WebSocket endpoints only. It does not apply to the regular `POST /v1/text-to-speech/{voice_id}` endpoint (which our pipeline currently uses). It only affects the `/stream` and WebSocket variants.

### API Usage

```python
# Only applies to streaming endpoint
url = f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream"
url += "?optimize_streaming_latency=2"  # Sweet spot

# Python SDK
audio_stream = client.text_to_speech.stream(
    voice_id="...",
    text="...",
    model_id="eleven_multilingual_v2",
    optimize_streaming_latency=2,  # 75% of max latency improvement
)
```

### Recommendation for Our Pipeline

**Not applicable to current implementation.** We use the regular (non-streaming) REST endpoint. If we switch to streaming, **level 2 is the sweet spot** -- 75% latency reduction with minimal quality degradation. Level 4 should be avoided for podcasts since it disables text normalization, causing mispronunciation of numbers and dates.

**Expected improvement**: 0% on current batch endpoint. ~75% TTFB reduction if we switch to streaming with level 2.

---

## 3. output_format Parameter

### Available Formats

| Format | Sample Rate | Bitrate | Tier Required | Use Case |
|---|---|---|---|---|
| `mp3_22050_32` | 22.05 kHz | 32 kbps | Free | Low-quality, smallest files |
| `mp3_44100_32` | 44.1 kHz | 32 kbps | Free | Low bitrate at full sample rate |
| `mp3_44100_64` | 44.1 kHz | 64 kbps | Free | Moderate quality |
| `mp3_44100_96` | 44.1 kHz | 96 kbps | Free | Good quality |
| `mp3_44100_128` | 44.1 kHz | 128 kbps | Free | **Standard quality (default)** |
| `mp3_44100_192` | 44.1 kHz | 192 kbps | Creator+ | High quality |
| `pcm_16000` | 16 kHz | Raw | Free | Telephony |
| `pcm_22050` | 22.05 kHz | Raw | Free | Medium raw audio |
| `pcm_24000` | 24 kHz | Raw | Free | Standard raw audio |
| `pcm_44100` | 44.1 kHz | Raw | Pro+ | High-fidelity raw |
| `pcm_48000` | 48 kHz | Raw | Pro+ | Maximum fidelity |
| `opus_48000_128` | 48 kHz | 128 kbps | Free | Efficient compression |
| `ulaw_8000` | 8 kHz | Raw | Free | Twilio/telephony |

### Does Format Affect Generation Speed?

**Marginally.** Real-world benchmarks show:
- PCM formats are ~22ms faster TTFB than MP3 (no encoding overhead)
- Lower sample rates do not meaningfully change model inference time
- The encoding step (MP3, Opus) adds small overhead but is not the bottleneck
- Higher bitrate MP3 (192 kbps) has negligible additional latency vs 128 kbps

The model generates audio at a fixed internal sample rate regardless of output format. The format only affects the final encoding step.

### Podcast Quality vs Speed Tradeoff

For podcast content:
- **`mp3_44100_128`** (current implicit default) is the sweet spot: good quality, universal compatibility, reasonable file size
- **`mp3_44100_192`** provides marginally better quality (requires Creator+ plan)
- **`mp3_22050_32`** is inappropriate for podcasts (noticeable quality degradation)
- **PCM formats** produce larger files and require post-processing to MP3

### Current Implementation Gap

Our `_call_api` method sends `Accept: audio/mpeg` but does not specify `output_format` as a query parameter. ElevenLabs defaults to `mp3_44100_128`, which is correct for podcasts.

### Recommendation

**Keep `mp3_44100_128` (the default).** The 22ms PCM advantage is irrelevant for batch processing. If file size is a concern, `mp3_44100_96` cuts size by ~25% with minimal perceptible quality loss for spoken content. Do not change format for speed reasons.

**Expected improvement**: ~22ms per request if switching to PCM (not recommended for podcast output).

---

## 4. Model Comparison

### Current Models Supported

| Model | Latency | Quality | Languages | Char Limit | Cost (credits/char) | Tags Support |
|---|---|---|---|---|---|---|
| `eleven_v3` | Higher (~1-2s) | **Highest** | 70+ | 5,000 | 1.0 | Yes (audio tags) |
| `eleven_multilingual_v2` | Medium | High | 29 | 10,000 | 1.0 | No |
| `eleven_flash_v2_5` | **~75ms** | Good | 32 | 40,000 | 0.5 | No |
| `eleven_turbo_v2_5` | Low (~250-300ms) | High | 32 | 40,000 | 0.5 | No |

### Critical Deprecation Notice

**`eleven_turbo_v2_5` and `eleven_turbo_v2` are deprecated.** ElevenLabs documentation explicitly states:

> The eleven_turbo_v2_5 and eleven_turbo_v2 models are functionally equivalent to the eleven_flash_v2_5 and eleven_flash_v2 models respectively, except the latency on the Flash models is lower on average. We recommend using the Flash models over Turbo models in all use cases.

**Action required**: Replace `eleven_turbo_v2_5` with `eleven_flash_v2_5` in `supported_models` list.

### Model Selection Guide for Podcasts

**For English-only premium podcasts**: `eleven_v3`
- Best emotional range and expressiveness
- Audio tag support ([whispers], [laughs], [sighs])
- Multi-speaker dialogue support
- Higher latency is acceptable for batch generation
- 5,000 char limit means more chunks (our current 5,000 char limit is aligned)

**For multilingual podcasts**: `eleven_multilingual_v2`
- Best language fidelity across 29 languages
- Rich emotional expression (though less than v3)
- 10,000 char limit means fewer chunks
- No audio tag support (tags get spoken aloud -- our code correctly strips them)

**For cost-sensitive batch generation**: `eleven_flash_v2_5`
- **50% cheaper** (0.5 credits/char vs 1.0)
- 40,000 char limit (could send entire scripts in one call)
- 75ms inference (but irrelevant for batch)
- Slight quality reduction vs v3/multilingual_v2

### Quality Rankings (Artificial Analysis Arena ELO)

Based on independent blind tests, quality ranking:
1. `eleven_v3` -- highest quality, best expressiveness
2. `eleven_multilingual_v2` -- close second, better stability
3. `eleven_flash_v2_5` -- good quality, best for cost/speed
4. `eleven_turbo_v2_5` -- equivalent to flash (deprecated alias)

### Recommendation

Our current model selection is correct:
- **Ultimate tier**: `eleven_v3` for maximum expression (already implemented)
- **Other tiers**: `eleven_multilingual_v2` for quality (already implemented)
- **Consider**: Adding `eleven_flash_v2_5` as a cost-tier option for free/starter users (50% cheaper)
- **Action**: Remove `eleven_turbo_v2_5` from supported models list (deprecated)

---

## 5. Voice Latency Optimization

### Voice Type Latency Hierarchy

ElevenLabs documents this ordering from fastest to slowest:

1. **Default/Premade voices** -- fastest (pre-optimized)
2. **Synthetic voices** (designed voices) -- fast
3. **Instant Voice Clones (IVC)** -- slightly slower
4. **Professional Voice Clones (PVC)** -- slowest (additional model complexity)

### voice_settings Impact on Latency

From ElevenLabs API documentation:

**`use_speaker_boost`**: "Requires a slightly higher computational load, which in turn **increases latency**."

**`style`**: "Consumes additional computational resources and **might increase latency** if set to anything other than 0."

**`stability`**: No documented latency impact.

**`similarity_boost`**: No documented latency impact.

### Our Current Settings Analysis

```python
NATURAL_BASELINE = {
    "stability": 0.45,
    "similarity_boost": 0.75,
    "style": 0.35,          # Adds latency (non-zero)
    "use_speaker_boost": True,  # Adds latency
}
```

Both `style > 0` and `use_speaker_boost = True` add latency. For batch podcast generation this is acceptable because quality matters more than speed. For a hypothetical real-time mode, reducing `style` to 0 and disabling `use_speaker_boost` would reduce latency.

### Pre-warming / Voice Caching

ElevenLabs does not offer an explicit "pre-warm" API. However:
- **First request after inactivity** has a cold-start penalty
- Sending a lightweight warmup request (~30 seconds before expected use) initializes the connection
- Voice data is cached server-side after first use in a session
- Using the same `voice_id` repeatedly within a short window benefits from server-side caching

### Recommendation

For batch podcasts: **No changes needed.** Our `style: 0.35` and `use_speaker_boost: True` add latency but are correct for quality. If we build a real-time preview feature, create a "fast" voice settings profile:

```python
FAST_SETTINGS = {
    "stability": 0.50,
    "similarity_boost": 0.75,
    "style": 0.0,              # Zero = no extra compute
    "use_speaker_boost": False,  # Skip speaker boost
}
```

**Expected improvement**: ~50-100ms per request by zeroing style and disabling speaker boost (not recommended for production podcast quality).

---

## 6. Chunking Strategy

### ElevenLabs Character Limits by Model

| Model | Max Characters per Request |
|---|---|
| `eleven_v3` | 5,000 |
| `eleven_multilingual_v2` | 10,000 |
| `eleven_flash_v2_5` | 40,000 |
| `eleven_flash_v2` | 30,000 |

### Our Current Implementation

```python
# elevenlabs_provider.py
max_chunk_size = 5000  # chars
chunk_by = "chars"
```

This is aligned with `eleven_v3`'s 5,000 char limit but leaves performance on the table for `eleven_multilingual_v2` (which allows 10,000).

### Optimal Chunk Size Analysis

From developer guides and real-world testing:

- **2,000 characters**: Most consistent latency per request
- **200-400 characters**: Recommended for WebSocket streaming (minimum 50 chars)
- **5,000 characters**: Upper practical limit for quality (v3's limit)
- **Larger chunks**: Fewer API calls but higher per-call latency and failure blast radius

For **podcast batch generation** (not real-time):
- Larger chunks = fewer API calls = fewer stitching artifacts
- Sentence-boundary splitting is critical (our `_find_sentence_boundary` does this correctly)
- Fewer chunks means less need for `previous_text`/`next_text` context

### WebSocket Chunk Recommendations

For WebSocket streaming specifically:
- Minimum: 50 characters
- Recommended: 200-400 characters
- Maximum: 1,000 characters
- Always flush at sentence breaks

### Dynamic Chunk Size by Model

```python
# Proposed improvement
MODEL_CHUNK_LIMITS = {
    "eleven_v3": 5000,
    "eleven_english_v3": 5000,
    "eleven_multilingual_v2": 10000,  # Currently wasted -- we chunk at 5000
    "eleven_flash_v2_5": 10000,      # Don't go to full 40k, diminishing returns
    "eleven_turbo_v2_5": 10000,
}
```

### Recommendation

**Increase chunk size for `eleven_multilingual_v2` to 10,000 chars.** This halves the number of API calls for multilingual content, reducing:
- Total wall-clock time (fewer round trips)
- Stitching artifacts at chunk boundaries
- Cost from retry overhead

Keep 5,000 for v3 models (hard API limit).

**Expected improvement**: ~50% fewer API calls for multilingual_v2 content, proportional reduction in total generation time.

---

## 7. Concurrency Limits

### Plan-Based Concurrency Limits

| Plan | Multilingual v2 Concurrent | Flash Concurrent | Priority |
|---|---|---|---|
| Free | 2 | 4 | 3 |
| Starter | 3 | 6 | 4 |
| Creator | 5 | 10 | 5 |
| Pro | 10 | 20 | 5 |
| Scale | 15 | 30 | 5 |
| Business | 15 | 30 | 5 |
| Enterprise | Elevated | Elevated | 6 |

**Flash models get 2x the concurrency of Multilingual v2.** This is significant for parallel generation.

### How Concurrency Is Counted

- Each in-flight API request counts as 1 concurrent request
- A "concurrency limit of 5 can typically support ~100 simultaneous audio broadcasts" because TTS requests are short-lived relative to audio playback
- Response headers include `current-concurrent-requests` and `maximum-concurrent-requests` for monitoring

### Rate Limit Behavior

HTTP 429 errors have two causes:
1. **`too_many_concurrent_requests`**: You've hit concurrency cap. Queue requests, don't retry blindly.
2. **`system_busy`**: Platform congestion. Retry with exponential backoff.

### Current Implementation Gap

Our `generate()` method processes chunks **sequentially**:

```python
for i, chunk in enumerate(chunks):
    audio = await self.generate_with_retry(...)
    audio_parts.append(audio)
```

This uses only 1 concurrent request at a time. For a podcast with 10 chunks, we could parallelize to use up to our plan's concurrency limit.

### Parallel Generation Pattern

```python
import asyncio

async def generate_parallel(self, chunks, voice_id, model, voice_settings, language):
    """Generate chunks in parallel up to concurrency limit."""
    semaphore = asyncio.Semaphore(5)  # Match plan limit

    async def generate_one(chunk, index):
        async with semaphore:
            return index, await self._call_api(chunk, voice_id, model, voice_settings, language)

    tasks = [generate_one(chunk, i) for i, chunk in enumerate(chunks)]
    results = await asyncio.gather(*tasks)
    # Sort by index to maintain order
    results.sort(key=lambda x: x[0])
    return [audio for _, audio in results]
```

### Recommendation

**Parallelize chunk generation within concurrency limits.** For a Pro plan with `eleven_multilingual_v2`, we can run 10 chunks simultaneously. This would reduce total generation time from `N * avg_latency` to `ceil(N/concurrency) * avg_latency`.

**Expected improvement**: For 10 chunks on Pro plan: ~10x faster total generation time (from sequential to parallel). Actual improvement depends on plan tier.

**Caveat**: Parallel generation breaks `previous_request_ids` stitching (chunks must be generated sequentially for stitching). See section 8 for the tradeoff.

---

## 8. previous_text and next_text Parameters

### Purpose

When splitting text into chunks, each chunk is synthesized independently. This causes:
- Abrupt prosody changes at chunk boundaries
- Inconsistent intonation between segments
- Short utterances getting random emphasis (e.g., "Ok." being screamed)

### Two Stitching Mechanisms

#### 1. Text-based: `previous_text` / `next_text`

```python
audio = client.text_to_speech.convert(
    text="current chunk text",
    voice_id="...",
    model_id="eleven_multilingual_v2",
    previous_text="text from previous chunk",  # Up to ~1000 chars
    next_text="text from next chunk",          # Up to ~1000 chars
)
```

- Provides textual context for prosody modeling
- Does **not** add meaningful latency (just text, no audio lookup)
- Works independently of request history
- Lower quality stitching than request ID method
- Best when `previous_text` is under 50 characters for STT context; for TTS, longer is fine

#### 2. Request ID-based: `previous_request_ids` / `next_request_ids`

```python
# Sequential generation with stitching
request_ids = []
for paragraph in paragraphs:
    with client.text_to_speech.with_raw_response.convert(
        text=paragraph,
        voice_id="...",
        model_id="eleven_multilingual_v2",
        previous_request_ids=request_ids[-3:]  # Max 3 IDs
    ) as response:
        request_id = response._response.headers.get("request-id")
        request_ids.append(request_id)
```

- Uses actual audio from previous generations as context
- **Requires sequential generation** (each request needs the previous one's ID)
- Request IDs expire after **2 hours**
- Maximum **3 previous request IDs** per call
- If both `previous_text` and `previous_request_ids` are provided, `previous_text` is **ignored**
- Best quality stitching available

### Latency Impact

- `previous_text` / `next_text`: **No meaningful latency addition** (just text context)
- `previous_request_ids`: **Requires sequential processing** (cannot parallelize), but the lookup itself adds minimal latency (~10-20ms to fetch cached audio fingerprint)

### The Parallelism vs Stitching Tradeoff

| Approach | Speed | Prosody Quality |
|---|---|---|
| Parallel chunks, no stitching | Fastest | Worst (abrupt transitions) |
| Sequential + `previous_text`/`next_text` | Medium | Good |
| Sequential + `previous_request_ids` | Slowest | Best |
| Parallel + `previous_text`/`next_text` | Fast | Good |

### Recommendation

**Implement `previous_text`/`next_text` with parallel generation.** This gives us:
- Parallel speed (section 7)
- Good prosody continuity (text context is known upfront)
- No sequential dependency (unlike `previous_request_ids`)

```python
async def generate_with_context(self, chunks, voice_id, model, voice_settings, language):
    semaphore = asyncio.Semaphore(5)

    async def generate_one(i, chunk):
        async with semaphore:
            prev_text = chunks[i - 1][-200:] if i > 0 else None
            next_text = chunks[i + 1][:200] if i < len(chunks) - 1 else None
            return i, await self._call_api_with_context(
                chunk, voice_id, model, voice_settings, language,
                previous_text=prev_text, next_text=next_text,
            )

    tasks = [generate_one(i, chunk) for i, chunk in enumerate(chunks)]
    results = await asyncio.gather(*tasks)
    results.sort(key=lambda x: x[0])
    return [audio for _, audio in results]
```

**Expected improvement**: Good prosody at chunk boundaries with zero latency penalty. Combining with parallel generation from section 7 gives both speed and quality.

---

## 9. Pronunciation Dictionaries

### What They Are

Pronunciation dictionaries let you define how specific words should be pronounced. Two formats:

1. **PLS files** (W3C standard): XML-based, use IPA or alias notation
2. **Rule-based**: Programmatic API with alias/phoneme rules

### Creating a Dictionary

```python
from elevenlabs import ElevenLabs

client = ElevenLabs(api_key="...")

# From rules (programmatic)
dictionary = client.pronunciation_dictionaries.create_from_rules(
    rules=[
        {
            "type": "alias",
            "string_to_replace": "KitesForU",
            "case_sensitive": True,
            "word_boundaries": True,
            "alias": "Kites For You",
        },
        {
            "type": "alias",
            "string_to_replace": "SCORM",
            "case_sensitive": True,
            "word_boundaries": True,
            "alias": "Scorm",
        },
    ],
    name="KitesForU Terms",
)

# Or from PLS file
with open("dictionary.pls", "rb") as f:
    dictionary = client.pronunciation_dictionaries.create_from_file(
        file=f.read(), name="KitesForU Terms"
    )
```

### Using with TTS

```python
audio = client.text_to_speech.convert(
    text="Welcome to KitesForU, your SCORM-compliant platform.",
    voice_id="...",
    model_id="eleven_multilingual_v2",
    pronunciation_dictionary_locators=[
        {
            "pronunciation_dictionary_id": dictionary.id,
            "version_id": dictionary.version_id,
        }
    ],
)
```

### Limitations

- Maximum **3 dictionary locators per request**
- Dictionaries are processed server-side (no client-side overhead)
- PLS files are case-sensitive (need both "tomato" and "Tomato")
- Supported on all models
- Phoneme tags (IPA/CMU Arpabet) only work with Flash v2, Turbo v2, English v1
- For v3 and Multilingual v2, use **alias tags** instead of phoneme tags

### Do They Affect Speed?

**No meaningful impact.** Dictionary lookup is a simple string-match preprocessing step that adds negligible latency (<5ms). The benefit is purely pronunciation quality.

### Recommendation

**Create a KitesForU pronunciation dictionary** for:
- Brand name: "KitesForU" -> "Kites For You"
- Technical terms specific to our content domains
- Common abbreviations in educational content

This is a quality improvement, not a speed improvement.

**Expected improvement**: Better pronunciation consistency. No speed impact.

---

## 10. Cost Optimization

### Pricing Model

ElevenLabs charges per **character**, not per API call.

| Model | Credits per Character | Cost per 1M chars (Pro plan) |
|---|---|---|
| `eleven_v3` | 1.0 | ~$170 |
| `eleven_multilingual_v2` | 1.0 | ~$170 |
| `eleven_flash_v2_5` | 0.5 | ~$85 |
| `eleven_turbo_v2_5` | 0.5 | ~$85 |

### What Counts as Characters?

- **Every character in the `text` field** counts, including:
  - Letters, numbers, punctuation
  - **Whitespace counts** (spaces, newlines)
  - **Expression tags count** ([whispers], [laughs] -- these are characters in the text)
- ElevenLabs does **not** support SSML for v3/multilingual_v2, so no SSML tag billing concern
- For models that support SSML (Flash v2, English v1), SSML markup tags are **not** counted (only the text content)

### Cost Reduction Strategies

#### 1. Strip unnecessary whitespace

```python
import re

def optimize_text_for_cost(text: str) -> str:
    # Collapse multiple spaces to single
    text = re.sub(r' +', ' ', text)
    # Remove leading/trailing whitespace per line
    text = '\n'.join(line.strip() for line in text.split('\n'))
    # Collapse multiple newlines
    text = re.sub(r'\n{3,}', '\n\n', text)
    return text.strip()
```

#### 2. Pre-normalize numbers and symbols

Letting ElevenLabs auto-normalize "1,234,567" costs 9 characters. Pre-normalizing to "one million two hundred thirty-four thousand five hundred sixty-seven" costs more characters. **Let ElevenLabs handle normalization** -- their auto mode is character-efficient.

However, with `apply_text_normalization: "off"` at level 4 latency optimization, you must pre-normalize. For batch generation, keep normalization on.

#### 3. Use Flash models for lower tiers

Flash v2.5 at 0.5 credits/char is **50% cheaper** than v3/multilingual_v2. For free-tier or cost-sensitive users, route to Flash.

#### 4. Cache generated audio

We already store generated audio in GCS. Ensure we cache by content hash so identical segments are not regenerated.

#### 5. Minimize expression tag verbosity

```python
# Expensive: 12 chars for the tag
"[whispering] Be very quiet"

# Cheaper: 10 chars for the tag
"[whispers] Be very quiet"
```

Use shorter tag forms where possible. The savings are small per tag but add up over thousands of segments.

### Free Regenerations

ElevenLabs offers **up to 2 free regenerations** per generation (same content and parameters). This means if audio quality is poor, we can retry twice without additional cost. Our retry logic should track this.

### Recommendation

- **Route free/starter users to `eleven_flash_v2_5`** (50% cost reduction)
- **Strip excess whitespace** before sending to API (small but free savings)
- **Cache audio by content hash** (already partially implemented)
- **Use shorter expression tags** where aliases exist

**Expected improvement**: Up to 50% cost reduction for non-premium tiers via Flash routing.

---

## 11. Quota Management

### Checking Quota: GET /v1/user/subscription

**Endpoint**: `GET https://api.elevenlabs.io/v1/user/subscription`
**Header**: `xi-api-key: YOUR_API_KEY`

### Response Fields

```json
{
    "tier": "pro",
    "character_count": 125000,       // Characters USED this period
    "character_limit": 500000,       // Total allowed this period
    "can_extend_character_limit": true,
    "allowed_to_extend_character_limit": true,
    "next_character_count_reset_unix": 1738356858,
    "voice_limit": 120,
    "voice_slots_used": 5,
    "professional_voice_limit": 1,
    "professional_voice_slots_used": 0,
    "can_use_instant_voice_cloning": true,
    "can_use_professional_voice_cloning": true,
    "status": "active",             // active, trialing, canceled, past_due, etc.
    "currency": "usd",
    "billing_period": "monthly_period",
    "character_refresh_period": "monthly_period",
    "has_open_invoices": false
}
```

### Key Fields for Pre-Flight Check

- **`character_count`**: Characters already used this billing cycle
- **`character_limit`**: Total allowed per cycle
- **`character_limit - character_count`** = remaining characters
- **`next_character_count_reset_unix`**: When quota resets (Unix timestamp)
- **`status`**: Must be "active" or "trialing" for API access

### Also Available via GET /v1/user

The `GET /v1/user` endpoint returns the same subscription data nested under `subscription`, plus user metadata.

### Response Headers on TTS Requests

Every TTS response includes:
- **`character-cost`**: Actual characters billed for this request
- **`request-id`**: For stitching and debugging
- **`current-concurrent-requests`**: Current in-flight count
- **`maximum-concurrent-requests`**: Your plan's concurrency limit

### Pre-Flight Check Implementation

```python
import httpx
from datetime import datetime

async def check_elevenlabs_quota(api_key: str, required_chars: int) -> dict:
    """Check if ElevenLabs has enough quota for a generation."""
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.get(
            "https://api.elevenlabs.io/v1/user/subscription",
            headers={"xi-api-key": api_key},
        )
        response.raise_for_status()
        data = response.json()

    used = data["character_count"]
    limit = data["character_limit"]
    remaining = limit - used
    reset_at = datetime.fromtimestamp(data["next_character_count_reset_unix"])

    return {
        "has_quota": remaining >= required_chars,
        "remaining_chars": remaining,
        "total_limit": limit,
        "used_chars": used,
        "utilization_pct": (used / limit * 100) if limit > 0 else 100,
        "reset_at": reset_at,
        "status": data["status"],
        "tier": data["tier"],
    }
```

### Integration with Fallback Chain

```python
async def should_use_elevenlabs(api_key: str, text: str) -> bool:
    """Decide whether to use ElevenLabs or fall back to another provider."""
    required_chars = len(text) * 1.1  # 10% buffer for retries
    quota = await check_elevenlabs_quota(api_key, int(required_chars))

    if not quota["has_quota"]:
        logger.warning(
            "ElevenLabs quota insufficient: %d remaining, %d required",
            quota["remaining_chars"],
            int(required_chars),
        )
        return False

    if quota["utilization_pct"] > 90:
        logger.warning(
            "ElevenLabs quota at %.1f%% utilization, conserving",
            quota["utilization_pct"],
        )
        # Could route to cheaper Flash model instead of blocking
        return True  # But flag for Flash routing

    return True
```

### Recommendation

**Implement pre-flight quota check before podcast generation.** A single podcast episode can consume 50,000-200,000 characters. Checking quota upfront prevents:
- Wasted compute on partial generations that fail mid-way
- Poor UX from partial podcast audio
- Allows graceful fallback to Inworld/OpenAI/Google TTS

**Expected improvement**: Zero wasted API calls on exhausted quota. Clean fallback behavior.

---

## Summary: Priority-Ordered Optimizations

| # | Optimization | Impact | Effort | Risk |
|---|---|---|---|---|
| 1 | **Parallel chunk generation** (Section 7) | High: ~5-10x total speed | Medium | Low -- respects concurrency limits |
| 2 | **previous_text/next_text stitching** (Section 8) | High: prosody quality | Low | None -- additive parameters |
| 3 | **Pre-flight quota check** (Section 11) | High: prevents wasted work | Low | None |
| 4 | **Dynamic chunk size by model** (Section 6) | Medium: ~50% fewer calls for v2 | Low | Low |
| 5 | **Replace turbo_v2_5 with flash_v2_5** (Section 4) | Medium: deprecated model | Trivial | None |
| 6 | **Flash routing for free/starter** (Section 10) | Medium: 50% cost reduction | Medium | Quality tradeoff |
| 7 | **Pronunciation dictionary** (Section 9) | Low: quality improvement | Low | None |
| 8 | **Whitespace stripping** (Section 10) | Low: small cost savings | Trivial | None |
| 9 | **WebSocket for multi-segment** (Section 1) | Low: marginal latency | High | Complexity |
| 10 | **optimize_streaming_latency** (Section 2) | N/A for batch | N/A | N/A |
| 11 | **Output format change** (Section 3) | Negligible for batch | N/A | N/A |

---

## Appendix: Current Implementation Gaps

### Files to Modify

1. **`src/workers/stages/audio/providers/elevenlabs_provider.py`**
   - Add `previous_text`/`next_text` to `_call_api()`
   - Add parallel chunk generation with semaphore
   - Dynamic `max_chunk_size` based on model
   - Add `output_format` query parameter (explicit is better than implicit)
   - Remove `eleven_turbo_v2_5` from `supported_models` (deprecated)

2. **`src/workers/stages/audio/providers/base.py`**
   - No changes needed (chunking logic is correct)

3. **New file: quota checker utility**
   - Pre-flight quota check before podcast generation
   - Integration with existing circuit breaker / fallback chain

4. **`src/workers/stages/audio/elevenlabs_v3.py`**
   - V3 conversation API -- no changes needed (already specialized)

### Response Header Tracking

Add response header capture to get:
- `request-id` for stitching via `previous_request_ids`
- `character-cost` for accurate cost tracking
- `current-concurrent-requests` for monitoring

```python
# Current: discards response metadata
response.raise_for_status()
return response.content

# Proposed: capture headers
response.raise_for_status()
request_id = response.headers.get("request-id")
char_cost = response.headers.get("character-cost")
concurrent = response.headers.get("current-concurrent-requests")
logger.info("ElevenLabs response", extra={
    "request_id": request_id,
    "character_cost": char_cost,
    "concurrent_requests": concurrent,
})
return response.content
```
