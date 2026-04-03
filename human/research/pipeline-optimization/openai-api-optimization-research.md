> **Research Date**: April 2026. Verify current API documentation before acting on specific parameters or pricing.

# OpenAI API Optimization Research for Podcast Generation Pipeline

**Date**: 2026-04-02
**Models in use**: gpt-5-mini, gpt-5.4
**Research scope**: 10 optimization areas with exact API parameters, code examples, and benchmarks

---

## Table of Contents

1. [Prompt Caching](#1-prompt-caching)
2. [Predicted Outputs](#2-predicted-outputs)
3. [reasoning_effort Parameter](#3-reasoning_effort-parameter)
4. [Batch API](#4-batch-api)
5. [Structured Outputs / JSON Mode](#5-structured-outputs--json-mode)
6. [Streaming](#6-streaming)
7. [max_completion_tokens vs max_tokens](#7-max_completion_tokens-vs-max_tokens)
8. [Parallel Function Calling](#8-parallel-function-calling)
9. [OpenAI TTS Optimizations](#9-openai-tts-optimizations)
10. [Connection Pooling with AsyncOpenAI](#10-connection-pooling-with-asyncopenai)

---

## 1. Prompt Caching

### How It Works

OpenAI's prompt caching is **fully automatic** -- no opt-in required. When you send a request, the system:

1. **Cache Routing**: Hashes the first ~256 tokens of your prompt prefix to route the request to a specific machine.
2. **Prefix Matching**: Compares your full prompt prefix against cached key/value tensors on that machine.
3. **Cache Hit**: If an exact prefix match is found, the cached computation is reused, skipping re-processing of those tokens.
4. **Incremental Matching**: Cache hits occur in **128-token increments** after the initial 1024-token minimum.

### Requirements

- **Minimum token threshold**: 1024 tokens. Prompts under 1024 tokens will always show `cached_tokens: 0`.
- **Exact prefix match**: The cached portion must be an identical prefix -- even a single character difference (like adding a timestamp) invalidates the cache.
- **Same machine routing**: Two requests must land on the same machine. The `prompt_cache_key` parameter influences routing.

### Cache TTL / Duration

| Retention Mode | Duration | How to Set |
|---|---|---|
| `in_memory` (default) | **5-10 minutes** of inactivity, up to ~1 hour | Default behavior |
| `24h` (extended) | **Up to 24 hours** | `"prompt_cache_retention": "24h"` |

Extended caching offloads key/value tensors to GPU-local storage when memory is full. OpenAI reports ~10% cache hit rate improvement for some use cases with extended retention.

### Cost Discount

- **50% discount on cached input tokens** (standard deployments)
- Cached tokens are billed at half the normal input token rate
- No additional cost for cache storage -- it is automatic

### What Can Be Cached

- Messages array (system, user, assistant)
- Images (links or base64)
- Tool definitions
- **Structured output schemas** (serves as prefix to system message)
- Audio tokens

### Compatibility

| Feature | Works with Caching? |
|---|---|
| JSON mode | Yes |
| Structured Outputs (json_schema) | Yes -- schema is cached as prefix |
| Tool calling / function definitions | Yes -- tool definitions are cached |
| Streaming | Yes |
| Batch API | Yes |

### Verifying Cache Hits

Cache hits appear in the `usage` response object:

```python
response = client.chat.completions.create(
    model="gpt-5-mini",
    messages=[
        {"role": "system", "content": LONG_SYSTEM_PROMPT},  # 1024+ tokens, static
        {"role": "user", "content": user_query},             # dynamic, at the end
    ]
)

# Check cache hit
usage = response.usage
cached = usage.prompt_tokens_details.cached_tokens
total_prompt = usage.prompt_tokens
print(f"Cache hit: {cached}/{total_prompt} tokens ({cached/total_prompt*100:.1f}%)")
```

Response structure:
```json
{
  "usage": {
    "prompt_tokens": 2006,
    "completion_tokens": 300,
    "total_tokens": 2306,
    "prompt_tokens_details": {
      "cached_tokens": 1920
    }
  }
}
```

### prompt_cache_key Parameter

For workloads where many requests share common prefixes, use `prompt_cache_key` to improve routing:

```python
response = client.chat.completions.create(
    model="gpt-5-mini",
    messages=messages,
    extra_body={
        "prompt_cache_key": "podcast-script-generation",
        "prompt_cache_retention": "24h"
    }
)
```

**Important constraint**: Keep each unique prefix + `prompt_cache_key` combination below **~15 requests per minute**. Exceeding this causes overflow to additional machines, reducing cache effectiveness.

### Best Practices for Podcast Pipeline

```python
# BEFORE: Dynamic content mixed throughout -- poor caching
messages = [
    {"role": "system", "content": f"You are a podcast script writer. Today is {date}. Topic: {topic}"},
    {"role": "user", "content": f"Generate a script about {topic}"}
]

# AFTER: Static prefix first, dynamic content last -- optimal caching
STATIC_SYSTEM_PROMPT = """You are a podcast script writer for KitesForU.
You generate engaging, educational podcast scripts...
[1024+ tokens of detailed instructions, format specs, examples]"""

messages = [
    {"role": "system", "content": STATIC_SYSTEM_PROMPT},  # cached
    {"role": "user", "content": f"Today is {date}. Topic: {topic}. Generate a script."}  # dynamic
]
```

### Expected Improvement

- **Cost**: 50% reduction on cached input tokens (typically 40-60% of prompt)
- **Latency**: Reduced prefill time -- varies by prompt length, typically 10-30% faster for long prompts
- **Verified benchmarks**: gpt-5-mini shows 97.67% cache coverage with 2490 token prompts (community testing, Jan 2026)

### Limitations

- Cache is best-effort, not guaranteed
- No way to force a cache hit -- only influence routing via `prompt_cache_key`
- Changing even one character in the prefix invalidates all subsequent cached tokens
- Avoid putting timestamps with seconds precision in system prompts (use "Evening" not "00:51:42")
- Compaction techniques that rewrite prompts can destroy cache locality

---

## 2. Predicted Outputs

### What It Is

The `prediction` parameter lets you provide expected output text. When a large portion of the response is known ahead of time, the model can skip regenerating those tokens, dramatically reducing latency.

### Model Support

Predicted Outputs are supported on:
- GPT-4o, GPT-4o-mini
- GPT-4.1, GPT-4.1-mini, GPT-4.1-nano

**NOT supported on GPT-5 series models (gpt-5-mini, gpt-5.4).**

### How It Works

```python
response = client.chat.completions.create(
    model="gpt-4.1",  # NOT available on gpt-5-mini or gpt-5.4
    messages=[
        {"role": "system", "content": "Regenerate this script with the correction applied."},
        {"role": "user", "content": f"Fix the pronunciation guide in this script:\n{existing_script}"}
    ],
    prediction={
        "type": "content",
        "content": existing_script  # the mostly-unchanged script
    }
)

# Check prediction effectiveness
usage = response.usage.completion_tokens_details
print(f"Accepted: {usage.accepted_prediction_tokens}")
print(f"Rejected: {usage.rejected_prediction_tokens}")  # these are still billed
```

### Latency Improvement

- Up to **2-4x faster** for regeneration tasks where >80% of output is unchanged
- The prediction text can appear **anywhere** in the response (not just at the start)
- Rejected prediction tokens are still **charged at output token rates**

### Limitations for Podcast Pipeline

- **Not available on gpt-5-mini or gpt-5.4** -- only GPT-4.x models
- Cannot be combined with: `n > 1`, `logprobs`, `presence_penalty > 0`, `frequency_penalty > 0`, `audio`
- Not compatible with streaming (in some implementations)
- Cost can increase if many tokens are rejected

### Applicability to Our Pipeline

**Low applicability**. Since we use gpt-5-mini and gpt-5.4, Predicted Outputs are not available. If we had a script-editing workflow on GPT-4.1, it would be useful for regenerating scripts with minor corrections. Monitor OpenAI for GPT-5 series support in future.

---

## 3. reasoning_effort Parameter

### What It Is

Controls how many internal reasoning tokens the model generates before producing a response. Lower effort = faster response, fewer tokens, lower cost. Higher effort = more thorough reasoning.

### Supported Values by Model

| Model | Supported Values | Default |
|---|---|---|
| GPT-5 (original) | `minimal`, `low`, `medium`, `high` | varies |
| GPT-5.2 | `none`, `low`, `medium`, `high` | `none` |
| **GPT-5.4** | **`none`, `low`, `medium`, `high`, `xhigh`** | **`none`** |
| **GPT-5-mini** | `low`, `medium` (likely), `high` | Check docs |
| GPT-5.4-mini | `none`, `low`, `medium`, `high`, `xhigh` | `none` |

**Note**: GPT-5.2 and newer default to `none` reasoning effort. If you need the model to think, you must explicitly set it higher.

### API Usage

**Responses API** (recommended for GPT-5.4):
```python
response = client.responses.create(
    model="gpt-5.4",
    input="Generate a podcast script outline about quantum computing",
    reasoning={
        "effort": "low"  # none | low | medium | high | xhigh
    }
)
```

**Chat Completions API** (for gpt-5-mini):
```python
response = client.chat.completions.create(
    model="gpt-5-mini",
    messages=[...],
    reasoning_effort="medium"  # varies by model
)
```

### Impact on Latency and Token Usage

| Effort Level | Reasoning Tokens | Latency | Quality | Cost |
|---|---|---|---|---|
| `none` | 0 | Fastest | Good for simple tasks | Lowest |
| `low` | Few | Fast | Better for moderate tasks | Low |
| `medium` | Moderate | Medium | Good balance | Medium |
| `high` | Many | Slower | Strong reasoning | Higher |
| `xhigh` | Maximum | Slowest | Best for complex tasks | Highest |

**Benchmarks from community testing (Jan 2026)**:
- gpt-5-mini at default: TTFT ~12.3s, stream rate ~212.9 tok/s
- gpt-5 at default: TTFT ~0.46s, stream rate ~45.3 tok/s
- Higher reasoning effort increases TTFT significantly on mini models

### Recommended Settings for Podcast Pipeline

```python
# Script generation (needs good structure, not deep reasoning)
SCRIPT_GENERATION_EFFORT = "low"  # fast, good enough for creative writing

# Research/planning stage (needs more thought)
RESEARCH_EFFORT = "medium"

# Simple formatting/extraction tasks
FORMATTING_EFFORT = "none"  # fastest, cheapest
```

### Additional Parameters with reasoning_effort

When `reasoning.effort` is set to `none` on GPT-5.4/5.2, these parameters become available:
- `temperature`
- `top_p`
- `logprobs`

These are **NOT available** at any other reasoning effort level or on older GPT-5 models.

### Expected Improvement

- **none vs medium**: ~3-5x latency reduction for simple tasks
- **none vs high**: ~5-10x latency reduction
- **Token savings**: Proportional to reasoning effort reduction
- **Tip**: Encourage the model to "think step by step" in the prompt even at `none` effort for better quality

---

## 4. Batch API

### How It Works

Submit a file of requests, OpenAI processes them asynchronously within 24 hours, you download results.

### Cost Savings

- **50% discount** on both input and output tokens compared to synchronous API
- Separate rate limit pool -- does not count against your real-time rate limits

### When to Use for Podcast Pipeline

- Pre-generating podcast scripts during off-peak hours
- Batch processing research summaries
- Generating multiple script variants
- Any job where results are NOT needed in real-time

### Step-by-Step Implementation

**Step 1: Create the JSONL input file**

```python
import json

requests = []
for i, topic in enumerate(topics):
    requests.append({
        "custom_id": f"script-{i}",
        "method": "POST",
        "url": "/v1/chat/completions",
        "body": {
            "model": "gpt-5-mini",
            "messages": [
                {"role": "system", "content": PODCAST_SYSTEM_PROMPT},
                {"role": "user", "content": f"Generate a podcast script about: {topic}"}
            ],
            "response_format": {"type": "json_object"},
            "max_completion_tokens": 4096
        }
    })

# Write JSONL file
with open("batch_input.jsonl", "w") as f:
    for req in requests:
        f.write(json.dumps(req) + "\n")
```

**Step 2: Upload and submit the batch**

```python
from openai import OpenAI
client = OpenAI()

# Upload the file
batch_input_file = client.files.create(
    file=open("batch_input.jsonl", "rb"),
    purpose="batch"
)

# Create the batch job
batch_job = client.batches.create(
    input_file_id=batch_input_file.id,
    endpoint="/v1/chat/completions",  # or "/v1/responses"
    completion_window="24h",
    metadata={"description": "podcast script generation batch"}
)
print(f"Batch ID: {batch_job.id}, Status: {batch_job.status}")
```

**Step 3: Poll for completion**

```python
import time

while True:
    batch = client.batches.retrieve(batch_job.id)
    print(f"Status: {batch.status}, "
          f"Completed: {batch.request_counts.completed}/"
          f"{batch.request_counts.total}")

    if batch.status in ("completed", "failed", "expired"):
        break
    time.sleep(60)  # check every minute
```

**Step 4: Download results**

```python
if batch.output_file_id:
    result_file = client.files.content(batch.output_file_id)
    results = []
    for line in result_file.text.strip().split("\n"):
        result = json.loads(line)
        custom_id = result["custom_id"]
        response_body = result["response"]["body"]
        content = response_body["choices"][0]["message"]["content"]
        results.append({"id": custom_id, "content": content})

# Handle errors
if batch.error_file_id:
    error_file = client.files.content(batch.error_file_id)
    for line in error_file.text.strip().split("\n"):
        error = json.loads(line)
        print(f"Error for {error['custom_id']}: {error['error']}")
```

### Expected Improvement

- **Cost**: 50% reduction on all tokens
- **Latency**: Results within 24 hours (often much faster -- reported under 1 hour for moderate batches)
- **Rate limits**: Substantially higher -- 250M input tokens enqueued for GPT-4T tier

### Limitations

- **24-hour completion window** -- cannot be shortened
- Sometimes batches do not complete and get cancelled
- No streaming -- you wait for the full batch
- Must construct raw HTTP request bodies (not SDK objects)

---

## 5. Structured Outputs / JSON Mode

### Three Options

| Feature | Parameter | Guarantee | Speed |
|---|---|---|---|
| **JSON Mode** | `response_format: {"type": "json_object"}` | Valid JSON, no schema guarantee | Fastest |
| **Structured Outputs** | `response_format: {"type": "json_schema", "json_schema": {...}}` | Valid JSON matching exact schema | Slightly slower (first request) |
| **Function Calling** | `tools: [...]` with `strict: true` | Schema-adherent function args | Similar to structured outputs |

### JSON Mode (Faster, Less Safe)

```python
response = client.chat.completions.create(
    model="gpt-5-mini",
    messages=[
        {"role": "system", "content": "Output valid JSON with keys: title, segments, duration"},
        {"role": "user", "content": "Generate podcast script structure for: AI in healthcare"}
    ],
    response_format={"type": "json_object"}
)
# Must still validate/parse -- no schema guarantee
script = json.loads(response.choices[0].message.content)
```

### Structured Outputs (Slower First Call, Guaranteed Schema)

```python
from pydantic import BaseModel
from typing import List

class PodcastSegment(BaseModel):
    speaker: str
    text: str
    duration_seconds: float

class PodcastScript(BaseModel):
    title: str
    segments: List[PodcastSegment]
    total_duration_seconds: float

response = client.beta.chat.completions.parse(
    model="gpt-5-mini",
    messages=[
        {"role": "system", "content": "Generate a podcast script."},
        {"role": "user", "content": "Topic: AI in healthcare"}
    ],
    response_format=PodcastScript
)
script = response.choices[0].message.parsed  # typed PodcastScript object
```

### Performance Comparison

| Metric | json_object | json_schema |
|---|---|---|
| First request latency | Normal | +1-60s (schema processing) |
| Subsequent requests | Normal | Normal (schema cached) |
| Parsing errors | Possible (no schema) | Zero (guaranteed) |
| Output tokens | Normal | Slightly more (whitespace) |
| Prompt caching compatible | Yes | Yes (schema is cached as prefix) |

### Recommendation for Pipeline

Use **Structured Outputs** (`json_schema`) for script generation:
- Eliminates parsing failures entirely
- Schema is cached after first request (no ongoing latency penalty)
- Structured output schemas contribute to the 1024-token caching minimum
- The slight first-request latency is negligible for a pipeline

### Caching Interaction

Structured output schemas act as a prefix to the system message and are cached. This means your schema + system prompt can together meet the 1024-token threshold even if neither alone would.

---

## 6. Streaming

### Does Streaming Reduce TTFT?

**No.** Streaming does not reduce the time the model takes to generate the first token. What it does:
- Delivers tokens **as they are generated** instead of waiting for the full response
- **Perceived latency** drops dramatically (user sees output immediately)
- Allows **parallel processing** while generation continues

### Time-to-First-Token

- TTFT is determined by model, prompt length, reasoning effort, and server load
- Streaming simply sends that first token to you immediately instead of buffering
- For a 2000-token response at 50 tok/s, streaming saves ~38 seconds of perceived wait

### Streaming with Usage Tracking

```python
stream = client.chat.completions.create(
    model="gpt-5-mini",
    messages=messages,
    stream=True,
    stream_options={"include_usage": True}  # get token counts
)

full_content = ""
usage_data = None

for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        token = chunk.choices[0].delta.content
        full_content += token
        # Can start processing here (e.g., partial JSON parsing)

    # Usage appears in the final chunk
    if chunk.usage:
        usage_data = chunk.usage
        print(f"Total tokens: {usage_data.total_tokens}")
        print(f"Cached tokens: {usage_data.prompt_tokens_details.cached_tokens}")
```

### Streaming JSON Parsing

You CAN parse JSON incrementally while streaming, but it requires careful handling:

```python
import json
from io import StringIO

buffer = StringIO()
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        buffer.write(chunk.choices[0].delta.content)

        # Attempt incremental parse (useful for detecting early fields)
        try:
            partial = json.loads(buffer.getvalue())
            # Successfully parsed so far
            if "title" in partial:
                print(f"Title detected: {partial['title']}")
        except json.JSONDecodeError:
            pass  # incomplete JSON, keep buffering
```

### Streaming Structured Outputs

```python
from openai import OpenAI

client = OpenAI()
with client.beta.chat.completions.stream(
    model="gpt-5-mini",
    messages=messages,
    response_format=PodcastScript,
    stream_options={"include_usage": True}
) as stream:
    for chunk in stream:
        if chunk.type == 'chunk':
            snapshot = chunk.to_dict()['snapshot']
            parsed = snapshot['choices'][0]['message'].get('parsed', {})
            # parsed grows incrementally as more JSON is streamed
```

### Expected Improvement

- **Perceived latency**: First token arrives in milliseconds after model starts generating (vs waiting for full response)
- **Pipeline optimization**: Can start TTS processing on early script segments while later segments are still generating
- **No cost difference**: Streaming does not change token pricing

### Limitations

- `stream_options: {"include_usage": true}` -- usage only appears in the **final** chunk
- Streaming adds slight overhead for very short responses
- Connection must stay open for the duration

---

## 7. max_completion_tokens vs max_tokens

### The Difference

| Parameter | API | Scope | Available On |
|---|---|---|---|
| `max_tokens` | Chat Completions | Output tokens only (visible) | GPT-3.5, GPT-4, GPT-4o (deprecated) |
| `max_completion_tokens` | Chat Completions | Output + reasoning tokens | o1, o3, o4-mini, GPT-5 series |
| `max_output_tokens` | Responses API | Output tokens | All models on Responses API |

### Why the Change

With reasoning models (o1 and later), the model generates **hidden reasoning tokens** that count toward the total but are not returned. `max_tokens` previously meant "tokens you receive" = "tokens generated". With reasoning, this is no longer true. `max_completion_tokens` makes the distinction explicit.

### Which to Use

```python
# For gpt-5-mini with Chat Completions API
response = client.chat.completions.create(
    model="gpt-5-mini",
    messages=messages,
    max_completion_tokens=4096  # includes reasoning tokens
)

# For gpt-5.4 with Responses API (preferred)
response = client.responses.create(
    model="gpt-5.4",
    input=messages,
    max_output_tokens=4096
)
```

### Does Lower max_tokens Reduce Latency?

**Not directly.** The model does not know the limit and does not generate faster. It simply **stops** when the limit is reached. However:
- A lower limit prevents runaway generation (cost protection)
- The reservation of output space in the context window can prevent errors
- For reasoning models, `max_completion_tokens` caps total generation including hidden reasoning -- setting it too low can cut off reasoning before any output is produced

### Recommendation

- Always set `max_completion_tokens` (or `max_output_tokens`) to a reasonable ceiling
- For podcast scripts (~2000-4000 tokens): set to 4096-8192
- Do NOT rely on this for latency optimization -- use `reasoning_effort` instead

---

## 8. Parallel Function Calling

### How It Works

The model can return **multiple function/tool calls in a single response turn**, enabling parallel execution.

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_topic",
            "description": "Search for information about a topic",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "source": {"type": "string", "enum": ["web", "academic", "news"]}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_statistics",
            "description": "Get statistics about a topic",
            "parameters": {
                "type": "object",
                "properties": {
                    "topic": {"type": "string"},
                    "year": {"type": "integer"}
                },
                "required": ["topic"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="gpt-5-mini",
    messages=[
        {"role": "user", "content": "Research AI in healthcare: get web info, academic papers, and 2025 statistics"}
    ],
    tools=tools,
    tool_choice="auto",
    parallel_tool_calls=True  # default is True for supported models
)

# Model may return multiple tool calls
for tool_call in response.choices[0].message.tool_calls:
    print(f"Call: {tool_call.function.name}({tool_call.function.arguments})")
    # Execute these in parallel with asyncio.gather()
```

### Model Support for parallel_tool_calls

| Model | Parallel Tool Calls |
|---|---|
| GPT-4o, GPT-4o-mini | Yes |
| GPT-4.1, GPT-4.1-mini | Yes |
| GPT-5, GPT-5-mini, GPT-5-nano | Yes |
| GPT-5.1, GPT-5.2 | Yes |
| o1, o3, o4-mini | **No** (reasoning models do sequential) |
| GPT-5.4 | Yes (check latest docs) |

### tool_choice Options

| Value | Behavior |
|---|---|
| `"auto"` (default) | Model decides when/how many tools to call |
| `"required"` | Must call one or more tools |
| `{"type": "function", "name": "..."}` | Must call exactly this function |
| `"none"` | No tool calls, text response only |

### Use Case for Podcast Pipeline

Research planning stage -- model can simultaneously request:
- Topic research from multiple sources
- Statistics lookup
- Related topics exploration
- Fact-checking queries

```python
# Execute parallel tool calls
import asyncio

async def execute_tool_calls(tool_calls):
    tasks = []
    for tc in tool_calls:
        fn_name = tc.function.name
        args = json.loads(tc.function.arguments)
        if fn_name == "search_topic":
            tasks.append(search_topic(**args))
        elif fn_name == "get_statistics":
            tasks.append(get_statistics(**args))
    return await asyncio.gather(*tasks)
```

### Expected Improvement

- **Latency**: Parallel execution of N tool calls reduces wall-clock time from N * avg_call_time to ~max(call_times)
- **For research stage**: 3 parallel searches at 2s each = 2s total vs 6s sequential

### Limitations

- Reasoning models (o-series) do NOT support parallel tool calls
- GPT-4.1-nano can sometimes duplicate tool calls when parallel is enabled
- Built-in tools (web_search, file_search) cannot be called in parallel

---

## 9. OpenAI TTS Optimizations

### Model Comparison

| Feature | tts-1 | tts-1-hd | gpt-4o-mini-tts |
|---|---|---|---|
| **Speed** | Fast | Medium | Fast |
| **Quality** | Good/Average | High | Higher |
| **Latency** | Lowest | ~1.5-2x tts-1 | Similar to tts-1 |
| **Cost** | $15/1M tokens | $30/1M tokens | $12/1M tokens |
| **Input Token Limit** | 4096 | 4096 | 2000 |
| **Voices** | 9 (alloy, ash, coral, echo, fable, onyx, nova, sage, shimmer) | 9 (same as tts-1) | 13 (adds ballad, verse, marin, cedar) |
| **Promptable** | No | No | Yes (accent, emotion, speed) |

### Streaming Audio

The TTS API supports **real-time audio streaming** via chunked transfer encoding:

```python
from openai import OpenAI
from pathlib import Path

client = OpenAI()

# Streaming TTS -- audio plays before full generation completes
with client.audio.speech.with_streaming_response.create(
    model="tts-1",
    voice="alloy",
    input="This is the first segment of our podcast about artificial intelligence.",
    response_format="mp3",  # or opus, aac, flac, wav, pcm
    speed=1.0  # 0.25 to 4.0
) as response:
    # Stream chunks to file or audio player
    with open("output.mp3", "wb") as f:
        for chunk in response.iter_bytes(chunk_size=1024):
            f.write(chunk)
            # Can also pipe to audio player for real-time playback
```

### Speed Parameter

```python
# speed: 0.25 to 4.0 (default 1.0)
response = client.audio.speech.create(
    model="tts-1",
    voice="alloy",
    input="Podcast segment text here",
    speed=1.0  # Changing speed does NOT affect generation time
)
```

**Key finding**: The `speed` parameter adjusts playback speed of the output audio but does **NOT** significantly affect generation time. A speed of 2.0 produces audio that plays back 2x faster but takes roughly the same time to generate.

### Chunking Strategy for Long Content

**Max input length**: 4096 tokens per request (tts-1 and tts-1-hd), 2000 tokens for gpt-4o-mini-tts.

For podcast scripts that exceed this:

```python
import re

def chunk_text_for_tts(text: str, max_chars: int = 4000) -> list[str]:
    """Split text at sentence boundaries for natural TTS chunking."""
    sentences = re.split(r'(?<=[.!?])\s+', text)
    chunks = []
    current_chunk = ""

    for sentence in sentences:
        if len(current_chunk) + len(sentence) + 1 > max_chars:
            if current_chunk:
                chunks.append(current_chunk.strip())
            current_chunk = sentence
        else:
            current_chunk += " " + sentence

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks

# Process chunks in parallel
import asyncio

async def generate_audio_parallel(chunks: list[str]):
    async_client = AsyncOpenAI()
    tasks = [
        async_client.audio.speech.create(
            model="tts-1",
            voice="alloy",
            input=chunk,
            response_format="mp3"
        )
        for chunk in chunks
    ]
    return await asyncio.gather(*tasks)
```

### Pipeline Optimization: Stream TTS While Generating Script

```python
async def pipeline_stream_tts(topic: str):
    """Generate script segments and immediately send to TTS."""
    # Start script generation with streaming
    stream = client.chat.completions.create(
        model="gpt-5-mini",
        messages=[...],
        stream=True
    )

    buffer = ""
    segment_index = 0

    for chunk in stream:
        if chunk.choices and chunk.choices[0].delta.content:
            buffer += chunk.choices[0].delta.content

            # When we have a complete segment (ends with period/newline)
            if buffer.rstrip().endswith(('.', '!', '?')) and len(buffer) > 200:
                # Fire off TTS immediately -- don't wait for full script
                asyncio.create_task(
                    generate_tts_segment(buffer, segment_index)
                )
                segment_index += 1
                buffer = ""
```

### Expected Improvement

- **tts-1 vs tts-1-hd**: ~40-50% faster generation for tts-1
- **Parallel chunking**: N chunks processed simultaneously instead of sequentially
- **Stream + TTS pipeline**: Can start audio generation seconds after script generation begins
- **Opus format**: Smallest file size, lowest streaming latency

### Recommendation for Pipeline

1. Use **tts-1** for drafts/previews (fast, cheap)
2. Use **tts-1-hd** for final podcast audio (quality)
3. Consider **gpt-4o-mini-tts** for promptable voice control at lower cost than tts-1-hd
4. Always use **streaming response** for TTS
5. Chunk scripts at sentence boundaries, process chunks in parallel
6. Use **opus** format for streaming, **mp3** for final output

---

## 10. Connection Pooling with AsyncOpenAI

### Does the SDK Handle Connection Pooling?

**Yes.** The OpenAI Python SDK uses httpx under the hood, which provides automatic connection pooling when you reuse a client instance.

### Default httpx Pool Settings

```python
# httpx defaults (what AsyncOpenAI uses internally)
httpx.Limits(
    max_connections=100,        # total concurrent connections
    max_keepalive_connections=20,  # idle connections kept alive
    keepalive_expiry=5.0        # seconds before closing idle connection
)
```

### Custom httpx Client Configuration

```python
import httpx
from openai import AsyncOpenAI

# Configure a custom httpx client with larger pool
custom_limits = httpx.Limits(
    max_connections=200,          # increase for high-concurrency pipeline
    max_keepalive_connections=50,  # more idle connections
    keepalive_expiry=30.0         # keep connections alive longer
)

custom_timeout = httpx.Timeout(
    connect=10.0,    # connection establishment
    read=120.0,      # reading response (long for LLM generation)
    write=10.0,      # sending request
    pool=30.0        # waiting for available connection from pool
)

# IMPORTANT: Use AsyncClient for AsyncOpenAI
http_client = httpx.AsyncClient(
    limits=custom_limits,
    timeout=custom_timeout
)

client = AsyncOpenAI(
    http_client=http_client
)
```

### Common Mistake: Sync Client with Async OpenAI

```python
# WRONG: Will crash -- sync httpx.Client with AsyncOpenAI
http_client = httpx.Client(limits=custom_limits)
client = AsyncOpenAI(http_client=http_client)  # ERROR!

# CORRECT: Use httpx.AsyncClient
http_client = httpx.AsyncClient(limits=custom_limits)
client = AsyncOpenAI(http_client=http_client)  # Works
```

### Singleton Pattern for Pipeline

```python
from openai import AsyncOpenAI
import httpx

class OpenAIClientPool:
    """Singleton client pool for the podcast pipeline."""

    _instance = None
    _client: AsyncOpenAI = None

    @classmethod
    def get_client(cls) -> AsyncOpenAI:
        if cls._client is None:
            http_client = httpx.AsyncClient(
                limits=httpx.Limits(
                    max_connections=200,
                    max_keepalive_connections=50,
                    keepalive_expiry=30.0
                ),
                timeout=httpx.Timeout(
                    connect=10.0,
                    read=300.0,  # 5 min for long generations
                    write=10.0,
                    pool=30.0
                ),
                http2=True  # Enable HTTP/2 for multiplexing
            )
            cls._client = AsyncOpenAI(http_client=http_client)
        return cls._client

    @classmethod
    async def close(cls):
        if cls._client:
            await cls._client.close()
            cls._client = None
```

### Stream Connection Leak Fix

When streaming, connections are NOT returned to the pool until the stream is fully consumed or explicitly closed:

```python
# WRONG: Connection leak with streaming
stream = await client.chat.completions.create(
    model="gpt-5-mini",
    messages=messages,
    stream=True
)
# If you break early or exception occurs, connection leaks

# CORRECT: Use context manager or ensure cleanup
async with await client.chat.completions.create(
    model="gpt-5-mini",
    messages=messages,
    stream=True
) as stream:
    async for chunk in stream:
        process(chunk)
# Connection properly returned to pool on exit

# Or manually close:
try:
    stream = await client.chat.completions.create(...)
    async for chunk in stream:
        process(chunk)
finally:
    await stream.response.aclose()  # ensure connection return
```

### Expected Improvement

- **Connection reuse**: Eliminates TCP+TLS handshake overhead (~100-200ms per request)
- **HTTP/2**: Multiplexes requests over a single connection
- **Higher concurrency**: Custom pool allows more simultaneous requests
- **Avoid timeouts**: Proper pool timeout prevents hanging when all connections are in use

### Monitoring

```python
import httpx
import logging

# Log request/response for debugging
def log_request(request):
    logging.debug(f"Request: {request.method} {request.url}")

def log_response(response):
    logging.debug(f"Response: {response.status_code} ({response.elapsed.total_seconds():.2f}s)")

http_client = httpx.AsyncClient(
    limits=custom_limits,
    event_hooks={
        "request": [log_request],
        "response": [log_response]
    }
)
```

---

## Summary: Priority Matrix for Podcast Pipeline

| Optimization | Impact | Effort | Priority | Available for gpt-5-mini/5.4? |
|---|---|---|---|---|
| **Prompt Caching** | HIGH (50% input cost, latency) | LOW (restructure prompts) | P0 | Yes |
| **reasoning_effort** | HIGH (3-10x latency) | LOW (one parameter) | P0 | Yes |
| **Structured Outputs** | MEDIUM (eliminates parse errors) | LOW (add schema) | P0 | Yes |
| **Connection Pooling** | MEDIUM (100-200ms/req) | LOW (singleton client) | P1 | Yes |
| **Streaming + TTS Pipeline** | HIGH (perceived latency) | MEDIUM (pipeline refactor) | P1 | Yes |
| **Batch API** | HIGH (50% all tokens) | MEDIUM (async workflow) | P1 (non-RT jobs) | Yes |
| **Parallel Function Calling** | MEDIUM (research stage) | LOW (already default) | P2 | Yes |
| **TTS Chunking/Parallel** | MEDIUM (audio gen speed) | MEDIUM (chunking logic) | P2 | Yes |
| **max_completion_tokens** | LOW (cost guard) | LOW (one parameter) | P3 | Yes |
| **Predicted Outputs** | N/A | N/A | N/A | **No** (GPT-4.x only) |

### Quick Win Checklist

1. Set `reasoning_effort` to `"none"` or `"low"` for script generation tasks
2. Move all static prompt content to the beginning of messages
3. Add `prompt_cache_key` for workloads sharing common prefixes
4. Use Structured Outputs with Pydantic for all JSON responses
5. Create a singleton `AsyncOpenAI` with custom httpx pool
6. Use `stream_options: {"include_usage": true}` on all streaming calls
7. Evaluate Batch API for overnight/scheduled script generation
8. Set `prompt_cache_retention: "24h"` for high-volume workloads

---

## Sources

- [OpenAI Prompt Caching Guide](https://developers.openai.com/api/docs/guides/prompt-caching/)
- [OpenAI Prompt Caching 201 Cookbook](https://developers.openai.com/cookbook/examples/prompt_caching_201/) (Feb 2026)
- [OpenAI Predicted Outputs Guide](https://developers.openai.com/api/docs/guides/predicted-outputs/)
- [OpenAI Using GPT-5.4 Guide](https://developers.openai.com/api/docs/guides/latest-model/)
- [OpenAI Batch API Guide](https://developers.openai.com/api/docs/guides/batch/)
- [OpenAI Structured Outputs Guide](https://developers.openai.com/api/docs/guides/structured-outputs/)
- [OpenAI Latency Optimization Guide](https://developers.openai.com/api/docs/guides/latency-optimization/)
- [OpenAI Text-to-Speech Guide](https://developers.openai.com/api/docs/guides/text-to-speech/)
- [OpenAI Function Calling Guide](https://developers.openai.com/api/docs/guides/function-calling/)
- [Azure OpenAI Prompt Caching](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/prompt-caching)
- [Azure OpenAI Predicted Outputs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/predicted-outputs)
- [HTTPX Async Client Docs](https://www.python-httpx.org/async/)
- [HTTPX Advanced Client Configuration](https://www.python-httpx.org/advanced/clients/)
- [OpenAI Python SDK - httpx connection pool issue #763](https://github.com/openai/openai-python/issues/763)
- [Artificial Analysis - GPT-5-mini benchmarks](https://artificialanalysis.ai/models/gpt-5-mini-medium/providers)
- [Artificial Analysis - GPT-5.4-mini benchmarks](https://artificialanalysis.ai/models/gpt-5-4-mini/providers)
- [OpenAI GPT-5.4 Announcement](https://openai.com/index/introducing-gpt-5-4/)
- [NxCode GPT-5.4 Complete Guide](https://www.nxcode.io/resources/news/gpt-5-4-complete-guide-features-pricing-models-2026)
- [OpenAI Community - Caching statistics for GPT-5 models](https://community.openai.com/t/caching-is-borked-for-gpt-5-models/1359574)
- [OpenAI Community - Cache rate drop with Responses API](https://community.openai.com/t/caching-rate-drop-after-switching-to-responses-api/1370788)
- [Inworld TTS Benchmarks 2026](https://inworld.ai/resources/best-voice-ai-tts-apis-for-real-time-voice-agents-2026-benchmarks)
