# R2 — Quality Auto-Regeneration Loop

**Status**: PHASE 1 SHIPPED (regular script worker), Phase 2 pending (streaming worker bad-path-serialize)
**Priority**: P1
**Effort**: ~1 week (workers only)
**Origin**: 2026-04-19 strategy pass — highest-leverage option (touches every episode, infrastructure already built)

## Implementation summary (Phase 1 — regular script worker, 2026-04-20)

**Shipped**:
- `kitesforu-workers` PR #289 — `regen_policy.py` module with `should_regenerate`, `build_regen_hints`, `regen_prompt_preamble`, `bump_temperature`. Critical thresholds 10–15% above the existing warn band. 26 unit tests cover each boundary.
- `kitesforu-workers` PR #293 — wired into the regular script worker (`stages/script/worker.py`). After `_generate_script` produces a script, `_run_quality_gate_and_maybe_regen` runs the gate; if critical, rebuilds the prompt with imperative hints threaded into `preferences['_custom_instructions']` (zero template change) plus `preferences['_quality_warnings']` for the streaming generator's consumer, then calls `_generate_script` once more. Writes `stages.quality_gate` with `attempt_1_metrics` + `attempt_2_metrics` + `regen_status` ('improved' | 'no_improvement' | 'regen_failed' | 'not_triggered') in a single `.update()` call.

**Variations from the proposal**:
- **No temperature bump in v1**: the script worker's failover engine picks model + temp internally. Plumbing override requires ~3 additional helper signatures; deferred. Hint threading alone is the primary lever.
- **Script worker does not read `_quality_warnings` directly** (streaming_generator does). We prepend hints into `_custom_instructions` with `regen_prompt_preamble()` so every script prompt picks it up without template changes.
- **Cost/temp tracking**: we write `regen_count` + `regen_hints_sent` to Firestore for debug visibility but do not cap per-job regen spend separately yet (v1 trusts the existing `budget_limit` check inside `_generate_script`).

**Phase 2 (pending)**: Streaming worker (`streaming_script_audio_worker.py`) wire-in using the **bad-path-serialize** strategy. The streaming pipeline interleaves LLM + TTS so a clean pre-TTS checkpoint does not exist; the fix is: stream as today, but when the last script chunk closes, run the gate, cancel pending / in-flight TTS tasks on CRITICAL failure, regenerate non-streamingly, then re-run TTS. Substantial architectural change vs. the Phase 1 additive path. Separate PR.

**Deferred follow-ups**:
- Temperature bump via the failover engine.
- Per-job regen spend cap.
- Regen signals for the Car Mode segment worker (third script code path per workers rule #8; currently runs neither the gate nor any regen).

---

## Problem

`run_quality_gate` already runs post-generation and classifies issues: AI-tell phrases, emotion variety, speaker-balance skew, conversation flow, tension curve. Today it LOGS findings and writes metrics to Firestore but does not act on them. Users still receive the low-quality output.

## Proposal

Conditional re-generation loop: when the validator flags **critical** issues, regenerate the script once with a stronger prompt that specifically addresses the flagged issues, then re-validate. If attempt 2 still fails, ship with a quality note attached.

The existing prompt hook at `streaming_generator.py:538-547` already consumes `preferences["_quality_warnings"]` and renders a `## QUALITY ISSUES TO AVOID` block. We thread the regen hints into that existing hook — no template changes needed.

## Why this option

- **Infrastructure already exists**: validator classifies issues, prompt hook injects warnings. The gap is a conditional re-trigger.
- **Touches every episode**: compounds over time across all content types.
- **Unblocked**: workers-only change. No frontend, no design debate.
- **Invisible to user** (no behavior change) but visible in retention curves.

## Trigger thresholds (`critical_failure`)

Trigger regen if ANY of:

- `ai_tell.count >= 3` (today's gate warns on `phrase_count > 0` — too noisy for regen)
- `emotion.neutral_ratio > 0.80` OR `emotion.distinct_count < 3` for `total_segments >= 10`
- `speaker_balance.dominant_share > 0.75` for dialogue formats (stricter than the existing 0.70 warn)
- `conversation_flow.echo_ratio > 0.30` when present
- `tension_curve.flatness > 0.7` when present

Thresholds sit ~10–15% above the existing WARN line — regen is expensive, so the critical band should be tighter than the log-only warn band.

Do NOT trigger on `naturalness` warnings — those are already auto-fixed by `validate_and_fix`.

## Implementation

### New module
- `src/workers/stages/audio/voice_intelligence/quality/regen_policy.py`
  - `should_regenerate(gate_result, attempt_count) -> RegenDecision` — returns reason list + bool
  - `build_regen_hints(gate_result) -> list[str]` — translates metrics into imperative strings that format-match the existing `_quality_warnings` consumer

### Wire-in point (architectural constraint — see below)
Today's gate runs at `streaming_script_audio_worker.py:460` — AFTER audio is built. That's too late for regen because TTS spend is already committed.

Naïve approach: add a **script-only** gate call BEFORE TTS workers dispatch. But this does not work as-is because `_run_streaming_pipeline` interleaves script-chunk generation with TTS enqueue (the whole point of streaming is that TTS starts before the full script is ready). There is no clean "script done, TTS not started" checkpoint in the current pipeline.

### Wire-in architecture — three strategies considered

1. **Bad-path serialize (recommended for v1).** Stream as today. When the *last* script chunk closes, run the quality gate but block the audio-combine step. On CRITICAL failure, cancel all pending / in-flight TTS tasks for that job, regenerate the script **non-streamingly** with regen hints, then re-run TTS on the new script. Happy path (≥85% of jobs) keeps full streaming speed; only the small bad-path tail serializes. Adds ~15–25s latency to the bad tail, saves a ~$0.04 TTS spend on the discarded first pass.

2. **Full buffer.** Buffer the entire script before TTS dispatch, gate, regen if needed, then start TTS. Gives clean pre-TTS gating but kills streaming for everyone (~30–60s user-visible delay on happy path). Rejected.

3. **Two-stage (speculative + confirm).** Start TTS on early chunks speculatively, gate at chunk boundary midway through, cancel + regen remainder if the partial gate fails. Much more complex; gate signals are mostly whole-script (emotion variety, speaker balance) and don't give useful partial signal. Deferred.

**v1 ships strategy 1.** The regular (non-streaming) script worker already has the clean checkpoint — mirror the loop there first as the reference implementation, then port the bad-path-serialize variant to the streaming worker.

Wrap script generation in a loop (`max_attempts=2`). On attempt-1 failure:
1. Merge `build_regen_hints(...)` into `preferences["_quality_warnings"]`
2. Prepend `"PREVIOUS ATTEMPT FAILED QUALITY GATE — MUST FIX:"` to distinguish from cross-job warnings
3. Bump `script_temperature` × 1.15 (clamped to 0.9) to break out of the local minimum
4. Re-invoke the generator
5. Ship attempt 2 regardless of its gate result

### Parity for regular script worker
Per workers rule #4 ("streaming generator and regular script worker are separate code paths; fixes to one don't apply to the other"), mirror the same loop in `src/workers/stages/script/worker.py:405 (_generate_script)`. The regular worker currently does NOT call the quality gate at all — add the gate call AND the regen loop.

## Failure-mode handling

- **Regen LLM timeout / exception** → catch, log, ship attempt 1 with `regen_status = "regen_failed"`.
- **Attempt 2 still fails gate** → ship attempt 2 (strictly more information than attempt 1) with `quality_note` populated.
- **Attempt 2 fails a DIFFERENT check** → still ship; write both failure sets to `regen_deltas`. Do NOT loop again.
- **Budget exhausted** → skip regen entirely (check `script_budget` before second call).

## Firestore fields (under `stages.quality_gate`)

```
regen_attempted: bool
regen_reason: list[str]          # ["ai_tell_count=4", "speaker_dominance=0.82"]
regen_count: int                 # 0 or 1
regen_hints_sent: list[str]      # what we told the LLM to fix
regen_status: "not_triggered" | "improved" | "no_improvement" | "regen_failed"
attempt_1_metrics: {...}
attempt_2_metrics: {...}
quality_note: str | null         # user-visible note if shipped with residual issues
```

All fields visible on the debug page automatically via the existing quality_gate view.

## Cost

Script generation is ~$0.02 of the $0.083/ep pipeline. At ~15% trigger rate and ~1.1× attempt-2 cost (temp bump + longer prompt):

**Amortized incremental cost: ~$0.0033/ep** — well under the $0.008/ep ceiling from the strategy doc.

## Acceptance criteria

- [ ] New `regen_policy` module with unit-tested `should_regenerate` + `build_regen_hints`
- [ ] Regular script worker wires gate + regen loop first (reference implementation, clean pre-TTS checkpoint)
- [ ] Streaming worker adopts bad-path-serialize strategy: gate at end of last script chunk, cancel pending TTS on CRITICAL failure, regenerate non-streamingly, re-run TTS
- [ ] Regular script worker mirrors the same loop
- [ ] Regen hints reach the script LLM via existing `_quality_warnings` preference hook (no template changes)
- [ ] Firestore fields (`regen_*`, `attempt_*_metrics`, `quality_note`) populate as specified
- [ ] Unit tests cover each threshold boundary (`tests/unit/test_regen_policy.py`)
- [ ] Integration test forces a bad first pass and asserts loop ran twice with hints propagated (`tests/integration/test_regen_loop.py`)
- [ ] Integration test for failure path: both attempts bad → ship succeeds with `regen_status="no_improvement"` and `quality_note` set
- [ ] Beta-verify per workers rule #11: create test job, inspect debug page `stages.quality_gate.regen_*`, listen to audio

## Key file references

- Validator: `kitesforu-workers/src/workers/stages/audio/voice_intelligence/quality/quality_gate.py` (`run_quality_gate` L79, thresholds L60 / L63 / L133 / L138 / L148)
- Current post-audio wire-in: `kitesforu-workers/src/workers/stages/combined/streaming_script_audio_worker.py:460` and `:1145`
- Streaming generator entry: `kitesforu-workers/src/workers/stages/script/streaming_generator.py:71` (`generate_streaming`), `:303` (`generate`)
- Prompt hook already wired: `kitesforu-workers/src/workers/stages/script/streaming_generator.py:538-547`
- Regular script worker (needs wire-in): `kitesforu-workers/src/workers/stages/script/worker.py:405` (`_generate_script`)
- AI-tell detector (genre-aware): `kitesforu-workers/src/workers/common/ai_tell_detector.py`

## Out of scope

- Changing existing WARN thresholds (they stay as-is — this proposal only adds the CRITICAL band and regen loop on top).
- Surfacing regen status to the end user on the creation UI (deferred; debug-page visibility is enough for v1 — user sees only the improved audio).
- Multi-attempt loops beyond 2 (diminishing returns; same local minimum risk).
- Applying the same loop to episode-outline generation (different failure modes; separate proposal if needed).
