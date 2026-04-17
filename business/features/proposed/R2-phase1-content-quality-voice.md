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
- [ ] Every new content item gets a `content_rating` at creation time
- [ ] Rating flows into system prompts as a content constraint
- [ ] Bedtime stories are always rated G (enforced, not suggested)
- [ ] Rating badge visible on library cards
- [ ] Parental controls option in account settings

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
- [ ] Bedtime: energy arc enforcement prevents exciting endings
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
