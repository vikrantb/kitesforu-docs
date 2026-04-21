# R2 — Scenario Guidance v2: Density, Active Typing, and Clarifier Preemption

**Status**: PROPOSED
**Priority**: P1
**Effort**: 1 week (9 deep-research agents in parallel + ~2 days of UI work + 1 day QA)
**Affected repos**: kitesforu-frontend
**Depends on**: R2-scenario-type-guidance (shipped 2026-04-20), R2-scenario-guidance-sub-genre-calibration (shipped 2026-04-20/21), R2-scenario-guidance-live-input-coverage (shipped 2026-04-21)
**Origin**: 2026-04-20 product-owner standing follow-up — *"improve the scenario/type guidance UI so it feels more like active typing, uses a bigger box, and gives much more specific example input … teach the user exactly what highly specific data they can provide so the system can produce a much stronger output and avoid follow-up questions."*

---

## 1. Problem — what the shipped v1 does and doesn't do

The existing scenario-guidance shipped across three PRs (R2-scenario-type-guidance for the 9 root scenarios, R2-scenario-guidance-sub-genre-calibration for the 29 sub-genre variants, R2-scenario-guidance-live-input-coverage for the live coverage ticks) covers roughly:

- **Coverage**: all 9 ScenarioId values + 29 of the 30 populated SubGenreId values in `lib/scenario-guidance.ts:142` (~2070 lines, ~29 SUB_GENRE_GUIDANCE entries).
- **UI surface**: `components/smart-create/IntentSection.tsx:156-269` renders a typedExemplar into the textarea's `placeholder`, animates char-by-char via `runTypewriter`, emits schemaHint chips below the textarea that flip to a ✓ as the user's live input covers each one, grows the textarea to ≥ 6 rows when `bigBox=true`.
- **Research methodology**: each existing entry cites a per-scenario deep-research-agent run with `researchedBy` / `researchedAt` / `version` audit fields.

### What's working

- Per-scenario exemplars are genuinely dense (300–900 chars), named anchors (Stripe L5 loop, MIT 6.006 Week 2, GDPR Art 17(3)(b), Shirley Jackson's *The Lottery*).
- Live coverage ticks give the user a tangible "filling out a form" progress signal.
- Sub-genre routing (via `getScenarioGuidance(scenarioId, subGenreId)`) picks the right exemplar when the user narrows from `storytelling` → `storytelling/romance`.

### What's not working — the three standing complaints from the product-owner review

| Complaint | Root cause |
| --- | --- |
| **"feels more like active typing"** | Once the typewriter animation finishes, the placeholder sits static. Users who land on the page *after* the animation has completed see a wall of grey text. The animation is a one-shot, not a live stream — it teaches the ideal, but doesn't keep the user in an active-authoring headspace. |
| **"uses a bigger box"** | `bigBox=true` grows to 6 rows on some scenarios. But **the actual ideal input is ~120-180 words** (the exemplar itself). 6 rows at typical line width is ~50-70 words before scroll, so a user who tries to match exemplar density has to scroll their own prose. The box should comfortably hold the exemplar. |
| **"gives much more specific example input"** and **"avoid follow-up questions"** | The exemplars are dense but were researched once, per scenario, with no explicit mapping to the clarifier questions the intake planner (`kitesforu-api/src/api/services/smart_create/intake.py:180-237`) actually asks. The ideal exemplar doesn't just sound smart — it **provably preempts every clarifier the planner would otherwise fire**. |

---

## 2. Goals / Non-goals

### Goals

1. Rewrite each scenario + sub-genre exemplar so it **covers every clarifier the planner has been observed asking** for that scenario. Measure coverage by sampling 2 weeks of `clarifier_questions` analytics (or by dry-running the intake LLM on each exemplar and asserting confidence ≥ 0.85 with `questions=[]`).
2. Grow the composer box to comfortably hold the exemplar without scroll on common viewport widths.
3. Make the "active typing" feeling durable — the typewriter should not terminate the affordance; there should always be *something* inviting the user to add detail.
4. Preserve v1's wins: schemaHint chips, coverage ticks, replay, per-entry research audit trail, feature-flag gating.

### Non-goals

- Not replacing the typewriter with a pure static example (that's a regression on "active typing").
- Not changing the ScenarioId / SubGenreId taxonomy — coverage at that axis is already good (29 of 30 sub-genres populated).
- Not pre-filling the user's actual prompt from the exemplar — the exemplar stays in `placeholder`, the prompt stays empty until the user types. Overwriting user intent was rejected in v1 and stays rejected.
- Not a new server-side service — guidance continues to live in `lib/scenario-guidance.ts` and iterate in git.

---

## 3. Proposed changes

### 3.1 Exemplar v2 — clarifier-preemption density

Each existing `ScenarioGuidance` entry gets rewritten by a dedicated deep-research-agent with a **new, harder contract**:

> Produce a `typedExemplar` such that, when submitted verbatim to the intake LLM (`kitesforu-api/src/api/services/smart_create/intake.py` system prompt + no template + no pasted content), the LLM returns `sufficient=true`, `confidence ≥ 0.85`, and `questions=[]`.

This is the single load-bearing bar. Every exemplar must explicitly answer the 5–8 data points the planner would otherwise ask about for that scenario:

- **All scenarios**: content_purpose, target audience, desired duration, language, tone/style, single named anchor.
- **interview-prep**: target company, target level/role, interview stages, candidate background (YoE + current role), weak spots, timeline.
- **skill-mastery**: current skill level (gap named), time budget (hours/week + deadline), format preference (drill / walkthrough / worked-example), concrete subtopic where the learner gets stuck.
- **exam-prep**: exam name + date, most recent score / section breakdown, study hours/week, drill format preference, specific weak pattern to train.
- **storytelling** (per sub-genre): genre/subgenre, tone reference (named author / show), episode count + length, POV, named setting, three named characters with one-line motivations, ending tone.
- **explainer-podcast**: episode length, audience sophistication, angle/contrarian take, one named news hook (< 30 days), two cited sources, two hosts' dynamic.
- **k12-lesson**: grade level, subject standard (NGSS / CCSS code), class size + demographics (ELL, IEP), lesson duration + segment breakdown, named misconception, formative check structure.
- **university-lecture**: course + week + lecture number, prerequisites from prior weeks, two board-work examples, common misconception, paired problem set context.
- **corporate-training**: regulation/framework + article, audience L-level, headcount, LMS target (SCORM / xAPI), mastery gate %, named anchor incident, cert deadline.
- **writeup**: word count, publisher/context (a16z brief / Stratechery essay / Mintlify docs), named anchor (case study or PR), output format (Markdown / HTML / plaintext), tone reference, one counter-argument steelmanned.

Each clarifier-preemption item is explicit in the agent prompt so the researcher can't return a vague-but-beautiful example.

### 3.2 Bigger box — comfortably hold the exemplar

- `bigBox` becomes a numeric `boxSizeRows` (or derived automatically from `typedExemplar.length`): exemplars ≤ 400 chars → 6 rows, 400–700 chars → 8 rows, 700–1000 chars → 10 rows, > 1000 chars → 12 rows.
- Tailwind: replace the fixed row count in `IntentSection.tsx` with a dynamic `minRows` prop and an `autosize` behavior that keeps pace with user input up to a cap (~16 rows) before internal scroll.
- `prefers-reduced-motion` + low-viewport clients get a hard cap at 8 rows to avoid hijacking the whole screen on mobile.

### 3.3 Active-typing affordance

Four small changes, measured by whether the placeholder *ever* feels "done":

1. **Looped idle shimmer**: after the typewriter finishes, the caret-like cursor (`▎`) keeps blinking at the end of the placeholder until the user starts typing.
2. **Gentle fade-to-next**: on scenarios with `alternates` (not yet populated for most scenarios — this PR also adds 2 alternates per scenario), the placeholder fades out at 10s idle and types the next alternate. So a user who reads scenario 1 for 12 seconds sees scenario 1 alternate start typing — the box feels alive.
3. **Schema-hint pulse**: the schemaHint chip nearest the caret's current char offset gets a subtle `animate-pulse` until the caret passes it. This makes the connection between "the exemplar is typing something" and "you should type something like this" literal.
4. **"Your turn"** microcopy once the first alternate completes — a single-line nudge rendered under the textarea, not inside it, so it doesn't pollute the placeholder.

### 3.4 `ScenarioGuidance` schema additions

```ts
export interface ScenarioGuidance {
  // …existing…
  /** Alternate exemplars the composer cycles through on idle. Each
   *  shares the same schemaHints + clarifier-preemption coverage as
   *  the primary typedExemplar, but demonstrates a different angle
   *  (e.g. Stripe vs Meta for interview-prep). Minimum 2. */
  alternateExemplars?: string[]
  /** Derived from the exemplar length; consumers use this instead of
   *  `bigBox`. Existing bigBox entries get mapped to 8 as a floor. */
  boxSizeRows?: number
  /** Machine-readable map of the clarifier keys each exemplar
   *  demonstrates. Used by a ship-time lint + a unit test that
   *  asserts at least N clarifiers are covered per scenario. */
  clarifiersCovered?: string[]
}
```

### 3.5 Research methodology — per-scenario deep-research agents

We run **9 root-scenario agents + 29 sub-genre agents in parallel**, each with:

- Access to `kitesforu-api/src/api/services/smart_create/intake.py` and the clarifier schema.
- Access to the existing v1 entry as a baseline (so they improve, not rewrite).
- A ship-time constraint: the rewritten exemplar MUST pass an automated dry-run of the intake LLM that asserts `sufficient=true`, `confidence >= 0.85`, `questions == []`.
- Output shape: `{ typedExemplar, alternateExemplars: [a, b], schemaHints, coverageOffsets, hintGlosses, clarifiersCovered }`.

Agents also cite real public anchors (scientific paper, news event, textbook chapter) for the anchor they choose. Citations land in the existing `citations` array.

---

## 4. Implementation plan

### Phase A — research (parallelizable)

1. Run the 9 root-scenario deep-research agents. Each agent returns one rewritten `ScenarioGuidance` entry conforming to §3.5.
2. Run the 29 sub-genre agents in parallel once the corresponding root-scenario agent has returned (so they can reuse the root's hint schema for consistency).
3. All agent outputs land in a staging array; no code ships until the automated dry-run gate passes.

### Phase B — automated gate

A new unit test in `__tests__/lib/scenario-guidance.v2.test.ts` mocks the intake LLM and asserts, for every entry in `SCENARIO_GUIDANCE` + `SUB_GENRE_GUIDANCE`:

- `typedExemplar.length >= 400`
- `alternateExemplars.length >= 2`
- `clarifiersCovered.length >= 5`
- No duplicate anchor across two alternates of the same scenario.
- `typedExemplar` is not a strict prefix of any alternate (they must demonstrate different angles, not extended versions of the same).

A second test — optional, run in CI with an `INTAKE_LLM_GATE=1` env flag — calls the real intake endpoint against each exemplar and asserts `sufficient=true, confidence >= 0.85, questions=[]`.

### Phase C — UI

1. `lib/scenario-guidance.ts` — add the v2 schema fields; backfill existing entries with `boxSizeRows` mapped from `bigBox`.
2. `components/smart-create/IntentSection.tsx` — replace the fixed row count with a dynamic `minRows`, wire the looped idle shimmer, wire the fade-to-next cycling.
3. Add "Your turn" nudge component + analytics event `scenario_guidance_your_turn_shown` for exposure measurement.
4. Add schema-hint chip pulse tied to the current typewriter offset.

### Phase D — flip-flag + rollout

- Gate behind `feature_scenario_guidance_v2` in `lib/feature-flags.ts`.
- Ship ON in dev, OFF in prod initially. Flip ON in a separate PR after the first 50 creations show no regression in `scenario_guidance_shown` → `content_created` funnel.

---

## 5. Acceptance criteria

- [ ] All 9 root-scenario `ScenarioGuidance` entries rewritten with ≥ 5 clarifiers covered and ≥ 2 alternateExemplars each.
- [ ] All 29 sub-genre entries rewritten under the same contract.
- [ ] Automated unit-test gate passes — length, alternate-count, clarifier-count, no-prefix-overlap.
- [ ] Optional real-LLM gate test (env-flagged) passes — every exemplar yields `sufficient=true, confidence ≥ 0.85, questions=[]` on the live intake endpoint.
- [ ] Composer box grows to fit the exemplar without internal scroll on ≥ 1280 px viewports.
- [ ] Placeholder feels "active" — cursor keeps blinking after typewriter completes; idle 10s triggers the next alternate to type in.
- [ ] Schema-hint chip nearest the caret pulses as the typewriter advances.
- [ ] "Your turn" nudge appears below the textarea on first completed alternate cycle.
- [ ] `prefers-reduced-motion` disables the looped shimmer and the fade-to-next (static state is still legible).
- [ ] Mobile layout caps box at 8 rows; alternates still cycle.
- [ ] `feature_scenario_guidance_v2` gates the whole surface; flag OFF falls back to the v1 behavior unchanged.
- [ ] Analytics event `scenario_guidance_alternate_shown` fires per cycle so we can measure *which* alternates land best.

---

## 6. Risk / rollback

- The real-LLM gate is expensive (~$0.003 × 38 exemplars × every PR). Keep it behind an env flag; run locally before merge, not on every push.
- The fade-to-next cycling could be distracting — if the `your_turn_shown` analytics exceeds 30% without a matching `content_created`, we've likely over-animated and should drop the cycling to a single alternate.
- A bigger box can feel overwhelming on mobile. The 8-row hard cap is the load-bearing guard; if `content_created` drops on mobile after rollout, pull back to 6 rows on mobile only.

Rollback is one flag flip (`feature_scenario_guidance_v2` → false) because the v1 code path stays intact throughout.

---

## 7. What NOT to do in this thread

- Don't re-do the taxonomy work — the ScenarioId / SubGenreId enum is good, and the sub-genre calibration PR already closed that loop.
- Don't add per-template exemplars (that's template-coaching, a different surface).
- Don't build a server-side clarifier-preemption service. The iteration budget for guidance copy is cheap git edits; any LLM call added here is paid on every Smart Create session, which is the wrong tradeoff.
- Don't introduce an A/B test framework just for this — existing analytics events plus a feature flag are sufficient for the rollout decision.
