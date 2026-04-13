# Master State Machine — Narrative vs. Mastery

**Status**: Architecture proposal
**Owner**: Voice + Backend + Frontend
**Depends on**: voice-mastery-loop-gold-standard (kitesforu-docs), kitesforu-api #228 (Teach + Practice)
**Date**: 2026-04-12

---

## The core idea

Every competitor in interview prep shows the user a button that says "Practice" or "Interview." The user clicks it. The system runs the matching mode. Dumb.

Our differentiator: **the user never clicks a mode button.** The narrative label they see ("Teach", "Practice", "Stress-Test") is decorative — a lagging indicator of a hidden three-variable state machine the system drives continuously. Pressure rises, support drops, interviewer warmth cools — and by turn 8 the user is in a full cold-direct stress interview while the screen still says "Practice". They never felt a modality shift, because from their perspective there wasn't one. There was only the system getting harder as they got better.

The narrative labels are theater. The hidden three-tuple is the product.

---

## The two parallel state systems

### Narrative State (user-facing)

```
'teach' | 'practice' | 'stress_test' | 'review'
```

Visible label in the phase ribbon. Rarely changes (0–2 transitions per session). Transitions are **hysteresis-guarded**: a region boundary must be crossed for **2 consecutive turns** before the label flips. This kills oscillation across 0.34 ↔ 0.36 boundaries and keeps the label feeling stable.

### Mastery State (hidden, logic-driving)

Three continuous variables, each `∈ [0,1]`:

| Variable | Meaning |
|---|---|
| **`pressure`** | Environmental demand — interviewer tone, question difficulty, time budget, follow-up intensity |
| **`support`** | Scaffolding level — hints available, STAR prompts visible, feedback density, retry allowance |
| **`mastery`** | Earned skill — EMA of rubric average, STAR completion rate, recovery-from-setback signal |

These drift continuously. Every turn updates all three.

---

## Per-turn update formulas

Clock: **turn-count primary**, real-time secondary (fatigue decay only). One "turn" = one user answer + 5-dim judge score.

After each rubric score `r ∈ [0,5]`, normalized `rn = r/5`:

```
Δmastery   = 0.15 * (rn - mastery) + 0.05 * star_completion_bonus
Δpressure  = 0.08 * (mastery - pressure) + 0.04 * streak_signal
Δsupport   = -0.06 * mastery + 0.03 * struggle_signal
```

Where:
- `streak_signal`: `+1` if last 3 turns avg `rn > 0.75`, `-1` if `< 0.45`, else `0`
- `struggle_signal`: `+1` if last 2 turns `rn < 0.5` or recovery-from-setback detected
- EMA smoothing on mastery (α=0.3) prevents single-turn collapse
- **Whiplash guard**: `|Δmastery| ≤ 0.12` per turn, hard clamped

Rubric-triggered, not time-triggered. **No turn count alone advances difficulty.** Pressure only rises when mastery has been earned.

---

## Narrative-to-mastery decision table

```
Region                                   | Narrative Label
-----------------------------------------|------------------
mastery < 0.35                           | "Teach"
0.35 ≤ mastery < 0.55, pressure < 0.5    | "Practice"
0.55 ≤ mastery < 0.75, pressure ≥ 0.5    | "Practice" (intensified)
mastery ≥ 0.75  OR  pressure ≥ 0.7       | "Stress-Test"
session.endRequested = true              | "Review"
```

**Hysteresis**: label flips only after **2 consecutive turns** in the new region. Track `pendingLabel + pendingTurns` counter. Any regression resets the counter.

---

## Seamless modality shift — 5 worker knobs

Every knob is a **pure function of `(pressure, support, mastery)`**. No discrete phase flag anywhere in the worker code.

| Knob | Driven by | Formula |
|---|---|---|
| **`questionTone`** | `pressure` | `<0.4`: `warm_curious` · `0.4–0.7`: `neutral_professional` · `>0.7`: `cold_direct` |
| **`followUpPersistence`** | `pressure × (1 − support)` | `0–1.0` float → drives `max_followups` (1→4) and "I need a number" probe likelihood |
| **`hintInjection`** | `support` | `>0.6`: full STAR scaffolding · `0.3–0.6`: gentle nudge · `<0.3`: zero hints |
| **`personaWarmth`** | `1 − pressure` | TTS prosody: `style ∈ [0.10, 0.40]`, `rate 0.95–1.05`, pause profile `conversational ↔ rapid` |
| **`latencyBudgetMs`** | `pressure` | Max think-time before interviewer probes silence: `<0.4`: `12000` · `>0.7`: `4000` |

**UI-side knobs** (also pure functions of hidden state, bound via Zustand selectors):
- `starShadow.opacity = clamp(0.3, 0.8, 0.3 + support * 0.6)`
- `roomAtmosphere.lightingTemp = lerp(warm, cool, pressure)`
- `coachingTarget.railOpacity` follows `support`, **not** narrative label

**Result**: at turn 8 a user is on `pressure=0.72, support=0.25`, hearing cold-direct questions with 4s probes and a nearly-invisible STAR rail, while the screen still says "Practice". The shift is *felt*, not announced.

---

## TypeScript schemas

```ts
// hooks/voice/slices/masterySlice.ts
export type NarrativeLabel = 'teach' | 'practice' | 'stress_test' | 'review'

export interface HiddenState {
  pressure: number    // [0,1]
  support: number     // [0,1]
  mastery: number     // [0,1]
  turnCount: number
  pendingLabel: NarrativeLabel | null
  pendingTurns: number
}

export interface RubricScore {
  specificity: number
  quantified_impact: number
  role_outcome_clarity: number
  ownership: number
  level_appropriate: number
  star_completion: boolean
}

export interface WorkerKnobs {
  questionTone: 'warm_curious' | 'neutral_professional' | 'cold_direct'
  followUpPersistence: number
  hintInjection: 'full_star' | 'nudge' | 'none'
  personaWarmth: { style: number; rate: number; pauseProfile: string }
  latencyBudgetMs: number
}

export interface TurnUpdateEvent {
  turnId: string
  rubric: RubricScore
  rubricMean: number
  prevState: HiddenState
  nextState: HiddenState
  deltaApplied: { dPressure: number; dSupport: number; dMastery: number }
  workerKnobs: WorkerKnobs
  narrativeLabel: NarrativeLabel
  labelChanged: boolean
  safeguardsTriggered: string[]
  timestamp: number
}

export interface MasterySlice {
  hidden: HiddenState
  label: NarrativeLabel
  knobs: WorkerKnobs
  lastUpdate: TurnUpdateEvent | null
  applyTurn: (rubric: RubricScore, sessionNumber: number) => TurnUpdateEvent
  resetSession: (sessionNumber: number, seed?: Partial<HiddenState>) => void
}
```

`applyTurn` is **pure** (no async, no side effects). The `TurnUpdateEvent` it returns is then pushed to the worker via the existing LiveKit data channel.

---

## Firestore telemetry

`sessions/{id}/masteryTrace/{turnId}`:

```python
class MasteryTraceDoc(BaseModel):
    model_config = ConfigDict(extra='ignore')
    turn_id: str
    turn_index: int
    timestamp: datetime
    rubric: RubricScoreModel
    rubric_mean: float
    prev: HiddenStateModel
    next: HiddenStateModel
    deltas: DeltasModel
    knobs: WorkerKnobsModel
    narrative_label: str
    label_changed: bool
    safeguards_triggered: list[str]
    user_session_number: int
```

Aggregate written to `sessions/{id}.masterySummary` on session end:
`{ maxPressure, minSupport, finalMastery, labelTransitions, avgRubric }`.

This powers a debug page that answers the only question that matters: **"did the engine make sensible decisions?"**

---

## Safeguards (hard-enforced)

```python
def apply_safeguards(state: HiddenState, session_num: int) -> HiddenState:
    p_ceiling = 0.65 if session_num <= 3 else 0.85
    state.pressure = min(state.pressure, p_ceiling)
    state.support  = max(state.support, 0.20)
    state.mastery  = clamp(state.mastery, 0.0, 1.0)
    return state
```

- **Mastery decay cap**: `Δmastery ≥ −0.08` per turn. One bad answer cannot demolish earned mastery.
- **Confidence-destroyer guard**: if 3 consecutive turns `rn < 0.4` AND `pressure > 0.6`, force `support += 0.15`, `pressure −= 0.10`. The worker drops back to scaffolding without saying so.
- **First-session cold start**: `pressure=0.25, support=0.7, mastery=0.3` regardless of any signal.
- **Anti-trap**: if a single dimension (e.g. `quantified_impact`) drags `rubric_mean` for 3+ turns, route to the `coachingTarget` chip rail rather than inflating pressure. Don't punish a single-knob weakness with global difficulty.

---

## Unit test invariants

11 invariants the slice must satisfy. Test file: `hooks/voice/slices/__tests__/masterySlice.test.ts`.

1. After 5 consecutive turns `rubric_mean ≥ 4.0/5`, pressure increased by `≥ 0.10`.
2. After 1 turn `rubric_mean = 0`, mastery decreased by `≤ 0.08` (decay cap).
3. Support never drops below `0.20` across any synthetic 50-turn sequence.
4. In sessions 1–3, pressure never exceeds `0.65` regardless of inputs.
5. Narrative label never flips on a single boundary-crossing turn (hysteresis).
6. After 3 consecutive turns `rubric_mean < 0.4` with `pressure > 0.6`, next turn's support increased by `≥ 0.10`.
7. `workerKnobs.questionTone` is **monotonic in pressure** — no tone regression while pressure rises.
8. `starShadow.opacity` is a pure function of `support` (deterministic, no hidden coupling to label).
9. Replaying the same `TurnUpdateEvent` stream from `MasteryTraceDoc` reproduces the identical final `HiddenState` (determinism).
10. Across 1000 random rubric sequences, `|Δmastery| ≤ 0.12` holds for every turn.
11. Label transitions per session `≤ 3` on a 20-turn synthetic ramp from `rn=0.2 → rn=0.9`.

---

## Integration points

### Frontend (`kitesforu-frontend`)
- **New slice**: `hooks/voice/slices/masterySlice.ts` — composed as the 6th slice into `useVoiceStore`
- **Existing slices subscribe** to `hidden` via selectors, not `label`. The phase ribbon is the **only** thing that reads `label` directly.
- **`StarShadow`, `CarryForwardChip`, `RoomSurface`** all derive their visual state from `hidden` — so the narrative-vs-mastery separation is enforced by React itself.
- **`MockVoiceController`** drives `applyTurn` on a scripted rubric stream to prove the separation visually in the preview harness.

### API (`kitesforu-api`)
- **New route**: `POST /v1/interview-prep/mock/sessions/{id}/turns/{turn_id}/mastery` — persists `TurnUpdateEvent` as a `MasteryTraceDoc`.
- **Existing Elo adapter** consumes `next.mastery` instead of raw rubric weighted score (smoother Elo trajectory).
- **`SessionConfig`** extended with `hidden_state_seed: HiddenState`:
  - `PRACTICE_CONFIG`: `(pressure=0.25, support=0.7, mastery=0.3)`
  - `INTERVIEW_CONFIG`: `(pressure=0.55, support=0.4, mastery=0.5)`
- The state machine itself is **identical across configs** — only seeds differ.

### Workers (`kitesforu-workers`)
- **Interview prep stage reads `WorkerKnobs`** from each `TurnUpdateEvent` and threads them into:
  - LLM question-generation prompt (system message + tone directive)
  - ElevenLabs `voice_settings` (style, rate, pause profile)
  - Follow-up controller (`max_followups`, probe likelihood)
  - VAD silence timeout (`latencyBudgetMs`)
- **Critical**: per the pipeline-integration rule "trace every new data field end-to-end before committing" — verify the knobs actually reach the TTS API call and the LLM system prompt. The most common failure mode in this product has been "selected but never used."

---

## Build order

Ship vertically, not horizontally. One full loop beats five partial pieces.

1. **`masterySlice` + 11 unit tests** — pure logic, zero deps, verifiable in isolation
2. **Wire into `useVoiceStore` + extend `MockVoiceController`** — preview harness drives scripted rubric stream, visual state changes WITHOUT narrative label changing
3. **Debug overlay page** — small route showing the `hidden` tuple + `knobs` live, proves the engine is making sense
4. **API persistence** — `MasteryTraceDoc` + the new POST route
5. **Worker wiring** — plumbs `WorkerKnobs` into the real LiveKit agent (follow-up PR, Phase 2)

---

## Anti-patterns (enforced)

- **No discrete phase flags in worker code.** Every worker knob is a function of `hidden`, nothing else.
- **No narrative label coupling.** The label drives UI only if it drives nothing else. If a worker knob depends on `label`, that's a bug.
- **No single-turn label flips.** Hysteresis is non-optional.
- **No arbitrary difficulty ramps.** Pressure only rises when mastery earns it. The engine is not a treadmill.
- **No hidden state that contradicts visible state.** If `pressure > 0.7` the room must feel cool; if `support > 0.6` hints must actually appear. User perception is ground truth.
- **No confidence-destroyer sequences.** The safeguard guard fires automatically; never hand-override.
