> **Research Date**: April 2026. Verify current API documentation before acting on specific parameters or pricing.

# Anthropic Claude API Optimization -- Deep Research Report

**Date**: 2026-04-02
**Model in scope**: claude-sonnet-4-6 (Pro/Ultimate tier podcast generation)
**Current pricing**: $3/MTok input, $15/MTok output

---

## Table of Contents

1. [Prompt Caching](#1-prompt-caching)
2. [Batch API](#2-batch-api)
3. [Extended Thinking](#3-extended-thinking)
4. [Streaming](#4-streaming)
5. [Token Efficiency](#5-token-efficiency)
6. [Tool Use for Structured Output](#6-tool-use-for-structured-output)
7. [System Prompt Best Practices](#7-system-prompt-best-practices)
8. [Model Selection](#8-model-selection)
9. [Beta Headers](#9-beta-headers)
10. [Combined Optimization Strategy for KitesForU](#10-combined-optimization-strategy-for-kitesforu)

---

## 1. Prompt Caching

### How It Works

Prompt caching stores a hash of the **prefix** of your request (tools + system + messages, in that order) and reuses the pre-computed KV cache on subsequent requests. The system checks from your breakpoint backward, looking for a matching prefix hash from a prior request.

### Two Modes: Automatic and Explicit

**Automatic caching** (recommended for most cases): Add `cache_control` at the top-level request. The breakpoint automatically moves to the last cacheable block in each request. No need to update markers as conversation grows.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    cache_control={"type": "ephemeral"},  # automatic caching
    system="You are a podcast script writer...",
    messages=[{"role": "user", "content": "Write a segment about..."}],
)
```

**Explicit (block-level) caching**: Place `cache_control` on specific content blocks for fine-grained control. Max 4 breakpoints.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    system=[
        {
            "type": "text",
            "text": "You are a podcast script writer for KitesForU...<long system prompt>",
            "cache_control": {"type": "ephemeral"}  # Breakpoint 1: system prompt
        }
    ],
    tools=[
        {
            "name": "format_script",
            "description": "Format podcast script as structured JSON",
            "input_schema": {
                "type": "object",
                "properties": {
                    "segments": {"type": "array", "items": {"type": "object"}},
                },
            },
            "cache_control": {"type": "ephemeral"}  # Breakpoint 2: tool defs
        }
    ],
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "<research_context>...50,000 tokens of research...</research_context>",
                    "cache_control": {"type": "ephemeral"}  # Breakpoint 3: research docs
                }
            ]
        },
        {
            "role": "assistant",
            "content": "I'll analyze this research..."
        },
        {
            "role": "user",
            "content": "Now write the final script for segment 3."
            # No cache_control here -- this is the dynamic part
        }
    ],
)
```

### Cache Breakpoint Rules

| Rule | Detail |
|------|--------|
| Max breakpoints | 4 per request |
| Minimum cacheable tokens (Sonnet) | 1,024 tokens |
| Minimum cacheable tokens (Haiku) | 2,048 tokens |
| Default TTL | 5 minutes (refreshed on each cache hit) |
| Extended TTL | 1 hour (at 2x base input cost for cache write) |
| Cache write cost | 1.25x base input price (5-min TTL) or 2x (1-hour TTL) |
| Cache read cost | 0.1x base input price (90% discount) |

### TTL Options

```python
# Default 5-minute TTL
{"cache_control": {"type": "ephemeral"}}

# Extended 1-hour TTL (good for batch processing)
{"cache_control": {"type": "ephemeral", "ttl": "1h"}}
```

### Strategic Breakpoint Placement (4 breakpoints max)

For podcast generation pipeline:
1. **Tool definitions** -- stable across all requests
2. **System prompt** -- stable per pipeline stage
3. **Research context / source material** -- stable per podcast episode
4. **Conversation history** -- grows with multi-turn, use automatic caching here

### Cacheable Locations

| Location | Cacheable? |
|----------|-----------|
| `tools` array | Yes -- place `cache_control` on the last tool |
| `system` messages | Yes -- on any system content block |
| `messages` (user) | Yes -- on user message content blocks |
| `messages` (assistant) | No -- cannot place cache_control on assistant messages |
| Tool definitions | Yes |
| Images/PDFs in messages | Yes (but changing images breaks cache) |

### Verifying Cache Hits

```python
response = client.messages.create(...)

# Check response usage
print(response.usage)
# Usage(
#   input_tokens=50,               # tokens NOT from cache
#   cache_creation_input_tokens=12500,  # tokens written to cache (first request)
#   cache_read_input_tokens=12500,      # tokens read from cache (subsequent)
#   output_tokens=500
# )
```

**Interpretation**:
- `cache_creation_input_tokens > 0`: Cache was written (first call or cache expired)
- `cache_read_input_tokens > 0`: Cache hit (reused from prior request)
- `input_tokens`: Only the non-cached tokens

### Pricing Impact (Sonnet 4.6 at $3/MTok input)

| Scenario | Per-MTok Cost | vs Base |
|----------|--------------|---------|
| Standard input | $3.00 | -- |
| Cache write (5-min TTL) | $3.75 | +25% (first call) |
| Cache write (1-hour TTL) | $6.00 | +100% (first call) |
| Cache read | $0.30 | **-90%** |

**Example**: 50K token system prompt + research context, 100 calls/hour:
- Without caching: 100 x 50K x $3/MTok = **$15.00/hr**
- With caching: 1 write ($0.19) + 99 reads (99 x $0.015) = **$1.67/hr** = **89% savings**

### Limitations

- Changing ANY block at or before a breakpoint invalidates the cache (different hash)
- Cache is per-organization, not shared across API keys
- Thinking blocks cannot have explicit `cache_control` but get cached automatically during tool use flows
- No manual cache invalidation -- relies on TTL expiry

---

## 2. Batch API (Message Batches)

### How It Works

Submit up to 10,000 requests as a batch. Processed asynchronously within 24 hours (typically 30-60 minutes). **50% discount** on all tokens (input and output).

### Creating a Batch

```python
import anthropic

client = anthropic.Anthropic()

# Prepare batch requests
batch_requests = [
    {
        "custom_id": f"podcast-segment-{i}",
        "params": {
            "model": "claude-sonnet-4-6",
            "max_tokens": 8192,
            "system": "You are a podcast script writer...",
            "messages": [
                {"role": "user", "content": f"Write podcast segment about: {topic}"}
            ]
        }
    }
    for i, topic in enumerate(segment_topics)
]

# Submit batch
batch = client.messages.batches.create(requests=batch_requests)
print(f"Batch ID: {batch.id}")
print(f"Status: {batch.processing_status}")  # "in_progress"
```

### Polling for Results

```python
import time

def wait_for_batch(client, batch_id, poll_interval=30):
    while True:
        batch = client.messages.batches.retrieve(batch_id)
        print(f"Status: {batch.processing_status}")
        print(f"Succeeded: {batch.request_counts.succeeded}")
        print(f"Processing: {batch.request_counts.processing}")

        if batch.processing_status == "ended":
            return batch
        time.sleep(poll_interval)

batch_result = wait_for_batch(client, batch.id)
```

### Retrieving Results

```python
for result in client.messages.batches.results(batch_result.id):
    print(f"Request: {result.custom_id}")
    if result.result.type == "succeeded":
        content = result.result.message.content[0].text
        print(f"Content: {content[:200]}...")
    elif result.result.type == "errored":
        print(f"Error: {result.result.error}")
    elif result.result.type == "canceled":
        print("Canceled")
```

### Stacking Batch + Prompt Caching

Yes, the discounts **stack**. Use `cache_control` inside batch request params:

```python
batch_requests = [
    {
        "custom_id": f"segment-{i}",
        "params": {
            "model": "claude-sonnet-4-6",
            "max_tokens": 8192,
            "system": [
                {
                    "type": "text",
                    "text": SHARED_SYSTEM_PROMPT,  # same across all batch items
                    "cache_control": {"type": "ephemeral", "ttl": "1h"}
                    # Use 1h TTL because batch processing > 5 minutes
                }
            ],
            "messages": [
                {"role": "user", "content": unique_prompt_per_segment}
            ]
        }
    }
    for i, unique_prompt_per_segment in enumerate(prompts)
]
```

### Stacked Pricing (Sonnet 4.6)

| Optimization | Input Cost/MTok | Output Cost/MTok | Combined Savings |
|-------------|----------------|-----------------|-----------------|
| Standard | $3.00 | $15.00 | 0% |
| Batch only | $1.50 | $7.50 | 50% |
| Cache read only | $0.30 | $15.00 | ~45% (input heavy) |
| Batch + Cache read | **$0.15** | **$7.50** | **~95% on input** |

**Note**: Cache hit rates in batches are 30-98% depending on traffic patterns since requests process asynchronously and concurrently. Using 1-hour TTL improves cache hits for batches.

### Batch Limitations

- Max 10,000 requests per batch
- 24-hour processing window (usually < 1 hour)
- No real-time streaming -- results only after completion
- No per-item progress tracking or cancellation
- All active models supported
- Extended output (300K tokens) supported via `output-300k-2026-03-24` beta header

### When to Use for KitesForU

- Research stage (multiple research queries per episode)
- Bulk segment generation (all segments of an episode)
- NOT for real-time/interactive generation where user is waiting

---

## 3. Extended Thinking

### How It Works

Extended thinking allows Claude to perform internal chain-of-thought reasoning before producing the final response. Thinking tokens are billed as **output tokens** ($15/MTok on Sonnet 4.6).

### Two Approaches (as of April 2026)

**1. Manual extended thinking** (Sonnet 4.6):
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # max tokens for reasoning
    },
    messages=[{"role": "user", "content": "Synthesize these research findings..."}],
)

# Response contains thinking + text blocks
for block in response.content:
    if block.type == "thinking":
        print(f"Thinking: {block.thinking[:200]}...")
    elif block.type == "text":
        print(f"Response: {block.text}")
```

**2. Adaptive thinking** (Opus 4.6 -- and Sonnet 4.6 also supports it):
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "adaptive"},  # Claude decides when/how much to think
    messages=[{"role": "user", "content": "Complex synthesis task..."}],
)
```

### Budget Tokens Parameter

| Setting | Detail |
|---------|--------|
| Minimum budget | 1,024 tokens |
| Max budget (Sonnet 4.6) | Up to 64K tokens |
| Max budget (Opus 4.6) | Up to 128K tokens |
| Constraint | `budget_tokens` must be < `max_tokens` |
| Interleaved thinking exception | `budget_tokens` can exceed `max_tokens` |
| Note | Budget is a *target*, not strict limit |

### When to Use for KitesForU Pipeline

| Stage | Extended Thinking? | Reasoning |
|-------|-------------------|-----------|
| Research/Planning | Yes (8K-16K budget) | Complex multi-source synthesis benefits |
| Assimilator | Yes (16K+ budget) | Deep reasoning for content organization |
| Script Writing | Optional (4K budget) | Creative synthesis, moderate benefit |
| Simple formatting | No | Unnecessary cost, adds latency |
| Audio script prep | No | Straightforward transformation |

### Cost Impact

Extended thinking tokens are billed as output tokens:
- 10K thinking tokens = 10K x $15/MTok = **$0.15 per request**
- 16K thinking tokens = 16K x $15/MTok = **$0.24 per request**
- For 100 requests/day: +$15-24/day

### Latency Impact

Thinking adds significant latency. Expect:
- 4K budget: +2-5 seconds
- 10K budget: +5-15 seconds
- 16K budget: +10-30 seconds

**Recommendation**: Use selectively. Enable for assimilator/synthesis stages. Skip for formatting/scripting stages.

### Streaming Requirement

When `max_tokens > 21,333`, SDKs require streaming to avoid HTTP timeouts. Use:
```python
# If you don't need incremental events:
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=32000,
    thinking={"type": "enabled", "budget_tokens": 16000},
    messages=[...],
) as stream:
    message = stream.get_final_message()  # blocks until complete
```

### Caching with Thinking

- Thinking blocks **cannot** have explicit `cache_control`
- They **are** cached automatically when passed back in tool use flows
- When a non-tool-result user message follows thinking blocks, all prior thinking blocks are stripped from context

---

## 4. Streaming

### How It Works

Set `stream=True` (or use `client.messages.stream()`) to receive Server-Sent Events (SSE) as the response generates.

### Time-to-First-Token (TTFT) Improvement

Streaming does NOT change TTFT -- the model still needs the same time to begin generating. What streaming provides:
- **Perceived latency reduction**: User sees output within milliseconds of first token generation
- **Parallel processing**: Start processing content blocks while the model is still generating
- **No HTTP timeout risk**: Essential for long responses (especially with thinking)

Typical TTFT for Sonnet 4.6 (non-thinking): 0.5-2 seconds
Typical TTFT for Sonnet 4.6 (with thinking): 2-15+ seconds (thinking happens first)

### SSE Event Flow

```
message_start         -> Message object with empty content
content_block_start   -> New content block (thinking, text, tool_use)
content_block_delta   -> Incremental content (thinking_delta, text_delta, input_json_delta)
content_block_stop    -> Block complete
message_delta         -> Top-level message changes (stop_reason, usage)
message_stop          -> Stream complete
ping                  -> Keepalive (ignore)
```

### Python SDK Implementation

```python
import anthropic

client = anthropic.Anthropic()

# Simple text streaming
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    system="You are a podcast script writer...",
    messages=[{"role": "user", "content": "Write a podcast segment about AI safety"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# Full usage after stream completes
final = stream.get_final_message()
print(f"\nUsage: {final.usage}")
```

### Event-Level Handling (for pipeline processing)

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": "..."}],
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            block_type = event.content_block.type
            print(f"Starting {block_type} block")

        elif event.type == "content_block_delta":
            if event.delta.type == "text_delta":
                # Process text incrementally
                process_chunk(event.delta.text)
            elif event.delta.type == "thinking_delta":
                # Optionally log thinking
                pass
            elif event.delta.type == "input_json_delta":
                # Tool input being streamed
                accumulate_json(event.delta.partial_json)

        elif event.type == "message_delta":
            # Contains stop_reason and cumulative usage
            print(f"Stop: {event.delta.stop_reason}")
            print(f"Output tokens: {event.usage.output_tokens}")
```

### Streaming with Extended Thinking

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},
    messages=[{"role": "user", "content": "Complex analysis task..."}],
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            if event.content_block.type == "thinking":
                print("Thinking started...")
            elif event.content_block.type == "text":
                print("\nResponse:")
        elif event.type == "content_block_delta":
            if event.delta.type == "thinking_delta":
                pass  # optionally show thinking progress
            elif event.delta.type == "text_delta":
                print(event.delta.text, end="", flush=True)
```

### Fine-Grained Tool Streaming

For reduced latency on tool input streaming, use the beta header:
```python
response = client.beta.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    betas=["fine-grained-tool-streaming-2025-05-14"],
    tools=tools,
    messages=[...],
    stream=True,
)
```

**Warning**: Fine-grained streaming may deliver partial/invalid JSON. Add error handling.

### When to Use for KitesForU

- **Always** for user-facing generation (perceived speed)
- **Required** when `max_tokens > 21,333` with thinking enabled
- **Optional** for background/batch processing (use `.get_final_message()` for simplicity)

---

## 5. Token Efficiency

### Reducing Input Tokens

1. **Use system prompts for stable instructions** -- they are cached more efficiently
2. **Front-load static content** -- tools, system prompt, then static user context, then dynamic content last
3. **Trim conversation history** -- summarize older turns instead of sending full history
4. **Use XML tags for structure** -- Claude processes structured XML more efficiently than verbose natural language framing

```python
# BEFORE: Verbose
messages = [{"role": "user", "content": """
I have some research that I'd like you to analyze. The research is about artificial intelligence
and specifically covers the topic of neural networks. Here is the full text of the research paper
that I would like you to read and then write a podcast script about:

[50,000 tokens of research]

Please write a podcast script based on this research. The script should be engaging and informative.
"""}]

# AFTER: Efficient with XML tags
messages = [{"role": "user", "content": """
<research>
[50,000 tokens of research]
</research>

Write a podcast script based on the above research.
"""}]
```

### max_tokens Optimization

- **Lower `max_tokens` does NOT reduce latency** -- the model generates until it hits a stop condition or the limit
- **Set it to a reasonable ceiling** for your use case to prevent runaway generation
- A request with `max_tokens=4096` that generates 500 tokens costs the same as `max_tokens=100000` generating 500 tokens
- **Exception**: Extended thinking -- `budget_tokens` sets the thinking ceiling and can save cost if set appropriately

### stop_sequences for Early Termination

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    stop_sequences=["</script>", "END_SEGMENT"],  # Stop as soon as script is complete
    messages=[{"role": "user", "content": "Write a podcast script ending with </script>"}],
)
# Saves output tokens by stopping generation early
```

### System vs User Prompt for Caching

| Content Type | Where to Put | Why |
|-------------|-------------|-----|
| Pipeline instructions | `system` | Cached separately, stable across episodes |
| Persona/voice guidelines | `system` | Rarely changes |
| Research content | First `user` message block | Cached per episode, changes per episode |
| Specific task instruction | Last `user` message | Dynamic, not cached |

### Token-Efficient Tool Use (Beta)

Reduces tool-related token overhead:
```python
response = client.beta.messages.create(
    model="claude-sonnet-4-6",
    betas=["token-efficient-tools-2025-02-19"],
    tools=tools,
    messages=[...],
)
```

This beta reduces the system prompt tokens injected for tool use support.

---

## 6. Tool Use for Structured Output

### Tool Use vs JSON in Prompt

| Aspect | Tool Use | "Return JSON" in prompt |
|--------|----------|----------------------|
| Schema enforcement | Defined via `input_schema` | Hope-based |
| Output reliability | High -- constrained to schema | May include markdown, extra text |
| Token efficiency | 50-80% fewer output tokens | Verbose, may include explanation |
| Cacheability | Tool defs are cacheable | N/A |
| Streaming | Streams as `input_json_delta` | Streams as `text_delta` |
| Latency | Comparable | Comparable |

### Implementation

```python
tools = [
    {
        "name": "generate_podcast_segment",
        "description": "Generate a structured podcast segment with host dialogue",
        "input_schema": {
            "type": "object",
            "properties": {
                "segment_title": {"type": "string"},
                "host_lines": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "speaker": {"type": "string", "enum": ["host", "expert"]},
                            "text": {"type": "string"},
                            "tone": {"type": "string", "enum": ["conversational", "serious", "excited"]},
                        },
                        "required": ["speaker", "text", "tone"]
                    }
                },
                "estimated_duration_seconds": {"type": "integer"},
            },
            "required": ["segment_title", "host_lines", "estimated_duration_seconds"]
        },
        "cache_control": {"type": "ephemeral"}  # Cache the tool definition
    }
]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    tools=tools,
    tool_choice={"type": "tool", "name": "generate_podcast_segment"},  # Force tool use
    messages=[{"role": "user", "content": "Generate podcast segment about..."}],
)

# Extract structured result
for block in response.content:
    if block.type == "tool_use":
        segment_data = block.input  # Already a dict, no parsing needed
        print(segment_data["segment_title"])
```

### Structured Outputs (GA since Jan 2026)

For guaranteed schema conformance without tool use:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    output_config={
        "format": {
            "type": "json_schema",
            "json_schema": {
                "name": "podcast_segment",
                "schema": {
                    "type": "object",
                    "properties": {
                        "title": {"type": "string"},
                        "lines": {"type": "array", "items": {"type": "object"}},
                    }
                }
            }
        }
    },
    messages=[{"role": "user", "content": "Generate podcast segment..."}],
)
```

### Caching Tool Definitions

Tool definitions ARE cacheable. Place `cache_control` on the last tool in the array:

```python
tools = [
    {"name": "tool_1", "description": "...", "input_schema": {...}},
    {"name": "tool_2", "description": "...", "input_schema": {...}},
    {
        "name": "tool_3",
        "description": "...",
        "input_schema": {...},
        "cache_control": {"type": "ephemeral"}  # Caches ALL tools (prefix-based)
    },
]
```

### Tool Use Token Overhead

Each model injects a system prompt when tools are provided. Approximate overhead per model:

| Model | Tool use system prompt tokens |
|-------|------------------------------|
| Claude Sonnet 4.6 | ~350-500 tokens |
| Claude Haiku 4.5 | ~350-500 tokens |

Use `token-efficient-tools-2025-02-19` beta to reduce this overhead.

---

## 7. System Prompt Best Practices

### Optimal Structure for Caching

```python
# OPTIMAL STRUCTURE for caching
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    # 1. Tools (cached first in prefix)
    tools=[
        {
            "name": "format_output",
            "description": "...",
            "input_schema": {...},
            "cache_control": {"type": "ephemeral"}  # BP1
        }
    ],
    # 2. System prompt (cached second)
    system=[
        {
            "type": "text",
            "text": """You are an expert podcast script writer for KitesForU.

<voice_guidelines>
- Conversational, engaging tone
- Use analogies and examples
- Vary sentence length
- Include natural transitions
</voice_guidelines>

<format_rules>
- Each segment: 3-5 minutes of dialogue
- Alternate between host perspectives
- Include listener engagement hooks
</format_rules>""",
            "cache_control": {"type": "ephemeral"}  # BP2
        }
    ],
    # 3. Messages with static context cached, dynamic uncached
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "<research_material>...large research content...</research_material>",
                    "cache_control": {"type": "ephemeral"}  # BP3
                },
                {
                    "type": "text",
                    "text": "Write segment 3 about the regulatory implications."
                    # No cache_control -- this is dynamic
                }
            ]
        }
    ],
)
```

### Cache Prefix Order

The cache prefix is evaluated in this exact order:
1. `tools` array
2. `system` messages
3. `messages` array (in order)

Everything BEFORE a breakpoint is included in the hash. Put stable content first.

### What Goes Where

| Content | Location | Cached? |
|---------|----------|---------|
| Tool definitions | `tools` | Yes (BP1) |
| Role/persona instructions | `system` | Yes (BP2) |
| Format guidelines | `system` | Yes (BP2) |
| Episode research material | First `user` block | Yes (BP3) |
| Specific segment instructions | Last `user` block | No (dynamic) |
| Previous segment outputs | Earlier `messages` | Automatic caching |

### Automatic Caching (Simplest Approach)

If you don't want to manage breakpoints manually:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    cache_control={"type": "ephemeral"},  # Top-level: auto breakpoint on last block
    system="Your stable system prompt here...",
    messages=[
        {"role": "user", "content": "Long context + question"}
    ],
)
```

The system automatically places one breakpoint on the last cacheable block. Combined with explicit breakpoints, it uses one of the 4 available slots.

---

## 8. Model Selection

### Current Model Lineup (April 2026)

| Model | Input $/MTok | Output $/MTok | Context | Max Output | Latency | TTFT |
|-------|-------------|--------------|---------|------------|---------|------|
| Claude Opus 4.6 | $5 | $25 | 1M | 128K | Moderate | Slowest |
| **Claude Sonnet 4.6** | **$3** | **$15** | **1M** | **64K** | **Fast** | **Fast** |
| Claude Haiku 4.5 | $1 | $5 | 200K | 64K | Fastest | Fastest |

### Stage-by-Stage Recommendation for Podcast Pipeline

| Pipeline Stage | Current (Sonnet 4.6) | Recommended | Savings | Quality Impact |
|---------------|---------------------|-------------|---------|---------------|
| Research query generation | Sonnet 4.6 | **Haiku 4.5** | 67% | Minimal -- simple query formatting |
| Research synthesis | Sonnet 4.6 | Sonnet 4.6 | 0% | Keep -- needs quality reasoning |
| Outline/planning | Sonnet 4.6 | Sonnet 4.6 | 0% | Keep -- structural reasoning |
| Script writing | Sonnet 4.6 | Sonnet 4.6 | 0% | Core quality task |
| Assimilator (deep synthesis) | Sonnet 4.6 | Sonnet 4.6 + thinking | +cost | Quality improvement |
| Script formatting/cleanup | Sonnet 4.6 | **Haiku 4.5** | 67% | Minimal -- mechanical task |
| SSML/audio markup | Sonnet 4.6 | **Haiku 4.5** | 67% | Minimal -- template-based |

### Quality Tradeoffs

- **Haiku 4.5** matches previous Sonnet 4 performance at 1/3 the cost
- For tasks requiring creativity, nuance, or complex reasoning: keep Sonnet 4.6
- For mechanical, format-based, or simple extraction tasks: Haiku 4.5 is sufficient
- **Opus 4.6** generally not needed unless facing quality issues on specific complex tasks

### Latency Comparison (approximate, non-thinking)

| Metric | Opus 4.6 | Sonnet 4.6 | Haiku 4.5 |
|--------|----------|-----------|-----------|
| TTFT | 1-3s | 0.5-2s | 0.2-1s |
| Tokens/sec (output) | ~60-80 | ~80-120 | ~150-200+ |

---

## 9. Beta Headers

### Currently Available Beta Features (April 2026)

| Beta Header | Feature | Relevant? |
|-------------|---------|-----------|
| `output-300k-2026-03-24` | 300K output tokens (Batch API only) | Yes -- long scripts in batch |
| `interleaved-thinking-2025-05-14` | Think between tool calls | Yes -- multi-step tool workflows |
| `token-efficient-tools-2025-02-19` | Reduced tool system prompt tokens | Yes -- always use with tools |
| `fine-grained-tool-streaming-2025-05-14` | Stream tool inputs without buffering | Maybe -- faster tool streaming |
| `prompt-caching-scope-2026-01-05` | Enhanced cache scoping | Investigate |
| `advanced-tool-use-2025-11-20` | Enhanced tool capabilities | Yes |
| `files-api-2025-04-14` | Files API | Maybe -- for research documents |
| `computer-use-2025-01-24` | Computer use | No |
| `context-management-2025-06-27` | Memory tool | No |

### No Longer Needed (GA or Obsolete)

| Feature | Notes |
|---------|-------|
| `prompt-caching-2024-07-31` | **No longer needed** -- prompt caching is GA |
| `max-tokens-3-5-sonnet-2024-07-15` | **Obsolete** -- old model |
| `message-batches-2024-09-24` | **No longer needed** -- Batches API is GA |
| `structured-outputs-2025-11-13` | **No longer needed** -- GA since Jan 2026 |

### Implementation

```python
import anthropic

client = anthropic.Anthropic()

# Using beta features via SDK
response = client.beta.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    betas=[
        "token-efficient-tools-2025-02-19",
        "interleaved-thinking-2025-05-14",
    ],
    tools=tools,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[...],
)

# Or via raw headers
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    extra_headers={
        "anthropic-beta": "token-efficient-tools-2025-02-19,interleaved-thinking-2025-05-14"
    },
    tools=tools,
    messages=[...],
)
```

---

## 10. Combined Optimization Strategy for KitesForU

### Immediate Wins (implement first)

| Optimization | Expected Savings | Effort |
|-------------|-----------------|--------|
| Add `cache_control` to system prompts + research context | 60-90% on input tokens | Low |
| Use Haiku 4.5 for formatting/SSML stages | 67% on those stages | Low |
| Add `token-efficient-tools-2025-02-19` beta header | ~10-15% on tool overhead | Trivial |
| Enable streaming for all user-facing calls | Better UX (perceived speed) | Low |
| Use `stop_sequences` where applicable | 5-15% output savings | Low |

### Medium-Term (implement after measuring baseline)

| Optimization | Expected Savings | Effort |
|-------------|-----------------|--------|
| Batch API for research stage | 50% on research calls | Medium |
| Extended thinking for assimilator | Quality improvement (not savings) | Medium |
| Structured outputs for script formatting | Reliability + token savings | Medium |
| Tool use instead of "return JSON" prompts | 50-80% output token savings | Medium |

### Architecture Pattern

```
Episode Generation Pipeline (optimized):

1. Research Stage (Batch API + Cache)
   Model: Haiku 4.5 (query gen) -> Sonnet 4.6 (synthesis)
   Cache: System prompt + tool defs (stable)
   Batch: All research queries submitted as batch
   Savings: ~70% vs current

2. Planning Stage (Cache)
   Model: Sonnet 4.6
   Cache: System prompt + research context
   Thinking: Enabled (8K budget)
   Savings: ~60% on input via caching

3. Script Generation (Cache + Streaming)
   Model: Sonnet 4.6
   Cache: System prompt + research + outline
   Tool use: Structured segment output
   Streaming: Enabled (user-facing)
   Savings: ~50% via caching + structured output

4. Formatting Stage (Haiku + Cache)
   Model: Haiku 4.5
   Cache: Format templates
   Savings: ~80% vs current (cheaper model + caching)

5. Audio Prep (Haiku)
   Model: Haiku 4.5
   Simple SSML wrapping
   Savings: ~67% (cheaper model)
```

### Estimated Total Cost Impact

Assuming current pipeline processes ~50K input tokens and ~10K output tokens per stage, 5 stages per episode:

| Metric | Current | Optimized | Savings |
|--------|---------|-----------|---------|
| Input cost/episode | ~$0.75 | ~$0.15 | **80%** |
| Output cost/episode | ~$0.75 | ~$0.55 | **27%** |
| Total/episode | ~$1.50 | ~$0.70 | **53%** |
| Monthly (100 episodes) | ~$150 | ~$70 | **$80/mo** |

The biggest wins come from prompt caching (input) and model selection (Haiku for simple stages). Extended thinking adds cost but improves quality where it matters.

---

## Sources

- [Anthropic Prompt Caching Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Anthropic Batch Processing Docs](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Anthropic Extended Thinking Docs](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)
- [Anthropic Streaming Docs](https://platform.claude.com/docs/en/build-with-claude/streaming)
- [Anthropic Tool Use Docs](https://platform.claude.com/docs/en/build-with-claude/tool-use/overview)
- [Anthropic Reducing Latency Guide](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-latency)
- [Anthropic Beta Headers Docs](https://platform.claude.com/docs/en/api/beta-headers)
- [Anthropic Models Overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Anthropic Release Notes](https://platform.claude.com/docs/en/release-notes/overview)
- [Anthropic Batch Cookbook](https://platform.claude.com/cookbook/misc-batch-processing)
- [Anthropic Caching Cookbook](https://platform.claude.com/cookbook/misc-prompt-caching)
- [Claude API Pricing 2026 -- MetaCTO](https://www.metacto.com/blogs/anthropic-api-pricing-a-full-breakdown-of-costs-and-integration)
- [Anthropic Batch API in Production -- Dotzlaw](https://dotzlaw.com/insights/obsidian-notes-02/)
