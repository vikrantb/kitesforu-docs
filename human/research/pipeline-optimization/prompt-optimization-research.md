> **Research Date**: April 2026. Verify current API documentation before acting on specific parameters or pricing.

# Prompt Engineering Optimization for LLM Content Generation Pipelines

Deep research report covering 15 areas of prompt optimization for reducing latency and cost in the KitesForU podcast generation pipeline.

**Date:** 2026-04-02
**Pipeline context:** Research -> Synthesis -> Planning -> Script Generation -> TTS

---

## Table of Contents

1. [Token Reduction Techniques](#1-token-reduction-techniques)
2. [System Prompt Optimization](#2-system-prompt-optimization)
3. [Structured Output vs Free-form](#3-structured-output-vs-free-form)
4. [Chain-of-Thought Control](#4-chain-of-thought-control)
5. [Output Length Control](#5-output-length-control)
6. [Prompt Decomposition](#6-prompt-decomposition)
7. [Few-Shot Learning Efficiency](#7-few-shot-learning-efficiency)
8. [Template Optimization](#8-template-optimization)
9. [Research Synthesis Prompt](#9-research-synthesis-prompt)
10. [Script Generation Prompt](#10-script-generation-prompt)
11. [Multi-turn vs Single-turn](#11-multi-turn-vs-single-turn)
12. [Temperature and Quality](#12-temperature-and-quality)
13. [Stop Sequences](#13-stop-sequences)
14. [Prompt Caching Architecture](#14-prompt-caching-architecture)
15. [Parallel Generation](#15-parallel-generation)

---

## 1. Token Reduction Techniques

### The Core Insight

Input token reductions alone produce modest latency improvements (1-5% for trimming 50% of input tokens) because **output tokens dominate both cost and latency**. Output tokens are 4-8x more expensive than input tokens across major providers, and each output token adds approximately 1ms of sequential latency. The real wins come from (a) making input tokens cacheable and (b) reducing output tokens.

### Concrete Techniques

**A. Minify JSON in prompts**
Raw research JSON with pretty-printing wastes 30-40% of tokens on whitespace.

```python
# BEFORE: 8000 tokens
research_json = json.dumps(research_results, indent=2)

# AFTER: ~5200 tokens (35% reduction)
research_json = json.dumps(research_results, separators=(',', ':'))
```

- Token savings: **30-40%** on JSON payloads
- Quality impact: None -- LLMs parse minified JSON equally well
- Applicable to: research_results passed to assimilator, outline passed to script worker

**B. Remove redundant instructions**
The current `dialogue_generation.yaml` contains the "no greeting" instruction THREE times (system prompt, user template top, user template mid-section with emoji block). LLMs follow instructions stated once clearly.

```yaml
# BEFORE: 3 repetitions of no-greeting rule (~250 tokens)
# AFTER: 1 clear statement in system prompt (~80 tokens)
# Savings: ~170 tokens per script generation call
```

- Token savings: **5-15%** depending on how much duplication exists
- Quality impact: Neutral to positive (less noise for the model to parse)
- Risk: Must verify the single instruction is followed. If compliance drops, add ONE reinforcement in user template.

**C. Compress static tables to compact form**
The banned_patterns table in `density_guidelines.yaml` uses a markdown table with headers. Compact alternatives:

```yaml
# BEFORE (full markdown table): ~180 tokens
## BANNED -> REQUIRED ALTERNATIVES
| Banned | Replace With |
|--------|--------------|
| "That's fascinating/interesting" | Follow-up question OR related fact |
...

# AFTER (compact pairs): ~100 tokens
## BANNED PHRASES (replace with specific content):
- "That's fascinating" -> follow-up question or related fact
- "Wow, really?" -> "So that means [implication]..."
- "A lot of people" -> specific number
- "Many studies show" -> cite specific study+year
```

- Token savings: **40-50%** on table-formatted content
- Quality impact: Neutral

**D. Use instruction IDs for fine-tuned models**
If fine-tuning is adopted, replace verbose instructions with short reference codes the model has learned:

```python
# BEFORE: 500 tokens of density guidelines
# AFTER: 5 tokens
{"instruction_id": "DENSITY_V2", "input": "..."}
```

- Token savings: **90%+** on instruction content
- Quality impact: Requires fine-tuning investment. Practical for high-volume stages.
- Recommendation: Defer until pipeline processes 10K+ jobs/month

**E. Semantic compression of research context**
Before sending raw research to the assimilator, apply extractive pre-processing:

```python
def compress_research(results: list[dict], max_tokens: int = 4000) -> str:
    """Extract only key-value facts from research results."""
    compressed = []
    for r in results:
        entry = {
            "src": r.get("source", "")[:50],  # truncate source name
            "facts": [],
        }
        # Extract sentences with numbers/stats (highest info density)
        for sentence in r.get("content", "").split(". "):
            if any(c.isdigit() for c in sentence) or "%" in sentence:
                entry["facts"].append(sentence.strip()[:200])
        if entry["facts"]:
            compressed.append(entry)
    return json.dumps(compressed, separators=(',', ':'))
```

- Token savings: **50-70%** on research payloads (4000-8000 -> 1500-3000 tokens)
- Quality impact: Moderate risk -- may lose qualitative insights. Use as pre-filter, not replacement.

### Summary Table

| Technique | Token Savings | Quality Risk | Implementation Effort |
|-----------|--------------|-------------|----------------------|
| Minify JSON | 30-40% | None | Trivial |
| Remove duplicated instructions | 5-15% | None | Low |
| Compact tables | 40-50% per table | None | Low |
| Instruction IDs (fine-tuning) | 90%+ | Requires training | High |
| Research pre-compression | 50-70% | Moderate | Medium |

---

## 2. System Prompt Optimization

### Optimal System Prompt Length

There is no single "optimal length" -- the key insight is **diminishing returns on instruction specificity**. Research and production experience show:

- **Under 200 tokens**: Too sparse for complex tasks. Model makes assumptions.
- **200-800 tokens**: Sweet spot for most tasks. Clear role, key rules, output format.
- **800-2000 tokens**: Acceptable if content is cacheable. Include few-shot examples, detailed rules.
- **2000+ tokens**: Diminishing returns on instruction quality. But if cacheable, cost impact is minimal.

The current `dialogue_generation.yaml` system prompt is approximately 80 tokens -- concise and effective. The cost is in the user_template which balloons with includes.

### Structuring for Prompt Caching

**The Static Prefix Pattern** is the single highest-impact optimization for the pipeline.

```
[SYSTEM: Immutable]           <-- Always cached (role, core rules)
[INCLUDES: Semi-static]       <-- Cached per content type (density rules, speaker rules, output format)
[FEW-SHOT: Fixed examples]    <-- Cached per use case
[DYNAMIC: Per-request]         <-- Never cached (topic, outline, research, language)
```

**Provider Requirements:**

| Provider | Min Tokens | Match Type | TTL | Discount |
|----------|-----------|------------|-----|----------|
| OpenAI | 1,024 | Exact prefix, 128-token increments | 5-10 min inactivity | 50% on cached input |
| Anthropic | Varies | Exact prefix (byte-level) | 5 min default | 90% on cached reads |

**Critical rule**: ANY difference in the prefix -- even a single character -- breaks the cache. This means:

1. No timestamps, UUIDs, or request-specific data in the prefix
2. Language instructions must come AFTER the cached prefix
3. Dynamic content type guidance must come AFTER static rules

### Current Pipeline Analysis

The current `PromptLoader.load()` builds prompts in this order:
1. System prompt (static -- good)
2. Includes resolved into variables (semi-static -- good)
3. Style overrides (per-style -- cacheable across same-style requests)
4. Content type rules (per-type -- cacheable across same-type requests)
5. User template rendered with variables (dynamic -- cannot cache)

**Problem**: The user template mixes static includes (`{speaker_rules}`, `{density_rules}`) with dynamic content (`{topic}`, `{research_summary}`). Since these render into a single user message, the entire user message is dynamic.

### Recommended Architecture Change

```python
def build_cacheable_prompt(self, prompt_path, variables, style, content_type):
    """Build prompt with maximum cacheable prefix."""
    parts = load_prompt_parts(prompt_path, variables, style, content_type)

    # STATIC PREFIX: system + all includes + examples
    # This is identical for all requests with same (style, content_type)
    static_system = parts["system"]
    static_includes = self._render_static_includes(parts)  # speaker_rules, density, format, etc.

    # DYNAMIC SUFFIX: topic, outline, research, language
    dynamic_user = self._render_dynamic_only(parts, variables)

    return {
        "system": static_system + "\n\n" + static_includes,  # CACHEABLE
        "user": dynamic_user,  # NOT CACHED
    }
```

- Token savings from caching: **50-90% cost reduction** on the static portion
- Latency improvement: **Up to 75% TTFT reduction** for cached prefixes
- Implementation: Requires restructuring `PromptLoader` to separate static from dynamic

### Cache Key Design

```python
cache_key = f"v{prompt_version}:{content_type}:{style}:{language_family}"
# Examples:
# "v2.0:educational:casual:en" -- all English casual educational podcasts share this prefix
# "v2.0:horror:storytelling:es" -- all Spanish horror storytelling share this
```

This means all requests for the same content_type + style + language family hit the same cached prefix. For a pipeline generating dozens of podcasts per hour, cache hit rates should exceed 80%.

---

## 3. Structured Output vs Free-form

### JSON Mode Variants and Their Tradeoffs

| Mode | Latency Impact | Token Impact | Reliability |
|------|---------------|-------------|-------------|
| No constraint | Baseline | Baseline | Low (requires parsing) |
| `response_format: {type: "json_object"}` | +5-10% first call, neutral after | Slightly fewer (no prose) | Medium (valid JSON, no schema guarantee) |
| `response_format: {type: "json_schema", schema: ...}` | +10-60s first call (schema compilation), neutral after caching | Schema tokens count as input | High (schema-compliant) |
| Function calling | +15-25% latency with explanations | Adds function definition tokens | High (typed output) |

### Key Findings

1. **First-call latency penalty is real**: Community reports show `json_schema` strict mode adding 10-60 seconds on the first call for a new schema. Subsequent calls with the same schema are fast because the provider caches the compiled schema.

2. **JSON mode reduces output tokens compared to free-form**: When the model outputs prose + JSON, it produces both explanation and data. JSON-only mode eliminates the prose wrapper, saving ~20-30% output tokens.

3. **Strict mode can be slower than non-strict**: Reports of strict mode adding 15-25 seconds vs 2-3 seconds non-strict for some models. This is because strict mode requires constrained decoding (finite state machine guiding token selection).

4. **Compressed finite state machine decoding** (used by SGLang, Outlines) can actually make structured output FASTER than unconstrained by skipping invalid token probabilities: up to 2x latency reduction and 2.5x throughput improvement.

### Recommendation for the Pipeline

The pipeline already requests JSON output via the prompt template. Current approach:

```yaml
# Current: JSON requested in prompt text
script_dialogue: |
  Return your response as a JSON object with this exact structure:
  ```json
  { "dialogue": [...], "metadata": {...} }
  ```
```

**Recommended change**: Use `response_format: {type: "json_object"}` (not `json_schema`) for script generation.

Rationale:
- The dialogue schema is flexible (variable number of entries), making strict schema enforcement complex
- `json_object` mode eliminates the need for JSON extraction/parsing from mixed text
- Avoids the first-call compilation penalty of strict mode
- Still include the schema example in the prompt for structural guidance

```python
# In generate_text_with_failover call for script stage:
response = await client.chat.completions.create(
    model=model,
    messages=messages,
    response_format={"type": "json_object"},  # Guarantee valid JSON
    max_tokens=max_tokens,
)
```

- Latency impact: Neutral to positive (eliminates retry-on-parse-failure)
- Cost impact: ~20-30% fewer output tokens (no prose wrapper)
- Reliability: Significantly improved (guaranteed valid JSON)

---

## 4. Chain-of-Thought Control

### When CoT Helps vs. Wastes Tokens

| Task Type | CoT Value | Reasoning |
|-----------|----------|-----------|
| Research planning | **High** | Multi-step: topic -> subtopics -> search queries -> prioritization |
| Research synthesis | **Medium** | Deduplication logic benefits, but can be done programmatically |
| Outline generation | **Medium** | Structural decisions benefit from reasoning |
| Script dialogue | **Low** | Creative generation, not analytical. CoT adds unwanted meta-commentary |
| Complexity analysis | **High** | Classification with justification |

### The Hidden Token Cost of CoT

Models like GPT-5, DeepSeek R1, and Kimi K2.5 generate reasoning traces by default. These "thinking tokens" are billed:

- DeepSeek R1: generates 4.8x more output tokens than non-reasoning models for the same task
- Kimi K2.5: 57,569 output tokens for 38 tasks vs Sonnet's ~12,000
- Cost impact: 2-5x higher output token costs when reasoning is enabled

### Reasoning Effort Parameter

OpenAI and other providers expose a `reasoning_effort` parameter:

| Setting | Use Case | Token Impact |
|---------|----------|-------------|
| `low` | Classification, extraction, formatting | ~0.5x reasoning tokens |
| `medium` | Analysis, synthesis, planning | ~1x reasoning tokens (default) |
| `high` | Complex multi-step reasoning, math | ~2x reasoning tokens |

### Recommendations for Pipeline Stages

```python
# Research planner: reasoning helps query design
response = await generate(
    model="gpt-5-mini",
    reasoning_effort="medium",  # Benefits from planning logic
    messages=planner_messages,
)

# Script dialogue: suppress reasoning, it's creative output
response = await generate(
    model="gpt-5-mini",
    reasoning_effort="low",  # Creative writing, not analysis
    messages=script_messages,
)

# Research synthesis: medium reasoning for dedup logic
response = await generate(
    model="gpt-5-mini",
    reasoning_effort="medium",
    messages=synthesis_messages,
)
```

- Token savings: **30-60%** on output tokens for stages where reasoning is suppressed
- Quality impact: Minimal for creative/formatting tasks, significant for analytical tasks
- Code pattern: Set `reasoning_effort` per-stage in the worker configuration

### GPT-5 Specific Warning

GPT-5 is a router-based system where saying "think hard about this" in the prompt literally triggers the reasoning model. Avoid phrases like "think step by step" or "reason carefully" in script generation prompts -- they increase latency and cost with no quality benefit for creative writing.

---

## 5. Output Length Control

### Does max_tokens Affect Speed?

**Yes, but indirectly.** The model generates tokens sequentially until it either (a) produces a stop token, (b) hits max_tokens, or (c) uses a stop sequence. If the model generates 1000 tokens naturally, setting max_tokens=2000 vs max_tokens=8000 produces identical latency -- the model stops at the natural endpoint regardless.

However:

1. **max_tokens acts as a safety net**: If the model "rambles" and produces 7000 tokens instead of the expected 1000, max_tokens=2000 cuts it off early, saving ~5 seconds and 5000 output tokens of cost.

2. **Predicted outputs**: OpenAI supports a `prediction` parameter for when you know most of the output. This enables speculative decoding and can reduce latency significantly for structured outputs.

### Rule of Thumb

```
actual_latency = TTFT + (output_tokens * per_token_latency)

# Per-token latency by model (2026 benchmarks):
# GPT-5.2:        0.020s/token
# Claude 4.5:     0.030s/token
# Mistral Large:  0.025s/token
# Grok 4.1 Fast:  0.010s/token
```

A 1000-token output on GPT-5.2: TTFT(0.6s) + 1000 * 0.020 = **20.6 seconds**

### Recommendations per Stage

| Stage | Expected Output | Recommended max_tokens | Rationale |
|-------|----------------|----------------------|-----------|
| Research planning | 500-1000 | 1500 | Short JSON with search queries |
| Research synthesis | 1000-2000 | 3000 | Condensed synthesis JSON |
| Outline generation | 500-1500 | 2000 | Structured outline JSON |
| Script generation (10min) | 3000-5000 | 6000 | Dialogue JSON, ~150 words/min |
| Script generation (30min) | 8000-15000 | 16000 | Longer episodes need more room |

```python
# Set max_tokens based on target duration
def get_script_max_tokens(duration_min: float) -> int:
    """Conservative upper bound: ~500 tokens per minute of podcast."""
    base = int(duration_min * 500)
    buffer = int(base * 0.2)  # 20% buffer for metadata
    return min(base + buffer, 16384)
```

- Cost savings: Prevents runaway generation (estimated 5-10% cost reduction from avoiding long-tail responses)
- Latency savings: Caps worst-case latency proportional to max_tokens

---

## 6. Prompt Decomposition

### One Big Prompt vs. Multiple Smaller Prompts

The pipeline ALREADY uses decomposition effectively:

```
Planner (outline) --> Research (queries) --> Tools (execution) --> Assimilator (synthesis) --> Script (dialogue)
```

The question is: should any of these stages be further decomposed or consolidated?

### When Decomposition Wins (Parallel Execution)

Decomposition is faster when:
1. Sub-tasks are **independent** and can run in parallel
2. Each sub-task is **significantly shorter** than the combined prompt
3. The overhead of multiple API calls (TTFT * N) is less than the single-call generation time

```
Single 15000-token output: TTFT + 15000 * 0.02 = 0.6 + 300 = 300.6s

Three parallel 5000-token outputs: max(TTFT + 5000 * 0.02) = 0.6 + 100 = 100.6s
                                   3x API overhead = ~1.8s TTFT total (but parallel)
                                   Wall time: ~100.6s (3x faster!)
```

### When One Prompt Wins

One prompt is faster when:
1. Sub-tasks are **sequential** (each depends on the previous)
2. The total output is **short** (under ~2000 tokens)
3. Maintaining context across sub-tasks is critical

### Current Pipeline Assessment

| Current Design | Assessment | Recommendation |
|---------------|------------|----------------|
| Research planner = separate worker | Correct -- planner output feeds research | Keep separate |
| Research assimilator = separate worker | Correct -- synthesis of research is independent of outline | Keep separate |
| Script = single prompt for full episode | **Candidate for decomposition** for long episodes (20+ min) | See Section 15 |
| Planner + script in one prompt | Tested and rejected -- too complex, quality drops | Keep separate |

### The Consolidation Opportunity

The planner and research synthesis could potentially be consolidated for SHORT podcasts (under 5 minutes):

```python
# For short podcasts: combined prompt saves one API round-trip
if duration_min <= 5:
    # Single call: outline + research query planning
    combined_prompt = load_prompt("stages/planner/combined_short")
    # Saves: 1 API call (~0.6s TTFT + overhead)
else:
    # Standard pipeline: separate planner and research stages
    outline = await plan_outline(...)
    queries = await plan_research(outline, ...)
```

- Latency savings: 0.5-1.0 seconds per eliminated API call
- Quality impact: Acceptable for short content, risky for longer content where quality of each stage matters
- Recommendation: Keep current decomposition. The overhead per call (0.6s TTFT) is small relative to the total pipeline time (30-120 seconds).

---

## 7. Few-Shot Learning Efficiency

### How Many Examples Are Optimal?

Research consensus for 2025-2026:

| Shots | When to Use | Token Cost | Quality |
|-------|-------------|------------|---------|
| 0-shot | Strong models (GPT-5, Claude 4), simple tasks | Lowest | Good for well-defined tasks |
| 1-shot | Risky -- model overfits to the single example | Low | Inconsistent |
| 2-shot | Minimum for pattern learning | Medium | Good baseline |
| 3-shot | Standard recommendation | Medium-High | Strong |
| 5+ shots | Diminishing returns for most tasks | High | Marginal improvement |

Key finding: "2 is a good rule-of-thumb minimum for few-shot examples. 1 example makes models overfit to the specifics." (Prompting Weekly, 2026)

### System Prompt vs. User Prompt Placement

**Critical guidance**: "Do not write your examples in your system prompt."

Reasoning:
1. Examples in the system prompt get treated as "persona rules" rather than "task demonstrations"
2. Examples in the user message flow naturally as "here's what I want, here are examples"
3. For caching purposes, examples should be in a SEPARATE cached message, not mixed with dynamic content

**Optimal structure for prompt caching:**

```python
messages = [
    # Message 1: System prompt (cached -- stable across all requests)
    {"role": "system", "content": system_prompt},

    # Message 2: Static examples (cached -- stable per content_type)
    {"role": "user", "content": few_shot_examples},
    {"role": "assistant", "content": few_shot_response},

    # Message 3: Dynamic request (not cached -- changes per request)
    {"role": "user", "content": actual_request},
]
```

This way, messages 1-2 form a cacheable prefix, and only message 3 changes per request.

### Pipeline-Specific Recommendations

**Script generation**: The current prompt has `include_examples: False` and a comment noting "Examples section removed to prevent LLM from pattern-matching English text." This is correct for multi-language support. Instead of full examples, use **format examples** (JSON structure only, no natural language content):

```yaml
examples:
  - note: "Structure-only example (no language content)"
    output: |
      {"dialogue": [
        {"speaker": "Host1", "text": "[CONTENT]", "emotion": "warm", "intensity": 0.7, "rate": 1.0},
        {"speaker": "Host2", "text": "[CONTENT]", "emotion": "curious", "intensity": 0.6, "rate": 1.05}
      ]}
```

- Token cost: ~50 tokens for a structural example (vs ~500 for a full content example)
- Benefit: Teaches JSON format without biasing language content
- Cache benefit: One structural example is identical across all languages and can be cached

**Research synthesis**: 0-shot is sufficient. The task is well-defined and the output format is specified in the prompt.

**Outline generation**: 0-shot is sufficient for modern models. The content_type classification table in the prompt acts as implicit few-shot guidance.

---

## 8. Template Optimization

### Current Architecture Analysis

The `PromptLoader` assembles prompts from:

```
dialogue_generation.yaml (base template)
  + shared/speaker_rules.yaml
  + shared/output_formats.yaml#script_dialogue
  + shared/quality_guidelines.yaml#content_quality
  + shared/density_guidelines.yaml (4 sections)
  + styles/{style}.yaml
  + content_types/{type}/dialogue_rules.yaml
  + content_domains/{domain}/profile.yaml
  + config/emotional_arcs.yaml
  + config/expression_examples.yaml
  + runtime variables (topic, outline, research, language)
```

Estimated total assembled prompt: **4000-8000 tokens** depending on content type and domain.

### Token Distribution (Estimated)

| Component | Tokens | Type | Cacheable? |
|-----------|--------|------|-----------|
| System prompt | ~80 | Static | Yes (always) |
| Speaker rules | ~200 | Static | Yes (always) |
| Output format (JSON schema) | ~300 | Static | Yes (always) |
| Quality guidelines | ~150 | Static | Yes (always) |
| Density guidelines (4 sections) | ~350 | Static | Yes (always) |
| Style guidance | ~200 | Semi-static | Yes (per style) |
| Content type rules | ~300 | Semi-static | Yes (per content type) |
| Content domain rules | ~200 | Semi-static | Yes (per domain) |
| Emotional arc | ~200 | Semi-static | Yes (per arc type) |
| Expression examples | ~300 | Semi-static | Yes (per language) |
| **Subtotal: Static/Semi-static** | **~2280** | | **Cacheable** |
| Topic | ~50 | Dynamic | No |
| Custom instructions | ~100 | Dynamic | No |
| Outline JSON | ~500 | Dynamic | No |
| Research summary | ~2000-5000 | Dynamic | No |
| Language instructions | ~100 | Dynamic | No |
| **Subtotal: Dynamic** | **~2750-5750** | | **Not cacheable** |

### Optimization Strategy: Tiered Caching

```
Tier 1 (Global): System + Speaker + Output Format + Quality + Density
  -> Identical for ALL podcast generations
  -> Cache key: "script-global-v2.0"
  -> ~1080 tokens cached

Tier 2 (Content Profile): Style + Content Type + Domain + Emotional Arc + Expressions
  -> Identical for same content configuration
  -> Cache key: "script-{style}-{content_type}-{domain}-{arc}"
  -> ~1200 tokens cached

Tier 3 (Request): Topic + Instructions + Outline + Research + Language
  -> Unique per request
  -> ~2750-5750 tokens (not cached)
```

### Concrete Optimization: Compact the Static Layer

The static layer (Tier 1) can be compressed without quality loss:

```yaml
# BEFORE: Verbose markdown with tables, headers, emoji (1080 tokens)
# AFTER: Compact rules (650 tokens, 40% reduction)

speaker_rules: |
  Speakers: Host1 (lead, introduces topics), Host2 (adds depth, asks questions).
  Names always "Host1"/"Host2". Natural conversation, not interview.

density_rules: |
  5-SECOND RULE: Every 5s delivers fact/insight/example/question/story-beat.
  BANNED: "That's fascinating" -> follow-up question. "A lot of people" -> specific number.
  "Many studies show" -> cite study+year. "Recently" -> specific date.
  HOST2: Every reply adds info/deepens/connects/gives perspective. Never pure reaction.
  DEPTH: Expert insights, not Wikipedia. Research with sources. Counter-intuitive facts.
```

- Token savings: ~430 tokens per request on static content
- Annual savings at 1000 jobs/day: ~430K fewer input tokens/day
- Quality: Must A/B test to confirm no regression

### The Variable Content Minimization

The largest token consumer is research_summary (2000-5000 tokens). This is the primary target for compression (see Section 9).

---

## 9. Research Synthesis Prompt

### Current Flow

```
Raw Research (JSON, 4000-8000 tokens)
  -> Assimilator LLM call (research_synthesis.yaml)
  -> Synthesized output (1000-2000 tokens)
  -> Passed to Script Worker
```

### Pre-Compression Before LLM

Apply programmatic extraction BEFORE sending to the assimilator:

```python
import re
from collections import defaultdict

def pre_compress_research(results: list[dict], max_entries: int = 20) -> str:
    """
    Extract high-density facts from raw research before LLM synthesis.

    Techniques:
    1. Deduplicate by URL (same source = merge)
    2. Extract sentences with numbers/stats (highest info density)
    3. Remove boilerplate (navigation, disclaimers, ads)
    4. Truncate to max_entries most relevant items
    """
    seen_urls = set()
    entries = []

    for result in results:
        url = result.get("url", "")
        if url in seen_urls:
            continue
        seen_urls.add(url)

        content = result.get("content", "")
        title = result.get("title", "")

        # Extract high-info sentences
        sentences = re.split(r'[.!?]\s+', content)
        high_value = []
        for s in sentences:
            s = s.strip()
            if len(s) < 20 or len(s) > 300:
                continue
            # Score: has number/stat/quote/date
            score = sum([
                bool(re.search(r'\d+%', s)),           # percentage
                bool(re.search(r'\$[\d,.]+', s)),       # dollar amount
                bool(re.search(r'\b20[12]\d\b', s)),    # year
                bool(re.search(r'\d+\s*(million|billion|thousand)', s, re.I)),
                bool('"' in s or "'" in s),             # quote
            ])
            if score > 0:
                high_value.append(s)

        if high_value:
            entries.append({
                "src": title[:60],
                "url": url,
                "facts": high_value[:5],  # top 5 facts per source
            })

    # Sort by fact count (most informative first), take top entries
    entries.sort(key=lambda x: len(x["facts"]), reverse=True)
    return json.dumps(entries[:max_entries], separators=(',', ':'))
```

### Token Impact

| Stage | Before | After | Savings |
|-------|--------|-------|---------|
| Raw research input to assimilator | 4000-8000 | 1500-3000 | 50-65% |
| Assimilator output | 1000-2000 | 800-1500 | 20-30% |
| Research passed to script worker | 1000-2000 | 800-1500 | 20-30% |

### Alternative: Skip the Assimilator for Short Podcasts

For podcasts under 5 minutes with low complexity, the assimilator stage may be unnecessary:

```python
if duration_min <= 5 and complexity_score < 0.5:
    # Skip assimilator, pass pre-compressed research directly to script
    research_for_script = pre_compress_research(raw_results)
    # Saves: 1 full LLM call (~5-15 seconds + ~2000 input tokens + ~1500 output tokens)
```

- Latency savings: 5-15 seconds (entire LLM call eliminated)
- Cost savings: ~3500 tokens (input + output of assimilator)
- Quality impact: Acceptable for short, simple topics. The script LLM can handle raw facts.

---

## 10. Script Generation Prompt

### Current Token Analysis

The `dialogue_generation.yaml` fully assembled prompt is the largest in the pipeline. Breaking down the user template:

```
Language requirement block:              ~50 tokens
Topic + context:                         ~150 tokens
Outline:                                 ~500 tokens
Research findings:                       ~2000 tokens
Requirements block:                      ~30 tokens
No-greeting block (3x repeated!):        ~250 tokens
Speaker rules:                           ~200 tokens
Content-adaptive approach:               ~100 tokens
Style guidance:                          ~200 tokens
Voice direction + emotion palette:       ~400 tokens
Emotional arc:                           ~200 tokens
Expression examples:                     ~300 tokens
Quality requirements (density, etc):     ~350 tokens
Content type rules:                      ~300 tokens
Domain rules:                            ~200 tokens
Dialogue progression:                    ~100 tokens
Content quality:                         ~150 tokens
Output format:                           ~300 tokens
Final checklist:                         ~100 tokens
------------------------------------------
TOTAL:                                   ~5380 tokens (input)
```

### Emotion Metadata: In-Prompt vs. Post-Processing

**Current approach**: Emotion metadata (emotion, intensity, rate) is requested in the prompt and generated inline with dialogue.

**Alternative**: Generate dialogue text only, then post-process emotions:

```python
# Option A: Current (emotions in prompt)
# Pro: Model has full context to choose appropriate emotions
# Con: +400 tokens of emotion guidance in prompt, +30% output tokens for metadata

# Option B: Post-process emotions
# Pro: Simpler prompt, fewer output tokens
# Con: Requires second pass or rule-based emotion assignment
# Quality risk: Rule-based emotions lack context awareness

# Option C: Hybrid -- minimal emotion prompt + post-process refinement
# Include ONLY core emotion palette (50 tokens instead of 400)
# Accept model's basic emotion choices
# Post-process only to validate/adjust intensity and rate
```

**Recommendation**: Keep emotions in-prompt (Option A) but compress the emotion guidance:

```yaml
# BEFORE: 400 tokens of emotion palettes, genre-specific lists, precedence rules
# AFTER: 120 tokens of compact guidance

voice_direction_compact: |
  Each line needs: emotion, intensity (0.3-1.0), rate (0.8-1.3).
  Core: warm|excited|curious|thoughtful|amazed|concerned|playful
  Genre-specific: suspenseful|dread|witty|sarcastic|tender|authoritative|inspiring
  Match emotions to content genre. Vary across dialogue for natural flow.
  Arc: opening(warm,0.6) -> peak(genre-primary,0.8) -> close(warm,0.6)
```

- Token savings: ~280 tokens per request
- Quality impact: Must A/B test. The compressed version may be sufficient for modern models that understand emotion metadata from training data.

### Output Token Optimization

The dialogue output contains verbose metadata. Consider compact output format:

```json
// BEFORE: Verbose (~30 tokens per dialogue line)
{"speaker": "Host1", "text": "...", "emotion": "warm", "intensity": 0.7, "rate": 1.0}

// AFTER: Compact (~22 tokens per dialogue line)
{"s": "H1", "t": "...", "e": "warm", "i": 0.7, "r": 1.0}
```

Post-process to expand:
```python
def expand_dialogue(compact: dict) -> dict:
    return {
        "speaker": "Host1" if compact["s"] == "H1" else "Host2",
        "text": compact["t"],
        "emotion": compact["e"],
        "intensity": compact["i"],
        "rate": compact["r"],
    }
```

- Token savings: ~25% on output tokens (the most expensive tokens)
- For a 10-minute podcast (~100 dialogue lines): saves ~800 output tokens
- At $14/M output tokens: saves ~$0.01 per podcast (adds up at scale)

---

## 11. Multi-turn vs. Single-turn

### Key Research Finding

"Users are better off consolidating all requirements into a single prompt rather than clarifying over multiple turns. If a conversation goes off-track, starting a new session with a consolidated summary leads to better outcomes." (LinkedIn analysis, 2026)

### Multi-turn Token Growth Problem

In multi-turn conversations, each API call includes ALL prior turns:

```
Turn 1: system(500) + user(200)                    = 700 input tokens
Turn 2: system(500) + user(200) + asst(400) + user(200) = 1300 input tokens
Turn 3: system(500) + user(200) + asst(400) + user(200) + asst(400) + user(200) = 1900 tokens
```

Token cost grows linearly per turn. By turn 10, you are paying for ~6000+ input tokens per call.

### Pipeline Implications

The pipeline's current design is ALREADY optimal: each stage is a **single-turn call** that passes structured data (JSON) to the next stage. There is no multi-turn conversation.

**Anti-pattern to avoid**: Do NOT convert the pipeline into a multi-turn conversation where the model "remembers" previous stages. This would:
1. Grow input tokens at each stage
2. Risk context window overflow for long episodes
3. Break cacheability (each turn has different prefix)

### When Multi-turn Could Help

The only scenario where multi-turn is beneficial: **iterative refinement** of a script. If quality checks fail, a follow-up turn asking "fix these issues" is cheaper than regenerating from scratch:

```python
# First attempt
script = await generate_script(prompt)

# If quality check fails:
if not passes_quality_check(script):
    # Multi-turn refinement (cheaper than full regeneration)
    fix_prompt = f"Fix these issues in the script: {issues}. Keep everything else."
    # Input: system + original prompt + script + fix instructions
    # This reuses the cached prefix from the first call
    fixed_script = await generate_refinement(
        messages=[
            original_system,      # cached
            original_user,        # cached (if same prefix)
            {"role": "assistant", "content": json.dumps(script)},
            {"role": "user", "content": fix_prompt},  # new
        ]
    )
```

- Benefit: Reuses prefix cache, avoids regenerating the entire script
- Cost: Previous assistant output counts as input (~5000 tokens)
- Recommendation: Use sparingly. Usually regeneration from scratch is simpler and comparable cost.

---

## 12. Temperature and Quality

### Temperature Does NOT Directly Affect Latency

Temperature controls the probability distribution over next tokens. It does not change:
- Number of forward passes (always one per token)
- Computation per token
- Model loading or memory usage

**However**, temperature indirectly affects cost and latency through:
1. **Retry rate**: High temperature = more randomness = more format violations = more retries
2. **Output variance**: High temperature = more diverse (and potentially longer) outputs
3. **Structured output compliance**: Temperature > 0.7 increases JSON parse failures

### Optimal Temperature by Task

| Task | Recommended Temperature | Rationale |
|------|------------------------|-----------|
| Research planning | 0.0-0.2 | Deterministic query generation. Same topic should produce same queries. |
| Complexity analysis | 0.0 | Classification task. Deterministic is correct. |
| Research synthesis | 0.2-0.3 | Mostly factual extraction. Low variance needed. |
| Outline generation | 0.3-0.5 | Some creativity for structure, but consistent quality. |
| Script dialogue | 0.6-0.8 | Creative writing benefits from diversity. But not too high or format breaks. |
| Script dialogue (comedy) | 0.7-0.9 | Humor requires more randomness for unexpected phrasing. |
| Script dialogue (news) | 0.3-0.5 | Factual accuracy matters more than creativity. |

### Research Findings on Temperature Impact

A 2026 study on temperature effects found:
- **Large models**: Temperature has modest effect on performance. Optimizing temperature is "generally less critical for larger models."
- **Small models**: Temperature can impact creativity by up to 186% and machine translation by up to 192%
- **Structured output**: Setting temperature to 0.0 "ensures consistently valid structured data"
- **High temperature + JSON**: "Developers sometimes use high temperatures for tasks requiring structured output, hoping to improve creativity. This is a mistake." High temperature increases format violation rates.

### Recommendation

```python
TEMPERATURE_MAP = {
    "research_planning": 0.1,
    "complexity_analysis": 0.0,
    "research_synthesis": 0.2,
    "outline_generation": 0.4,
    "script_dialogue": 0.7,
    "script_narration": 0.6,
    "script_monologue": 0.5,
}

# Content-type adjustment
if content_type == "news":
    temp = max(temp - 0.2, 0.0)  # More deterministic for factual content
elif content_type == "comedy":
    temp = min(temp + 0.1, 0.9)  # More creative for humor
```

- Direct latency impact: None
- Indirect cost savings: 5-15% fewer retries from better format compliance
- Quality impact: Significant -- matching temperature to task type improves output consistency

---

## 13. Stop Sequences

### How Stop Sequences Work

Stop sequences tell the model to halt generation when a specific string is produced. The model stops IMMEDIATELY upon generating the sequence, saving all subsequent tokens.

### Application for JSON Output

For JSON output, stop sequences can terminate generation after the closing structure:

```python
# For dialogue generation (JSON output)
response = await client.chat.completions.create(
    model=model,
    messages=messages,
    stop=["\n```", "```\n"],  # Stop after JSON code block closes
)
```

**However**, with `response_format: {type: "json_object"}`, the model inherently stops after producing valid JSON. Stop sequences are redundant in this mode.

### Where Stop Sequences ARE Useful

1. **Free-form with embedded JSON**: If the prompt asks for JSON but the model might add commentary after:
   ```python
   stop=["}\n\nNote:", "}\n\nI ", "}\n\nPlease"]  # Prevent post-JSON commentary
   ```

2. **Research planning**: Stop after the tasks array closes to prevent additional text:
   ```python
   stop=['"enabled": true\n    }\n  ]\n}']  # Exact end of research tasks JSON
   ```

3. **Streaming with early detection**: For streaming responses, detect JSON completeness mid-stream and cancel:
   ```python
   async for chunk in stream:
       buffer += chunk
       if is_complete_json(buffer):
           await stream.cancel()  # Stop immediately
           break
   ```

### Token Savings Estimate

Models typically add 50-200 "padding" tokens after the main content (explanations, notes, caveats). Stop sequences eliminate these:

- Per-call savings: **50-200 output tokens**
- At $14/M output tokens: ~$0.001-$0.003 per call
- At 1000 calls/day: **$1-3/day** savings

### Recommendation

Use `response_format: {type: "json_object"}` (which handles stops implicitly) rather than manual stop sequences. The implicit stop is more reliable and handles edge cases (nested braces, strings containing the stop sequence).

---

## 14. Prompt Caching Architecture

### Provider-Level Caching

**OpenAI Automatic Caching:**
- Triggers automatically for prompts >= 1,024 tokens
- Cache hits in 128-token increments
- TTL: 5-10 minutes of inactivity
- Discount: 50% on cached input tokens
- No code changes required (but optimization helps)
- Optional `prompt_cache_key` for extended retention

**Anthropic Explicit Caching:**
- Requires `cache_control` directive in messages
- Exact prefix byte-level matching
- TTL: 5 minutes default (configurable)
- Discount: 90% on cached read tokens
- Cache creation has a write premium (~25% extra on first write)
- Up to 4 cache breakpoints per request

### Optimal Architecture for the Podcast Pipeline

```
                    CACHE TIER 1: Global Prompt Prefix
                    (System + Speaker Rules + Quality + Density + Format)
                    ~1080 tokens, shared across ALL requests
                    Cache key: "prompt-v2.0-global"
                              |
                    CACHE TIER 2: Content Configuration
                    (Style + Content Type + Domain + Emotional Arc)
                    ~1200 tokens, shared across same-config requests
                    Cache key: "prompt-v2.0-{style}-{type}-{domain}"
                              |
                    DYNAMIC LAYER: Per-Request Content
                    (Topic + Outline + Research + Language)
                    ~2750-5750 tokens, unique per request
```

### Implementation Pattern

```python
class CacheAwarePromptBuilder:
    """Build prompts optimized for provider prefix caching."""

    def __init__(self):
        self.loader = PromptLoader()

    def build_messages(
        self,
        prompt_path: str,
        variables: dict,
        style: str,
        content_type: str,
        content_domain: str,
    ) -> list[dict]:
        """
        Build messages with cache-optimal ordering.

        Returns messages structured for maximum cache hits:
        1. System message: global rules (cached across all requests)
        2. User message 1: content config (cached per config)
        3. User message 2: dynamic content (not cached)
        """
        parts = self.loader.load_raw(prompt_path)

        # LAYER 1: System prompt (always cached)
        system = parts.get("system", "").strip()

        # LAYER 2: Static includes + content config
        # These are resolved once and reused across requests with same config
        static_rules = self._assemble_static_rules(parts, style, content_type, content_domain)

        # LAYER 3: Dynamic per-request content
        dynamic_content = self._assemble_dynamic(variables)

        messages = [
            {
                "role": "system",
                "content": system,
                # Anthropic: cache_control here
            },
            {
                "role": "user",
                "content": static_rules,
                # This message is identical for same (style, type, domain)
                # Anthropic: cache_control here
            },
            {
                "role": "user",
                "content": dynamic_content,
            },
        ]

        return messages

    def _assemble_static_rules(self, parts, style, content_type, domain) -> str:
        """Assemble all static/semi-static rules into one cacheable block."""
        sections = []

        # Resolve all includes
        for include_path in parts.get("includes", []):
            content = self.loader._resolve_include(include_path)
            if content:
                sections.append(content)

        # Style guidance
        if style and "style_overrides" in parts:
            style_content = self.loader._load_style(parts["style_overrides"].get(style, ""))
            if style_content:
                sections.append(f"## STYLE GUIDANCE\n{style_content}")

        # Content type rules
        if content_type:
            type_content = self.loader._load_content_type_rules(content_type, ...)
            if type_content:
                sections.append(f"## CONTENT TYPE RULES\n{type_content}")

        return "\n\n".join(sections)
```

### Cross-Job Caching Analysis

For the KitesForU pipeline, cross-job cache hit potential:

| Content Configuration | Estimated Daily Volume | Cache Value |
|----------------------|----------------------|-------------|
| educational + casual + en | ~30% of jobs | High (same prefix) |
| news + professional + en | ~15% of jobs | High |
| story + storytelling + en | ~10% of jobs | Medium |
| course + professional + en | ~10% of jobs | Medium |
| all configs combined | 100% | Tier 1 always hits |

Expected cache hit rate: **60-80%** on Tier 1, **30-50%** on Tier 2

### Cost Projection

Assumptions: 500 podcast generations/day, average 5000 input tokens per script call

| Scenario | Daily Input Tokens | Daily Cost (GPT-5.2 @ $1.75/M) |
|----------|-------------------|-------------------------------|
| No caching | 2,500,000 | $4.38 |
| Tier 1 only (50% cached) | 1,960,000 effective | $3.19 |
| Tier 1 + Tier 2 (70% cached) | 1,595,000 effective | $2.41 |

**Annual savings from caching**: ~$500-700 on script stage alone, more across all stages.

---

## 15. Parallel Generation

### The Opportunity: Parallel Script Segments

For longer episodes (15+ minutes), generate script segments in parallel:

```
                    Outline (with N segments)
                         |
            +------------+------------+
            |            |            |
     Segment 1      Segment 2     Segment 3
     (Intro)        (Body)        (Outro)
     3 min          9 min         3 min
            |            |            |
            +------------+------------+
                         |
                   Merge & Polish
```

### Wall-Time Savings

```
Serial:    TTFT + (5000 + 3000 + 2000) * 0.02 = 0.6 + 200 = 200.6s
Parallel:  max(TTFT + 5000*0.02, TTFT + 3000*0.02, TTFT + 2000*0.02)
         = max(100.6, 60.6, 40.6) = 100.6s
         + merge overhead (~2s)
         = 102.6s

Speedup: ~2x for 3-segment parallel generation
```

### The Merge Challenge

Parallel segments need coherent transitions. Two approaches:

**A. Overlap Context (Recommended)**
Each segment prompt includes the previous segment's outline AND the last 2-3 dialogue lines (or summary) from the previous segment to ensure continuity:

```python
async def generate_parallel_segments(outline, research, config):
    segments = outline["segments"]

    # First segment always generates first (it sets the tone)
    first_segment = await generate_segment(
        segment=segments[0],
        research=research,
        config=config,
        prior_context=None,
    )

    # Remaining segments can run in parallel, each with first segment's ending as context
    last_lines = first_segment["dialogue"][-3:]  # Last 3 lines for continuity

    tasks = []
    for i, segment in enumerate(segments[1:], 1):
        tasks.append(generate_segment(
            segment=segment,
            research=research,
            config=config,
            prior_context={
                "previous_segment_title": segments[i-1]["title"],
                "transition_from": json.dumps(last_lines),
            },
        ))

    remaining = await asyncio.gather(*tasks)

    # Merge: first_segment + remaining segments
    all_dialogue = first_segment["dialogue"]
    for seg in remaining:
        all_dialogue.extend(seg["dialogue"])

    return {"dialogue": all_dialogue, "metadata": {...}}
```

**B. Post-Merge Polish (Higher quality, higher cost)**
Generate all segments in parallel with only outline context, then run a lightweight "polish" pass to smooth transitions:

```python
# All segments in parallel (maximum speed)
segments = await asyncio.gather(*[
    generate_segment(seg, research, config) for seg in outline["segments"]
])

# Polish pass: fix transitions (lightweight, ~500 output tokens)
merged = merge_segments(segments)
polished = await polish_transitions(merged)  # Small LLM call
```

### Token Efficiency: Parallel vs. Serial

| Approach | Input Tokens | Output Tokens | Total Cost | Wall Time |
|----------|-------------|---------------|-----------|----------|
| Single prompt (serial) | 5000 | 5000 | 1x | 1x |
| 3 parallel segments | 3 * 3500 = 10500 | 3 * 1800 = 5400 | ~1.8x | ~0.5x |
| 3 parallel + polish | 10500 + 6000 = 16500 | 5400 + 500 = 5900 | ~2.3x | ~0.55x |

**Trade-off**: Parallel generation costs ~1.8-2.3x more tokens but is ~2x faster in wall time.

### When to Use Parallel Generation

```python
def should_parallelize(duration_min: float, segment_count: int) -> bool:
    """
    Parallelize when wall-time savings justify the token cost.

    Rules:
    - Duration >= 15 min AND >= 3 segments: parallel is worth it
    - Duration < 10 min: single prompt is fine
    - Duration 10-15 min: marginal, depends on latency SLA
    """
    if duration_min >= 15 and segment_count >= 3:
        return True
    return False
```

### Recommendation

1. **Keep single-prompt for episodes under 15 minutes** (majority of traffic)
2. **Enable parallel generation for 15+ minute episodes** as an opt-in optimization
3. **Use Approach A (overlap context)** -- it is cheaper than Approach B and produces acceptable continuity
4. **Monitor merge quality** -- if transitions feel jarring, add the polish pass

---

## Summary: Priority-Ranked Optimizations

| Priority | Technique | Est. Savings | Effort | Quality Risk |
|----------|-----------|-------------|--------|-------------|
| **P0** | Prompt caching architecture (Section 14) | 50-90% input cost, 75% TTFT | Medium | None |
| **P0** | JSON response_format mode (Section 3) | Eliminates parse retries, 20-30% output tokens | Low | None |
| **P1** | Research pre-compression (Section 9) | 50-65% on research tokens | Medium | Low |
| **P1** | Remove duplicated instructions (Section 1) | 5-15% input tokens | Low | None |
| **P1** | Per-stage reasoning_effort (Section 4) | 30-60% output tokens on non-analytical stages | Low | Low |
| **P1** | Temperature per task type (Section 12) | 5-15% fewer retries | Low | None |
| **P2** | Compact emotion guidance (Section 10) | ~280 tokens per request | Low | Must A/B test |
| **P2** | Minify JSON in prompts (Section 1) | 30-40% on JSON payloads | Trivial | None |
| **P2** | Compact output format (Section 10) | ~25% output tokens | Low | None |
| **P2** | Set max_tokens per stage (Section 5) | Prevents runaway, 5-10% cost | Low | None |
| **P3** | Parallel segment generation (Section 15) | 2x wall-time for 15+ min episodes | High | Medium |
| **P3** | Skip assimilator for short podcasts (Section 9) | 5-15s latency, ~3500 tokens | Medium | Medium |
| **P3** | Few-shot structural examples (Section 7) | Better format compliance | Low | Low |
| **DEFER** | Instruction IDs via fine-tuning (Section 1) | 90%+ on instructions | Very High | Requires training |
| **DEFER** | Prompt consolidation for short podcasts (Section 6) | 0.5-1.0s per eliminated call | Medium | Medium |

---

## Sources

### Provider Documentation
- OpenAI Prompt Caching Guide: https://developers.openai.com/api/docs/guides/prompt-caching/
- OpenAI Latency Optimization: https://developers.openai.com/api/docs/guides/latency-optimization/
- OpenAI Structured Outputs: https://developers.openai.com/api/docs/guides/structured-outputs/
- OpenAI Compaction Guide: https://developers.openai.com/api/docs/guides/compaction/
- OpenAI Prompt Caching Cookbook: https://developers.openai.com/cookbook/examples/prompt_caching_201/
- Anthropic Context Engineering: https://anthropic.com/engineering/effective-context-engineering-for-ai-agents

### Research and Benchmarks
- LLM Latency Benchmarks 2026: https://aimultiple.com/llm-latency-benchmark
- CoT Performance Analysis: https://arxiv.org/html/2506.14641v3
- Temperature Impact Study: https://arxiv.org/html/2506.07295v1
- Compressed FSM for JSON Decoding: https://lmsys.org/blog/2024-02-05-compressed-fsm/
- 38-Task 15-Model Benchmark: https://ianlpaterson.com/blog/llm-benchmark-2026-38-actual-tasks-15-models-for-2-29/

### Production Guides
- incident.io Prompt Optimization: https://incident.io/building-with-ai/optimizing-llm-prompts
- Redis Token Optimization: https://redis.io/blog/llm-token-optimization-speed-up-apps/
- Context Compression Techniques: https://www.buildmvpfast.com/blog/context-compression-techniques-fewer-tokens-llm-optimization-2026
- Prompt Caching Infrastructure: https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025
- AWS LLM Caching: https://aws.amazon.com/blogs/database/optimize-llm-response-costs-and-latency-with-effective-caching/
- Prompt Engineering Best Practices 2026: https://thomas-wiegold.com/blog/prompt-engineering-best-practices-2026/
- Few-Shot Prompting Guide: https://mem0.ai/blog/few-shot-prompting-guide
- Prompt Caching Zero to Production: https://atalupadhyay.wordpress.com/2026/02/10/prompt-caching-from-zero-to-production-ready-llm-optimization/

### Multi-turn and Decomposition
- LLMs Struggle in Multi-Turn: https://www.linkedin.com/posts/omarsar_llms-get-lost-in-multi-turn-conversation-activity-7328560484532588544-D_Gh
- Token Usage Cost Projection: https://iternal.ai/token-usage-guide
- Understanding LLM Cost Per Token: https://www.silicondata.com/blog/llm-cost-per-token
