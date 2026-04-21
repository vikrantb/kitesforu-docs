# R2 Phase 1 — Content Quality: Voice & Audio

**Status**: PROPOSED
**Priority**: P1
**Effort**: 3 weeks
**Affected repos**: kitesforu-workers, kitesforu-frontend
**Depends on**: R1 complete (UX foundation shipped)
**Absorbs**: P2-voice-architecture-consolidation.md (existing proposal)

---

## Why This Is the First Content Release

The UX redesign (R1) fixes navigation and discovery. R2 fixes the product itself — the audio. For entertainment personas (Horror, Comedy, Romance, Bedtime), the voice IS the product. For learning personas (Exam Crammer, ADHD Learner), content structure IS the product. No amount of beautiful UI compensates for mediocre audio.

---

## Deliverable 1: Voice Architecture Consolidation

**Absorbs**: P2-voice-architecture-consolidation.md

**Problem**: Two divergent voice stacks exist:
1. Car Mode uses `useCarVoiceOrchestrator` — real SpeechRecognition, SpeechSynthesis
2. Voice-First Simulation uses `MockVoiceController` — setTimeout theater

**Fix**:
1. Extract `useVoiceOrchestrator` from Car Mode as shared hook
2. Rewrite Voice-First Simulation against the shared orchestrator
3. Add barge-in handling
4. Wire halo state to real audio events

### Acceptance Criteria
- [ ] One voice orchestrator serves Car Mode AND Voice Simulation
- [ ] ListeningHalo reflects real audio state
- [ ] Barge-in works (user can interrupt)
- [ ] Car Mode regression test passes
- [ ] Voice Simulation regression test passes

---

## Deliverable 2: Series Memory (Full Wiring)

**Problem**: Multiple personas want serialized content (Horror worldbuilder, Romance listener, Bedtime Parent). Series memory exists in the workers but is not fully wired into the streaming pipeline.

**Fix**:
1. Wire `common/series_memory.py` into `streaming_script_audio_worker.py`
2. When generating episode N of a series, pass episodes 1..N-1 context to the script prompt
3. Store series state in Firestore: character names, plot points, world rules, emotional arcs
4. Frontend: "Continue series" button on the last episode that generates the next one

### Data Model

```python
# Firestore: series/{series_id}
{
  "course_id": "...",
  "episodes_generated": 5,
  "characters": [
    {"name": "Mara", "role": "protagonist", "last_state": "discovered the map"},
    {"name": "The Keeper", "role": "antagonist", "last_state": "watching from the tower"}
  ],
  "world_rules": ["Magic costs memory", "The forest shifts at midnight"],
  "plot_threads": [
    {"thread": "The missing lighthouse keeper", "status": "unresolved"},
    {"thread": "Mara's lost brother", "status": "hinted at in ep 3"}
  ]
}
```

### Acceptance Criteria
- [ ] Episode N references characters and events from episodes 1..N-1
- [ ] Character names are consistent across episodes
- [ ] Plot threads carry forward (not forgotten or contradicted)
- [ ] Frontend shows "Continue series" button
- [ ] Bedtime Parent: "Continue the Mermaid Zara adventures" works
- [ ] Horror Fan: recurring universe across episodes maintains lore

---

## Deliverable 3: Content Rating & Safety System

**Problem**: Bedtime parents need safety guarantees. Romance listeners want intensity control. Horror fans want it dark. No content rating system exists.

### Rating Levels

| Level | Label | What It Means |
|-------|-------|---------------|
| G | Family | No violence, no mature themes, gentle language |
| PG | General | Mild conflict, no explicit content |
| T | Teen | Moderate intensity, mild violence, some mature themes |
| M | Mature | Strong themes, violence, intense emotional content |
| E | Explicit | Explicit content (opt-in only) |

### Implementation

1. **At creation time**: Smart-create sets `content_rating` based on template + topic:
   - Bedtime stories: always G
   - Educational: PG
   - Horror: T or M (based on subgenre)
   - Romance: PG to E (user-selected "heat level")
   
2. **In generation**: Rating flows into the system prompt as a constraint:
   ```
   Content rating: G (Family). Do NOT include: violence, death, scary imagery,
   complex emotional situations. Language must be gentle and age-appropriate.
   ```

3. **In the library**: Rating badge on cards (small, unobtrusive)

4. **Parental controls**: Optional — parent can set max rating in account settings. Content above the threshold is hidden.

### Acceptance Criteria
- [x] Every new **course** gets a `content_rating` at creation time — schemas PR #73 (v1.53.0) added `ContentRating` enum (G/PG/PG_13/R) + `Course.content_rating` field; api PR #268 wired a deterministic classifier (`services/content_rating.py`) into `create_course`. Covers `course` kind only — classes / writeups / podcasts still pending; follow-up PRs share the same classifier.
- [x] Rating flows into system prompts as a content constraint — full chain shipped (schemas #74 → api #269 → course-workers #58 → workers #302). PromptComposer emits per-tier constraint block at priority 105 (immediately after SystemIdentityBuilder) so the LLM sees the rating before any genre / language rules layer on. Silent-when-null for legacy / single-podcast flows.
- [x] Bedtime stories are always rated G (enforced, not suggested) — end-to-end enforcement shipped: api #268 classifier **hard-floors BEDTIME style to G**; workers #302 `ContentRatingBuilder` hard-collapses any non-G rating to G when `content_type == bedtime` (the worker refuses to lie to the badge). Parental-trust enforcement block emitted on every bedtime prompt. Hard output-validation refusal is an optional post-gen follow-up; prompt-level enforcement is the load-bearing safety mechanism and is shipped.
- [x] Rating badge visible on library cards — frontend PR #529: `lib/content-rating.ts` helper + `AudioContentCard` renders an emerald/amber/orange/rose pill with dark-mode variants. Silent-when-null so legacy courses render unchanged.
- [x] Parental controls option in account settings — schemas PR #75 (v1.55.0) added `max_content_rating` to `UserProfile` / `UpdateUserProfileRequest` / `UserProfileResponse`; api PR #270 bumped the dep (route layer auto-forwards the field via generic `_to_response`); frontend PR #530 shipped the dropdown in `/settings/profile` (No filter / G only / Up to PG / Up to PG-13 / Up to R) plus the client-side library filter in `app/library/page.tsx` using `isAllowedByMaxRating(itemRating, maxRating)` helper. Fail-open on unknown / legacy ratings by design so pre-classifier content never silently vanishes behind a filter the user applies later.

**D3 STATUS — ALL 5 ACs SHIPPED.** 10-PR chain complete across 5 repos. See `done/R2-p1-D3-content-rating.md` for the full implementation summary once the rest of R2-p1 ships.

### Naming note

Shipped with MPAA-style tiers (**G / PG / PG_13 / R**) instead of the proposal's original ESRB-style (G / PG / T / M / E). MPAA is the more legible taxonomy for a general audience and maps cleanly to iTunes `explicit` for RSS. The `E`-tier (explicit, opt-in) is absorbed into `R` with parental-controls handling the gating. NC-17 is intentionally rejected at the Pydantic boundary.

---

## Deliverable 4: Genre-Specific Quality Improvements

### Comedy Timing

- Punchline delivery rate: 0.92 (verified in production)
- Add: topicality detection — if user asks about current events, trigger research for fresh material
- Add: callback structure — jokes reference earlier setups, not just sequential punchlines
- Metric: listen-through rate on comedy episodes > 90%

### Horror Atmosphere

- 8-phase tension arc already in place
- Add: ambient sound layer selection based on subgenre (cosmic = drones, gothic = rain/wind, psychological = silence)
- Add: voice persona auto-selection for horror (Vincent Graves default, user can override)
- Metric: share rate on horror episodes (the "you HAVE to listen to this" signal)

### Romance Chemistry

- Add: chemistry-building dialogue rules — subtext, unspoken tension, what characters DON'T say
- Add: POV switching — "hear this scene from his perspective"
- Add: slow burn pacing control — user selects "slow burn" / "fast pace" / "let the AI decide"
- Metric: series continuation rate > 80% (they want the next episode)

### Bedtime Wind-Down

- Add: energy arc enforcement — last 30% of story must decrease in pace, volume, and excitement
- Add: "sleep-inducing" voice settings — slower rate, lower pitch, softer delivery
- Add: no cliffhangers for bedtime stories (user preference)
- Metric: parent-reported "child fell asleep during story" rate

### Acceptance Criteria
- [ ] Comedy: topicality detection works for current-event humor
- [ ] Horror: ambient sound layer matches subgenre
- [ ] Romance: POV switching generates alternate-perspective episodes
- [x] Bedtime: energy arc enforcement prevents exciting endings — shipped end-to-end via a 3-PR chain. workers #303 added `_check_bedtime_energy_arc` to the quality gate with an `_EMOTION_ENERGY_SCORE` map + `_BEDTIME_FORBIDDEN_TAIL_EMOTIONS` set (excited / tense / dramatic / urgent forbidden in the final 30%); workers #304 wired `CRITICAL_BEDTIME_FORBIDDEN_IN_TAIL_COUNT = 1` into `regen_policy.py` so any tail-emotion leak auto-triggers script regeneration with a targeted hint ("Bedtime requires monotonically decreasing energy; rewrite the last third calmer"); workers #305 fixed the load-bearing dead-code bug where the `_derive_types` call site read `AudioConfig.content_type` (a `ContentType` enum with no `bedtime` value) so the gate path never executed in production — replaced with `_derive_types_with_profile_fallback` that promotes umbrella enum values (educational / storytelling / meditation / etc.) to the profile-level genre when `workers.prompts.get_content_type_context` detects `bedtime` / `horror` / `comedy` / `romance` / `drama`. 16 new unit tests pin the narrow-override semantics. Chain is now live on main.
- [ ] Each genre's quality metric improves measurably

---

## Testing Plan

### Automated
- [ ] Unit tests for series memory Firestore read/write
- [ ] Unit tests for content rating assignment
- [ ] Integration test: generate episode 2 of a series, verify character names match episode 1

### Manual Quality Checks
- [ ] Generate a 3-episode horror series — verify atmosphere consistency
- [ ] Generate a bedtime story — verify it's genuinely calming at the end
- [ ] Generate a comedy episode about today's news — verify topicality
- [ ] Generate a romance episode — verify character chemistry in dialogue
- [ ] Generate a G-rated bedtime story with "dragon" in the prompt — verify no scary content
