# R2 — Series Memory: Full Wiring

**Status**: PROPOSED (awaiting Codex audit before any code per triangulation rule)
**Priority**: P1 (closes R2-phase1 D2; unblocks "Continue series" UX for Bedtime Parent / Horror Worldbuilder / Romance personas)
**Effort**: ~2 weeks (workers + frontend + schemas; phased)
**Affected repos**: `kitesforu-workers` (analyze_episode + Firestore writes + composer-context plumbing), `kitesforu-api` (course endpoint + series_id resolution), `kitesforu-frontend` (Continue Series CTA on detail page), `kitesforu-schemas` (Course.series_id + SeriesState models)
**Depends on**: R2-phase1 D3 content rating (shipped), persona_analyzer scaffolding (4 helper functions exist; orchestrator missing), Anthropic prompt caching (the static series-state block is cacheable across episodes within the same run).
**Origin**: R2-phase1 D2 close-out summary at `done/R2-phase1-content-quality-voice.md` named series memory as the dedicated thread; per memory `analyze_episode()` is missing (stub docstring only), no `series_id` concept exists anywhere in the codebase.

---

## 1. One-paragraph thesis

`persona_analyzer.py` was scaffolded as the replacement for the deleted `series_memory.py` regex implementation. Four supporting functions are in place — `build_analysis_prompt`, `build_continuity_prompt`, `parse_llm_response`, `analysis_to_firestore` — but the top-level orchestrator `analyze_episode()` (referenced only in a docstring example at line 15) was never written. There is also no `series_id` concept anywhere in the codebase: no field on `Course`, no `series` Firestore collection, no script-side prompt section that injects prior-episode context. To deliver the original R2-phase1 D2 user value ("Episode N references characters and events from episodes 1..N-1, character names consistent, plot threads carry forward"), this proposal adds the missing series_id (additive, defaults from courseId for new series), implements `analyze_episode()` with a single LLM call gated to the LAST segment of an episode's audio post-completion, persists `SeriesState` to a new Firestore collection, threads it into the composer's `PromptContext` for the next episode's generation, and adds a "Continue series" frontend CTA. Cost: ~$0.002 per analysis (gpt-4o-mini, async post-completion, never blocks the user). Latency: ~1-2s but off the critical path.

## 2. Goals / Non-goals

### Goals

1. Episode N's script generation receives a structured "what came before" block — character names + traits, relationship dynamics, plot threads, world rules, last-episode emotional state — drawn from the analyzed state of episodes 1..N-1.
2. Character names stay consistent ("Mara stays Mara"; no "Sarah" appearing in episode 3 for the same role).
3. Plot threads carry forward (the missing lighthouse keeper introduced in episode 1 doesn't disappear by episode 4 unless deliberately resolved).
4. World rules persist ("Magic costs memory" stays true; the LLM doesn't contradict it episode-to-episode).
5. Frontend exposes a "Continue series" button on the detail page that creates the next episode with prior context attached.
6. The system is genre-aware: Bedtime + Romance + Horror + Mystery + Sci-Fi all use it; Comedy uses a lighter version (callback structure rather than full character continuity).

### Non-goals

- **Multi-track series** (parallel storylines across episodes — different protagonists, recurring side characters). v1 is single-track linear continuity.
- **User-authored canon** (let user pin "Mara never dies" as a rule). Future feature; v1 trusts the analyzer.
- **Cross-series memory** (universe-level lore across multiple series — Marvel-style). Out of scope.
- **Real-time series state preview** during generation. The analyzer runs ASYNC post-completion; the user sees the result on the next episode generation, not the current one.
- **Series memory for explainer/educational content.** The thread is genre-gated to fiction + storytelling; educational content's continuity is structural (curriculum order), not narrative.
- **Retrofit analysis on already-shipped episodes.** v1 only analyzes episodes generated AFTER the wiring lands. A backfill pass is a follow-up if it's worth it.

## 3. Current surface — file:line references

| Concern | Location |
| --- | --- |
| Analyzer scaffolding (orchestrator missing) | `kitesforu-workers/src/workers/common/persona_analyzer.py` (293 lines) |
| `analyze_episode()` referenced only in docstring | `persona_analyzer.py:15` (example usage in module docstring) |
| `build_analysis_prompt` (single-LLM call prompt) | `persona_analyzer.py:115` |
| `build_continuity_prompt` (next-episode context block) | `persona_analyzer.py:140` |
| `parse_llm_response` | `persona_analyzer.py:188` |
| `analysis_to_firestore` | `persona_analyzer.py:252` |
| Firestore path target | `series_persona/{series_id}/episodes/{episode_number}` (referenced in line 255 docstring) |
| `series_id` grep result | **ZERO matches** anywhere in workers/api/frontend/schemas (verified 2026-04-24) |
| Streaming script worker (where context injection lands) | `kitesforu-workers/src/workers/stages/combined/streaming_script_audio_worker.py` |
| PromptComposer | `kitesforu-workers/src/workers/prompts/composer/composer.py` |
| Course schema | `kitesforu-schemas/src/kitesforu_schemas/courses.py` |
| Course detail page (Continue Series CTA target) | `kitesforu-frontend/app/courses/[courseId]/page.tsx` |

## 4. Architecture

### 4.1 Schemas — `series_id` + `SeriesState`

```python
# kitesforu-schemas/courses.py — additive
class Course(BaseModel):
    model_config = ConfigDict(extra='ignore')
    # ...existing fields...
    series_id: Optional[str] = None   # set on series-marked courses; default
                                       # = course.id for the FIRST episode of a
                                       # new series. Subsequent episodes share
                                       # the same series_id.
    series_episode_number: int = 1     # 1-indexed; 1 = first episode of series

# kitesforu-schemas — new file `series.py`
class SeriesState(BaseModel):
    model_config = ConfigDict(extra='ignore')
    series_id: str
    last_analyzed_episode: int          # episode number of most recent analysis
    characters: list[CharacterProfile]  # cumulative across episodes
    relationships: list[RelationshipDynamic]
    narrative_threads: list[NarrativeThread]
    world_rules: list[str]              # ["Magic costs memory", ...]
    continuity_prompt: str              # cached prompt block for next episode
    updated_at: datetime
```

`SeriesState` Firestore path: `series_state/{series_id}` (NOT the `series_persona/{series_id}/episodes/{n}` path the docstring mentions — that's the per-episode raw analysis; the rolled-up state is the thing the next episode reads). Both writes happen.

### 4.2 Workers — `analyze_episode()` orchestrator

```python
# kitesforu-workers/src/workers/common/persona_analyzer.py — new top-level

async def analyze_episode(
    dialogue_items: list[DialogueItem],
    *,
    genre: str,
    episode_number: int,
    series_id: str,
    prior_state: SeriesState | None,  # cumulative state from episodes 1..N-1
) -> PersonaAnalysis:
    """Analyze a completed episode for character/relationship/plot data.

    Single LLM call (~$0.002, gpt-4o-mini, ~500 input + 300 output tokens).
    Runs async post-audio-completion; never blocks the listener.
    """
    prompt = build_analysis_prompt(dialogue_items, genre, episode_number, prior_state)
    response = await call_llm(prompt, model="gpt-4o-mini", temperature=0.2)
    analysis = parse_llm_response(response.text, episode_number, genre)
    # Merge with prior_state: take new characters/threads, update existing
    # ones with delta, retire resolved threads.
    new_state = merge_with_prior(analysis, prior_state)
    new_state.continuity_prompt = build_continuity_prompt(new_state)
    return analysis, new_state
```

### 4.3 Workers — wire-in point

The analyzer runs **post-audio-completion**, after the assembled MP3 lands in GCS but before the Firestore status flips to `completed`. This makes the analyzer cost asymmetric — only successful episodes get analyzed; failures are not charged.

```python
# streaming_script_audio_worker.py — at the end of _run_streaming_pipeline
if ctx.series_id and ctx.episode_number > 0 and _is_continuity_genre(ctx.genre):
    asyncio.create_task(_analyze_and_persist(
        dialogue_items=collected_segments,
        ctx=ctx,
        firestore_client=firestore_client,
    ))
```

Failure handling: analyzer task swallows all exceptions and logs structured. Series state failing to update is NOT a job failure — the next episode just falls back to "no prior context, use the topic alone."

### 4.4 Composer — context injection

```python
# kitesforu-workers/src/workers/prompts/composer/builders/series_continuity_builder.py
class SeriesContinuityBuilder(SectionBuilder):
    priority = 220  # AFTER ContentRating (105), BEFORE ContentTypeCraft (250)
    name = "series_continuity"

    def build(self, ctx: PromptContext) -> str | None:
        if not ctx.series_state or not ctx.series_state.continuity_prompt:
            return None
        if ctx.episode_number <= 1:
            return None
        if not _is_continuity_genre(ctx.genre):
            return None
        return f"""
# SERIES CONTINUITY — what came before

{ctx.series_state.continuity_prompt}

Honor every named character, plot thread, and world rule above. If you
introduce a contradiction, you have failed this episode.
""".strip()
```

The block is silently omitted when no prior state exists, when this is episode 1, or when the genre doesn't qualify. This keeps the proposal additive — every existing prompt path that doesn't have a series simply renders identically.

### 4.5 API — series creation + episode endpoint

```python
# kitesforu-api — new endpoint
@router.post("/v1/courses/{course_id}/continue-series")
async def continue_series(course_id: str, user: AuthedUser):
    parent = await get_course(course_id, user.id)
    if not parent.series_id:
        # First continuation creates the series_id (= parent course id)
        parent.series_id = parent.id
        await update_course(parent)
    next_episode_number = parent.series_episode_number + 1
    new_course = await create_course(
        topic=parent.topic,
        genre=parent.style,
        series_id=parent.series_id,
        series_episode_number=next_episode_number,
        # ...inherit smart_create_outline, persona, etc.
    )
    return new_course
```

The new course inherits parent metadata; the workers stage reads `series_state/{series_id}` and injects the continuity block.

### 4.6 Frontend — Continue Series CTA

`/courses/{courseId}/page.tsx` — when `course.series_id` is set OR `_is_continuity_genre(course.style)`, show a "Continue series" button next to "Play All". On click → `POST /v1/courses/{id}/continue-series` → redirect to the new course's detail page.

For Bedtime Parent persona: the CTA reads "Tell another Mermaid Zara adventure" (interpolating the analyzed primary character name). Falls back to "Continue series" if no character data yet.

## 5. Phased scope

### Phase 1 — schema + analyzer + Firestore writes (1 week)

- schemas: `series_id` + `series_episode_number` on `Course`; new `SeriesState` model.
- workers: implement `analyze_episode()` orchestrator + `merge_with_prior()`.
- workers: wire async post-completion task in streaming pipeline.
- workers: composer `SeriesContinuityBuilder` (silent when no state).
- Firestore writes to `series_state/{series_id}` + `series_persona/{series_id}/episodes/{n}`.
- Unit tests for analyzer (mocked LLM); integration test for state merge across 3 episodes.

### Phase 2 — API endpoint + frontend CTA (3-4 days)

- `POST /v1/courses/{id}/continue-series` endpoint.
- Frontend "Continue series" button on course detail page (genre-gated).
- Persona-aware CTA copy ("Tell another Mermaid Zara adventure" when character analysis available).

### Phase 3 — quality + observability + comedy variant (3-4 days)

- Comedy-specific lighter analysis (callback structure, not full character continuity).
- Quality gate check: episode N's script must reference at least 1 character from prior episode (silent gate, not a hard fail).
- Firestore debug page surfaces: `series_id`, `last_analyzed_episode`, character names, world rules.
- Operator script: `python scripts/show_series_state.py {series_id}` prints the cumulative state for ops debugging.

## 6. Acceptance criteria

- [ ] `Course.series_id` + `series_episode_number` shipped via schemas bump (additive, defaults preserve legacy).
- [ ] `analyze_episode()` implemented, single LLM call, ~$0.002, async post-completion.
- [ ] `series_state/{series_id}` Firestore document written after every successful continuity-genre episode.
- [ ] `SeriesContinuityBuilder` injected at composer priority 220; silent when no prior state.
- [ ] Episode 2 of a series references at least one character name from episode 1 (verified by integration test).
- [ ] Episode 3 of a series respects world rules established in episodes 1–2 (verified by adversarial test: prompt the LLM to violate a rule, assert it doesn't).
- [ ] `POST /v1/courses/{id}/continue-series` creates a new course with `series_id` matching the parent + `series_episode_number = parent + 1`.
- [ ] Frontend "Continue series" CTA visible on detail pages for continuity genres (Bedtime / Horror / Romance / Mystery / Sci-Fi); persona-aware copy when character data is available.
- [ ] Comedy uses lighter callback-structure analyzer (not full character continuity).
- [ ] Failure of the analyzer task does NOT mark the parent episode as failed.
- [ ] Failure of the analyzer task DOES log a structured error.
- [ ] Cost telemetry: every analyzer run writes `(input_tokens, output_tokens, cost_usd)` to Firestore for ops visibility.
- [ ] Playwright E2E on `beta.kitesforu.com`: create a 2-episode bedtime story → verify episode 2's script references the protagonist by name.
- [ ] Workers + api + frontend hard-block test gates pass.

## 7. Risks & mitigations

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Analyzer hallucinates a character that doesn't exist in the script | Medium | gpt-4o-mini at temp 0.2 + structured JSON output schema. Validate parsed names appear in `dialogue_items` before persisting. |
| State grows unbounded across many episodes (10+ episodes of horror) | Medium | Cap `characters` at 10 (prune by recency × importance), `narrative_threads` at 5 (resolved threads age out), `world_rules` at 8 (LLM-prompted to consolidate before adding). |
| Two episodes generated concurrently in the same series corrupt state | Low | Firestore transaction: read state → analyze → write back ONLY if `last_analyzed_episode == read_value` (optimistic lock). Conflicts re-queue. |
| Continuity prompt grows beyond the prompt cache TTL → cost spike | Low | Continuity block is part of the dynamic suffix (per-job), not the static prefix. So no caching impact, but also no surprise. Cost is bounded by output tokens (each episode generates ~30k output regardless). |
| User starts a series, analyzer fails on episode 1, episode 2 has no context | Medium | Composer falls back silently to "no continuity" block. Subsequent successful analyses recover the chain naturally. |
| Genre gate too narrow — user creates a "drama" series and gets no continuity | Medium | `_is_continuity_genre()` includes drama, horror, romance, bedtime, mystery, sci-fi, comedy (lighter), thriller. Operator-tunable list. |
| Comedy's lighter analyzer drifts into full character continuity | Medium | Separate analyzer prompt for comedy; explicit "do not enforce character names across episodes" clause. |
| Firestore reads during composer build add latency | Low | One read per episode, ~50 ms. Off-critical-path because the prompt assembly is already paying for content_types YAMLs. |

## 8. Out of scope (recap)

- Multi-track / parallel-storyline series.
- User-authored canon pinning.
- Cross-series universe lore.
- Real-time state preview during generation.
- Educational / explainer continuity (their continuity is curriculum order).
- Backfill of already-shipped episodes.

## 9. Codex audit asks

1. **Continuity-genre allowlist** — which genres get full character continuity, which get the lighter comedy variant, which get nothing? Default proposal: bedtime + horror + romance + mystery + sci-fi + drama + thriller = full; comedy = lighter; everything else = none. Confirm.
2. **`series_id` semantics on the FIRST course** — default proposal sets `series_id = course.id` only when the user clicks "Continue series." Alternative: every continuity-genre course gets a `series_id` at creation time (so the first episode is also analyzed for future continuity). Cost trade — pre-analyzing episodes that never get a continuation burns ~$0.002 per orphaned episode.
3. **Concurrent-generation race** — the Firestore optimistic lock approach assumes most series are linear (user finishes episode 2 before requesting episode 3). If users batch-request 3 episodes at once, lock conflicts retry but episodes 2 and 3 might both have only episode 1's state. Acceptable? Or should we serialize series-episode generation at the worker level?
4. **Character-name pinning hard guarantee** — should the quality gate hard-fail an episode that contradicts a prior character name, or silent-warn? Default proposal: silent-warn first, escalate to hard-fail in Phase 3 if the LLM's compliance rate is < 90%.
5. **State pruning thresholds** — 10 chars / 5 threads / 8 world rules. These are guesses. Do we need adaptive caps (long-running horror series sustain more) or is a hard cap fine?
6. **API endpoint path** — `/v1/courses/{id}/continue-series` (default) vs `/v1/series/{series_id}/next-episode` (more series-centric, but no series resource exists yet). Confirm.

## 10. Rollback

Phase 1 ships behind `feature_series_memory` (default OFF in workers). When OFF: `analyze_episode()` is not called; `SeriesContinuityBuilder` returns None. Composer behaves identically to today. Phase 2 ships behind `feature_continue_series_cta` in frontend (default OFF). Phase 3 hard-fail gate ships behind `feature_series_continuity_gate` (default OFF). Each phase is independently rollbackable.

## 11. Sources

- `kitesforu-workers/src/workers/common/persona_analyzer.py` — scaffolding the proposal completes
- `persona_analyzer.py:8` — confirms it replaces the deleted `series_memory.py`
- `persona_analyzer.py:15` — docstring reference to `analyze_episode()` (function not implemented)
- `persona_analyzer.py:255` — Firestore path target
- `kitesforu-workers/src/workers/stages/combined/streaming_script_audio_worker.py` — wire-in target
- `done/R2-phase1-content-quality-voice.md` — phase close-out that named this thread
- `proposed/unit_economics_tracking/README.md` — caching notes (continuity block is per-job dynamic, NOT in the cached prefix)
- Memory record `feedback_pipeline-integration.md` rule #1: trace every new field end-to-end before committing — explicitly applies to `series_id` threading
