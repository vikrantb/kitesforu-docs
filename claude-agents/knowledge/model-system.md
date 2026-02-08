# KitesForU Model System
<!-- last_verified: 2026-02-07 -->

## Model Catalog
- **Source of truth**: `kitesforu-workers/config/model_catalog.csv`
- **Fallback**: Google Sheets (configurable via `GoogleSheetsClient` in `routing/catalog/sheets_client.py`)
- **Parser**: `ModelCatalogParser` (in `routing/catalog/parser.py`, validates on load)
- **Cache**: `ModelCatalogCache` (in `routing/catalog/cache.py`, tries Sheets first, falls back to CSV)
- **Reload**: On app restart only (no hot-swap)

## Available Models (20 total, 12 enabled)

### LLM Models (Enabled)
| Model ID | Provider | Input $/MTok | Output $/MTok | Tier Default | Agentic | Notes |
|----------|----------|-------------|--------------|-------------|---------|-------|
| gpt-4.1-mini | OPENAI | $0.70 | $2.80 | trial, enthusiast, payg | no | Budget LLM, 1M context |
| gpt-5-mini | OPENAI | $0.25 | $3.60 | trial, enthusiast, payg | no | Cheapest GPT-5, 400K context |
| gpt-5 | OPENAI | $1.25 | $10.00 | creator | yes (1.5x) | Mid-tier agentic, 400K context |
| claude-sonnet-4-5-20250929 | ANTHROPIC | $3.00 | $15.00 | pro_creator, ultimate_creator | yes (1.2x) | Best for agents, 200K context |
| gemini-2.0-flash-exp | GOOGLE | $0.10 | free? | trial, enthusiast, payg | no | Experimental, 1M context |

### LLM Models (Disabled, available for manual selection)
| Model ID | Provider | Input $/MTok | Notes |
|----------|----------|-------------|-------|
| gpt-5.2 | OPENAI | $1.75 | Current flagship, agentic (1.3x) |
| gpt-5.1 | OPENAI | $2.50 | Previous flagship, agentic (1.4x) |
| gpt-4.1 | OPENAI | $3.50 | Fine-tunable, 1M context |
| gpt-4.1-nano | OPENAI | $0.20 | Cheapest overall |
| claude-opus-4-5-20251101 | ANTHROPIC | $5.00 | Most intelligent, agentic (1.1x) |
| claude-haiku-4-5-20251001 | ANTHROPIC | $1.00 | Fast/cheap |
| gemini-2.0-flash | GOOGLE | $0.075 | Budget option |
| gemini-1.5-pro | GOOGLE | $1.25 | Long context (2M), agentic (1.3x) |

### TTS Models (All Enabled)
| Model ID | Provider | Cost/1M chars | Languages | Tier Default |
|----------|----------|--------------|-----------|-------------|
| tts-1 | OPENAI | $15 | EN only | trial, enthusiast |
| tts-1-hd | OPENAI | $30 | EN only | (manual only) |
| eleven_flash_v2_5 | ELEVENLABS | $150 | 9 languages | pro_creator |
| eleven_turbo_v2_5 | ELEVENLABS | $150 | 9 languages | (manual only) |
| eleven_multilingual_v2 | ELEVENLABS | $300 | 29 languages | ultimate_creator |
| gcp-neural2 | GOOGLE | $16 | 11 languages | (fallback) |

### Transcription (Enabled)
| Model ID | Provider | Cost | Notes |
|----------|----------|------|-------|
| whisper-1 | OPENAI | $0.006/min | Speech-to-text, tier_default=all |

## Model Selection Algorithm (7-Step Pipeline)

### Location: `kitesforu-workers/src/workers/routing/router.py` (`ModelRouter.select_model()`)

```
Step 1: FILTER BY CAPABILITIES (_filter_by_capabilities)
  - Task type match (LLM, TTS, TRANSCRIPTION)
  - Language support (BCP 47 normalized to short codes via routing/languages.py)
  - Streaming requirement
  - Input length vs max_input_length
  - Model must be enabled=true

Step 2: SCORE EACH MODEL (_score_models, weighted composite)
  - Health Score (40% weight): healthy=success_rate*100, degraded=success_rate*50, unavailable=0
  - Quota Score (30% weight): (1 - utilization_ratio) * 100
  - Cost Score (20% weight): normalized 0-100 (cheaper=higher), 30% penalty for ultra-cheap (<$0.50)
  - Performance Score (10% weight): HARDCODED PLACEHOLDER 50/100

Step 3: CIRCUIT BREAKER CHECK (within _score_models)
  - OPEN + not half-open -> skip (score = 0)
  - HALF-OPEN -> allow as probe (score * 0.8 penalty)
  - Exhausted quota -> skip (score = 0)

Step 4: SELECT PRIMARY (highest total score after sorting)

Step 5: BUILD FALLBACK CHAIN
  - Pass 1: Different providers (diversity, up to 3)
  - Pass 2: Fill remaining with highest scores (non-skipped)

Step 6: LOG ROUTING DECISION (Firestore: routing_decisions collection)

Step 7: RETURN ModelSelection (model_info, score, reason, estimated_cost, fallback_chain)
```

## Tier-Based Model Selection

### Location: `ModelRouter.get_default_model_for_tier()`

Workers can request a model by tier name instead of going through the full scoring pipeline:
- `get_tts_model_for_tier(tier)` - returns model_id string for TTS
- `get_llm_model_for_tier(tier)` - returns model_id string for LLM
- `get_tts_model_for_tier_and_language(tier, language)` - language-aware TTS selection
- `get_tts_model_for_provider(provider, tier, language)` - provider-specific TTS selection

Language-aware TTS upgrade logic:
- Non-English + ultimate_creator -> eleven_multilingual_v2
- Non-English + pro_creator -> eleven_flash_v2_5
- Non-English + creator -> eleven_turbo_v2_5 or eleven_flash_v2_5
- English or no upgrade available -> tier default

## Health Monitoring

### Location: `kitesforu-workers/src/workers/routing/health.py` (`ProviderHealthMonitor`)

**Health States** (determined in `get_health_status()`):
- **HEALTHY**: success_rate >= 80% AND latency < 2000ms
- **DEGRADED**: success_rate 50-80% OR latency 2000-5000ms
- **UNAVAILABLE**: success_rate < 50% OR latency > 5000ms OR circuit breaker open

**Passive tracking**: Health determined by execution results via `record_execution_result()`. Active health checks have a placeholder (`check_health()` always returns success).

**EWMA smoothing**: success_rate and avg_latency use exponentially weighted moving average (alpha=0.2)

**Storage**: Firestore `model_provider_health/{provider}`

## Circuit Breaker Pattern

### Configuration: `kitesforu-workers/src/workers/config.py` (`CircuitBreakerConfig`)

```
CLOSED -[5 errors in 15min window]-> OPEN -[300s / 5min timeout]-> HALF-OPEN -[2 successes]-> CLOSED
                                                                       |
                                                                   [1 failure]
                                                                       |
                                                                     OPEN (timer resets)
```

Default values (all configurable via env vars):
- `error_threshold`: 5 (`CIRCUIT_ERROR_THRESHOLD`)
- `window_minutes`: 15 (`CIRCUIT_WINDOW_MINUTES`)
- `recovery_attempts`: 10 (`CIRCUIT_RECOVERY_ATTEMPTS`, for normal non-half-open recovery)
- `half_open_seconds`: 300 / 5 minutes (`CIRCUIT_HALF_OPEN_SECONDS`)
- `probe_recovery_threshold`: 2 (`CIRCUIT_PROBE_RECOVERY`)

Manual reset available via `ProviderHealthMonitor.reset_circuit_breaker()`.

## Quota Management

### Location: `kitesforu-workers/src/workers/routing/quota.py` (`QuotaManager`)

**Statuses** (threshold-based):
- **AVAILABLE**: utilization < 80%
- **WARNING**: 80-95%
- **CRITICAL**: 95-99%
- **EXHAUSTED**: 100% (remaining == 0)

**Atomic reservations**: `reserve_quota()` uses Firestore `@firestore.transactional` to prevent race conditions.

**Reset periods**: hourly, daily, or monthly (calculated in `_calculate_next_reset()`)

**Storage**: Firestore `model_quotas/{provider}_{model_id}_{quota_type}`

**Default limit**: 10,000 (if quota doc does not exist)

## Failover System

### Location: `kitesforu-workers/src/workers/routing/failover.py` (`FailoverEngine`)

**Execution chain**: primary + up to 3 fallbacks = max 4 models tried

**Retry config**:
- `max_retries`: 2 (so 3 total attempts per model)
- `initial_retry_delay_ms`: 100
- `retry_backoff_multiplier`: 2 (100ms -> 200ms -> 400ms)

**Error classification** (from `routing/providers/`):
- `ProviderQuotaExceededError`: non-retryable, skip to next provider, open circuit, send CRITICAL alert
- `ProviderRateLimitError`: non-retryable, skip to next provider
- `ProviderAuthenticationError`: non-retryable, skip to next provider
- All other exceptions: retryable with backoff

**Verbose logging**: enabled via `VERBOSE_LOGGING_ENABLED=true` env var.

## Alert System

### Location: `kitesforu-workers/src/workers/routing/alerts.py` (`AlertManager`)

**Alert Types** (from `workers.models.AlertType`):
- QUOTA_WARNING, QUOTA_CRITICAL, QUOTA_EXHAUSTED
- PROVIDER_UNHEALTHY, PROVIDER_UNAVAILABLE
- FAILOVER_EXECUTED
- CIRCUIT_BREAKER_OPENED, CIRCUIT_BREAKER_HALF_OPEN

**Severity levels**: info, warning, critical, emergency

**Destinations** (from `routing/alert_destinations.py`):
- `EmailDestination` (active)
- PagerDuty, Ticketing (commented out, not implemented)

**Rate limiting**: max 5 alerts of same fingerprint per 60-minute window. Fingerprint = hash of (type + severity + provider + model_id).

**Storage**: Firestore `model_router_alerts`

## Integration Layer

### Location: `kitesforu-workers/src/workers/routing/integration.py`

Drop-in functions for workers:
- `generate_script_with_router(prompt, job_id, ...)` -> str (script text)
- `synthesize_tts_with_router(text, job_id, ...)` -> str (audio file path)
- `generate_text_with_failover(prompt, job_id, ...)` -> ProviderResponse (full metadata)
- `get_best_model_for_task(task, context_size, budget)` -> dict (lightweight selection)

These wrap the full router+failover pipeline. Workers call these instead of raw provider APIs.

## Admin Dashboard

### Frontend
- **Page**: `kitesforu-frontend/app/admin/model-router/page.tsx`
- **Route**: `/admin/model-router`

### API
- **Route file**: `kitesforu-api/src/api/routes/admin/model_router.py`
- **Service**: `kitesforu-api/src/api/services/provider_health_monitor.py`
- **Endpoints**: `/admin/model-router/health`, `/quotas`, `/decisions`, `/alerts`, `/summary`

## How to Add a New Model
1. Add a row to `kitesforu-workers/config/model_catalog.csv` with all required fields
2. Required CSV columns: `model_id`, `provider`, `name`, `task_types`, `max_input_length`, `context_window`, `supported_languages`, `supports_streaming`, `cost_per_unit`, `unit_description`, `enabled`, `tier_default`, `purpose_priority`, `supports_tools`, `agentic_capable`, `agentic_cost_multiplier`, `notes`
3. Provider must match a `ModelProvider` enum value (OPENAI, ANTHROPIC, GOOGLE, ELEVENLABS)
4. Task type must match a `TaskType` enum value (LLM, TTS, TRANSCRIPTION)
5. Languages are pipe-separated short codes (e.g., `en|es|fr|de`)
6. Restart the service (CSV parsed on `ModelRouter.__init__` via `ModelCatalogCache`)
7. No code changes needed

## How to Change Tier Defaults
1. Edit the `tier_default` column in `model_catalog.csv`
2. Format: pipe-separated tier names (e.g., `trial|enthusiast|payg`)
3. Special value `all` matches every tier
4. Empty = not a default for any tier (still available via manual selection or scoring)
5. Restart the service

## Key Files

| File | Purpose |
|------|---------|
| `kitesforu-workers/config/model_catalog.csv` | Model definitions (source of truth) |
| `kitesforu-workers/src/workers/routing/router.py` | Selection algorithm, tier-based selection |
| `kitesforu-workers/src/workers/routing/health.py` | Health monitoring, circuit breaker |
| `kitesforu-workers/src/workers/routing/quota.py` | Quota tracking, atomic reservations |
| `kitesforu-workers/src/workers/routing/failover.py` | Failover execution, retry with backoff |
| `kitesforu-workers/src/workers/routing/alerts.py` | Alert management, rate limiting |
| `kitesforu-workers/src/workers/routing/alert_destinations.py` | Email destination implementation |
| `kitesforu-workers/src/workers/routing/integration.py` | Drop-in functions for workers |
| `kitesforu-workers/src/workers/routing/languages.py` | BCP 47 normalization, language support |
| `kitesforu-workers/src/workers/routing/catalog/` | Catalog cache, parser, Google Sheets client |
| `kitesforu-workers/src/workers/routing/providers/` | Provider abstractions (OpenAI, Anthropic, etc.) |
| `kitesforu-workers/src/workers/config.py` | CircuitBreakerConfig, AlertConfig, centralized config |
| `kitesforu-workers/src/workers/common/pricing.py` | Cost calculation |
| `kitesforu-workers/src/workers/common/cost_reporter.py` | Offline cost analysis |
| `kitesforu-api/src/api/routes/admin/model_router.py` | Admin API endpoints |
| `kitesforu-api/src/api/services/provider_health_monitor.py` | API-side health service |
| `kitesforu-frontend/app/admin/model-router/page.tsx` | Admin dashboard UI |

## Known Issues / TODOs
1. **Active health checks are placeholder only** -- `check_health()` always returns success; no actual API pings implemented (TODO in code)
2. **No hot-reload of CSV** -- requires service restart to pick up catalog changes
3. **No model aliasing** for deprecation migrations
4. **Purpose-based scoring not fully implemented** -- purpose_priority column parsed but not used in scoring
5. **Performance score is hardcoded placeholder** (50/100) in `_score_models()`
6. **Language fallback is silent** -- no user-facing warning when falling back to a different TTS model for language support
7. **Quota exhaustion prediction** not implemented -- `_predict_exhaustion()` returns None
8. **Cost estimation is placeholder** -- `_estimate_cost()` returns `cost_per_unit * 0.1`
