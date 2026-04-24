# Activation Runbook — 2026-04-24 Workers Arc (24 PRs)

**Purpose**: a single-page handoff letting an operator activate tonight's shipped work with confidence. 24 workers PRs (#331–354) + 2 docs PRs shipped; **two env-var flips activate the full stack in production**. This doc gives the flip order, validation commands, expected signals, and rollback path.

**Prerequisite**: GCP Cloud Run access to the workers service.

---

## TL;DR

```bash
# Flip 1 (cost): enables Anthropic prompt caching in streaming + non-streaming paths
gcloud run services update kitesforu-worker-audio \
  --region=us-central1 \
  --update-env-vars=ENABLE_ANTHROPIC_PROMPT_CACHING=true

# Flip 2 (quality): enables episode_type classifier + full cascade
gcloud run services update kitesforu-worker-audio \
  --region=us-central1 \
  --update-env-vars=ENABLE_EPISODE_TYPE_CLASSIFIER=true
```

**Apply the same flips** to `kitesforu-worker-script` and `kitesforu-worker-research` / `kitesforu-worker-research-assimilator` / `kitesforu-worker-tools` / `kitesforu-worker-planner` to cover all 5 LLM-calling stages the cache-telemetry was wired into (#351).

**Projected impact** (per `unit_economics_tracking/README.md`):
- Flip 1: ~$2.0–3.4k/mo savings at 1k fiction ep/day (78% of prompt tokens cacheable, 90% discount on cache reads)
- Flip 2: episode-type-specialized craft + post-gen verification on ~5-10% of titles whose phrasing unambiguously signals final_prep / autopsy / pressure / drill / retrieval / recognition / lecture

---

## What each flag does

### `ENABLE_ANTHROPIC_PROMPT_CACHING=true`

Activates the split-prompt pattern in the Anthropic provider:
- Static portion (rules, craft, format) → `system` block with `cache_control: {type: "ephemeral"}` — cached for 5 min at 90% discount.
- Dynamic portion (topic, outline, research, user profile) → user message — not cached.

Shipped in PRs #333 (Anthropic streaming), #334 (non-streaming + OpenAI parity + structured telemetry), #346 (Firestore persistence), #351 (cross-stage wiring across 5 LLM-calling stages: script, research, assimilator, planner, agentic).

**Where it takes effect**: every `AnthropicProvider.generate_text_stream` and `AnthropicProvider.generate_text` call made by workers. OpenAI path (#334) is always on — no flag needed — and reports `cached_tokens` via `prompt_tokens_details`.

### `ENABLE_EPISODE_TYPE_CLASSIFIER=true`

Activates the conservative phrase-match classifier in `workers.common.episode_type_classifier.classify_episode_type` inside `_resolve_episode_type` at the composer-context build step.

Resolves an `episode_type` when the episode's title/topic unambiguously matches one of 7 documented values (drill / retrieval / pressure / autopsy / lecture / recognition / final_prep). The classifier **writes the decision back to `preferences["_episode_type"]`** (#353 fix) so ALL downstream consumers (composer, quality gate, regen policy, debug logs) see the same value.

Shipped in PRs #330 (plumbing), #337 (EpisodeTypeSpecializationBuilder + 5 craft blocks), #338 (final_prep gate), #340 (preference threading on secondary), #342 (classifier heuristic), #350 (primary-path threading), #352 (remaining callers), #353 (write-back fix), #354 (E2E cascade test).

**Where it takes effect**:
1. Classifier runs during prompt composition
2. `EpisodeTypeSpecializationBuilder` activates matching craft block in system prompt
3. `run_quality_gate(episode_type=...)` fires episode_type-gated checks (final_prep_no_new_material)
4. Violations → `should_regenerate` promotes to regen with targeted hint
5. `diff_regen_reasons` persists `regen_resolved_reasons` / `regen_persisted_reasons` to Firestore

---

## Validation commands

Both flips can be validated without manual log inspection:

### Validate caching is actually hitting

```bash
cd /path/to/kitesforu-workers
python scripts/analyze_regen_effectiveness.py --recent 100
```

Look for:
- `cache_read_tokens > 0` in structured log fields (Cloud Logging query: `jsonPayload.cache_read_tokens > 0 AND resource.labels.service_name="kitesforu-worker-audio"`)
- In Firestore: `podcast_jobs/<job_id>/llm_call_logs[].cache_read_tokens` populated

**Expected hit rate** (per unit_economics_tracking §"Realized-savings telemetry"):
- Fiction dialogue (high volume, stable shape): 85-95%
- Narration: 60-80%
- Cold-start after deploy: 20-30% for first ~10 min

### Validate classifier is firing

Create a test job with title "Night before your interview: a calm rehearsal" and check:
- Firestore `podcast_jobs/<job_id>/preferences._episode_type` == `"final_prep"` (written by classifier per #353)
- Firestore `podcast_jobs/<job_id>/stages.job-script.sections_included` contains `"episode_type_specialization"` (composer activated the craft block)
- If the generated script contains "here's a new framework" or "one more technique":
  - `stages.quality_gate.final_prep_no_new_material.new_material_hits` has entries
  - `stages.quality_gate.regen_attempted == true`
  - `stages.quality_gate.regen_reason` contains `"final_prep_new_material=..."`
  - After retry: `stages.quality_gate.regen_resolved_reasons` contains `"final_prep_new_material"` if regen fixed it

### Aggregate resolution rates across jobs

```bash
python scripts/analyze_regen_effectiveness.py --recent 500
```

Prints a per-reason resolution-rate table. Used to answer "which gate hints actually fix their violations on retry?"

---

## Expected signals post-flip

### Flip 1 (caching)

| Signal                                          | Where to watch                     | Expected                         |
|:------------------------------------------------|:-----------------------------------|:---------------------------------|
| Daily LLM spend                                 | GCP Billing                        | Drops 30-50% over 24h            |
| `cache_read_tokens` in logs                     | Cloud Logging structured fields    | > 0 on ~60-80% of LLM calls      |
| `cache_creation_input_tokens` in logs           | Cloud Logging structured fields    | > 0 on first call of each shape, then drops to ~0 |
| Firestore `llm_call_logs[].cache_read_tokens`   | Debug page per job                 | Populated on Anthropic calls     |
| Latency                                         | Cloud Run metrics                  | Slight drop (cached reads are faster) |

### Flip 2 (episode_type classifier)

| Signal                                          | Where to watch                         | Expected                         |
|:------------------------------------------------|:---------------------------------------|:---------------------------------|
| Classifier firing rate                          | Firestore `preferences._episode_type` populated | ~5-10% of titles |
| Craft activation rate                           | Firestore `sections_included` contains `episode_type_specialization` | matches classifier rate |
| Gate firing rate on classifier-triggered jobs   | Firestore `stages.quality_gate.final_prep_no_new_material.issues` | fires when violations present |
| Regen resolution rate                           | `python scripts/analyze_regen_effectiveness.py` | > 70% target |

---

## Rollback

Both flags default to OFF. Roll back with a single command:

```bash
# Rollback caching
gcloud run services update kitesforu-worker-audio \
  --region=us-central1 \
  --remove-env-vars=ENABLE_ANTHROPIC_PROMPT_CACHING

# Rollback classifier
gcloud run services update kitesforu-worker-audio \
  --region=us-central1 \
  --remove-env-vars=ENABLE_EPISODE_TYPE_CLASSIFIER
```

**No code rollback needed** — both flags are additive and code paths default-OFF. Existing jobs continue unchanged.

---

## Per-job kill switches

Beyond the service-wide env vars, both flags can also be controlled per-job via preferences:

- `preferences["_use_prompt_caching"] = False` → forces caching OFF for that job even if env var is ON (e.g. for debugging)
- `preferences["_classify_episode_type"] = False` → forces classifier OFF
- `preferences["_episode_type"] = "drill"` → explicit override always wins; classifier short-circuits

See `workers/stages/script/streaming_generator.py::_resolve_episode_type` for the full precedence logic (explicit preference > env var > classifier result > empty).

---

## Dependency graph of tonight's 24 PRs

For reviewers understanding the arc shape:

```
Infrastructure:
  #331 ContentTypeCraftBuilder (bridge YAMLs into composer)
     ↓
  #332 Fantasy deep-craft (10th content-type)
     ↓
Caching (cost lever):
  #333 Anthropic streaming caching
     ↓
  #334 Non-streaming + OpenAI + telemetry
     ↓
  #346 Worker cache telemetry → Firestore
     ↓
  #351 Cross-stage cache telemetry (all 5 stages)
     ↓
Episode_type axis (quality lever):
  #330 PromptContext.episode_type field + PauseAndTryBuilder dual gate
     ↓
  #337 EpisodeTypeSpecializationBuilder (5 dormant values activated)
     ↓
  #338 Final_prep no-new-material gate check + episode_type signature on gate
     ↓
  #339 Regen policy wires all 3 new gate checks
     ↓
  #340 _episode_type preference threading (secondary path)
     ↓
  #342 Classifier heuristic
     ↓
Audit fixes (regression-guard arc):
  #341 Content_types loader coverage sweep (175 sections)
     ↓
  #344 Loader order-preservation pin
     ↓
  #343 Regen effectiveness diff + Firestore persistence
     ↓
  #347 RegenEffectivenessReporter CLI + offline analysis
     ↓
  #350 Primary-path episode_type + regen-diff fix
     ↓
  #352 Remaining run_quality_gate callers episode_type fix + sweep guard
     ↓
  #353 Classifier write-back to preferences fix
     ↓
  #354 End-to-end cascade test (10-hop smoke)

Gate coverage (7 content types):
  bedtime energy-arc (pre-session)
  motivational rumination (#335)
  horror ambient-markers (#336)
  final_prep no-new-material (#338)
  documentary sourcing (#345)
  comedy trope-zones (#348)
  romance POV-announcements (#349)
```

---

## What's NOT included in tonight's arc

Transparent about scope limits so operators don't expect what wasn't shipped:

- **Mystery, scifi, fantasy, true_crime semantic gates** — deferred; require LLM-as-judge which violates the "no extra per-request inference" directive.
- **Tier B #5 interview-prep episode-type picker UI** — multi-repo (frontend + API), 2-3 day effort; workers-side contract is fully established, UI layer awaits fresh session.
- **Homepage simplification, UX-language critique, content-strategy deep-dive** — explicitly deferred to `--ultrathink --seq` sessions with multiple specialized research agents; wrong tool for tonight's shipping-focused session.

---

## Next sessions to pick up

Ordered by impact × readiness:

1. **Post-flip data review** (this activation). Collect 24h of real traffic post-both-flips, then update `unit_economics_tracking/README.md` with actual cache-hit rates and classifier firing rates. Replaces my projections with measured numbers.
2. **Tier B #5 picker UI** once flip 2 validates classifier precision is ≥80%. Picker extends activation from 5-10% of titles to 100% for users who explicitly select.
3. **Mystery/scifi/fantasy gates** — accept the LLM-as-judge cost and evaluate whether quality gain justifies. Requires a separate cost-benefit analysis.
4. **UX/content deferred threads** — one fresh `--ultrathink --seq` session each.
