# R2 — Scenario Guidance: Live Input Coverage

**Status**: SHIPPED (moved from proposed/ to done/)
**Priority**: P1 (user-reported; identified in standing deferred follow-up)
**Effort**: 1 week estimated; actual ~1 session, 2 frontend PRs
**Affected repos**: kitesforu-frontend (primary), kitesforu-docs (this)
**Builds on**: `done/R2-scenario-type-guidance.md` + `done/R2-scenario-guidance-sub-genre-calibration.md` (29 typed sub-genres + typewriter + tick infrastructure)

---

## Implementation summary

Two-PR frontend chain closed all four code deliverables:

| PR | Scope |
|---|---|
| `frontend#535` | **D1 + D2** — chip row stays visible while the user types (gate changed from `promptText.length === 0` to `scenario != null`); new pure helper `computeHintCoverageFromInput(entry, userInput)` drives ticks from user input (tokenize label + gloss, intersect with input tokens, named-anchor short-circuit). 8 new unit tests. |
| `frontend#536` | **D3 + D4** — `coverageMeterCopy(covered, total)` helper + plain-English progress line below the chip row (`"3 of 5 · Strong input — 2 more chips..."`); `trackScenarioCoverageAtSubmit` telemetry fires once per real submission, PII-free. 8 new unit tests. |

**53 total tests** now green in `__tests__/lib/scenario-guidance.test.ts` (37 pre-existing + 16 new across the two PRs).

### Closes end-to-end

- ✅ D1: Chip row stays visible after keystroke 1
- ✅ D2: Input-driven coverage helper (heuristic, < 1ms/keystroke, zero LLM cost)
- ✅ D3: Plain-English "3 of 5 — strong input" progress signal
- ✅ D4: `scenario_coverage_at_submit` telemetry in Cloud Logging
- 🕒 Phase 5: Threshold calibration from live usage data — **future post-launch tune**, no code action until baseline data lands

### User-visible result

**Before**: chip row vanished at keystroke 1. Teaching signal died at the exact moment a typing user could have benefited from it.

**After**: chips stay visible, tick live as the user types matching vocabulary (e.g. typing "Priya, 4" ticks the "child name + age" chip for the bedtime sub-genre), and a progress line below translates the tick count into plain English. Users see in real time whether their input is dense enough to unlock the planner's strong-output path, and the submit telemetry lets us measure the effect across the cohort.

---

## Why This Is A Follow-Up

User's standing deferred ask, verbatim:

> "Improve the scenario/type guidance UI so it feels more like active typing, uses a bigger box, and gives much more specific example input. The objective is to **teach the user exactly what highly specific data they can provide so the system can produce a much stronger output and avoid follow-up questions**. Do deeper research per scenario/type in the system before deciding what guidance/examples to show."

The R2-scenario-type-guidance + sub-genre-calibration proposals shipped **input exemplars** (what great input looks like) but the **teaching feedback loop** is one-directional: the typewriter demonstrates, then the chip row disappears as soon as the user starts typing.

The missed opportunity: once the user is typing their own input, there's no real-time signal about whether they're hitting the same density as the exemplar. They type, submit, and only learn at follow-up-question time that their input was sparse. By then the planner has already committed to a weaker path.

This proposal closes that loop: **chips stay visible while typing, and each chip's tick is driven by the user's own input rather than the typewriter's position**.

---

## Problem

In `components/smart-create/IntentSection.tsx` (line ~642), the chip row is gated on `promptText.length === 0`:

```tsx
{scenario && promptText.length === 0 ? (
  <div className="space-y-2">
    {/* chip row with typewriter-driven ticks */}
  </div>
) : null}
```

Once the user types even one character, the guidance UI vanishes. The typewriter-driven ticks (PR #500) only demonstrate what *coverage* looks like on the exemplar — they never grade the user's input.

Verified behaviors that are load-bearing and must be preserved:
- Typewriter exemplar animation into the placeholder (existing)
- Sub-genre chip row above textarea (existing)
- Chip click expands a hint gloss inline (existing)
- `coverageOffsets` parallel array on each `ScenarioGuidance` entry (existing)
- `bigBox` flag grows textarea for dense scenarios (existing)

---

## Deliverable 1: Always-Visible Chip Row

**Problem**: Chip row disappears at `promptText.length > 0`, cutting off the teaching signal at the exact moment the user could benefit from it most.

**Fix**: Render the chip row for every state where a `scenario` is active. Change the gate from `promptText.length === 0` to `scenario != null`.

**Visual contract during typing**:
- Empty input state: ticks track the typewriter (existing behavior)
- Typing state: ticks track the user's input via the new helper (new behavior)
- Completed coverage: all 5 hints ticked → chip row can optionally fade to 60% opacity or show a subtle "Looking great — ready to submit" hint

### Acceptance Criteria
- [ ] Chip row visible for every non-null scenario regardless of `promptText.length`
- [ ] Hint gloss expansion on click still works while typing
- [ ] No layout shift between empty and typing states (pre-reserve row height)
- [ ] Dark-mode parity maintained

---

## Deliverable 2: Input-Driven Coverage Helper

**New pure helper** in `lib/scenario-guidance.ts`:

```ts
export function computeHintCoverageFromInput(
  entry: ScenarioGuidance,
  userInput: string,
): Set<number>
```

**Matching strategy** (lightweight, zero-LLM, <1ms per keystroke):

1. For each `schemaHint[i]`, collect a candidate keyword list from:
   - The hint label itself, tokenized + stopword-filtered (e.g. "child name + age (3-5 / 5-8)" → `[child, name, age]`)
   - The hint gloss in `hintGlosses[hint]`, same tokenization (e.g. for bedtime: `[name, personalization, age, vocabulary, image]` plus any named anchors like "Priya", "Aanya")
2. Normalize user input: lowercase + strip punctuation + split on whitespace → token set
3. Hint i is "covered" when the user-input token set intersects the hint's candidate keyword list by ≥ 2 tokens (or ≥ 1 if the hint has < 3 candidate keywords).
4. Named anchors (capitalized words in the gloss) count for 2 — e.g. typing "Priya" alone covers the "child name + age" hint because "Priya" is a strong anchor.

**Why ≥ 2 tokens**: a single accidental word shouldn't falsely tick the hint. But named-anchors are rare enough to count double.

**Scope note**: this is a BEST-EFFORT heuristic. Occasional false-positives are fine — the chip turning green when the user typed the word "name" means the hint is "probably addressed", not "perfectly covered". The alternative (LLM evaluation per keystroke) would cost ~$0.002 × N keystrokes and add latency.

### Acceptance Criteria
- [ ] `computeHintCoverageFromInput(entry, '')` returns empty Set
- [ ] Typing "Priya, 4" ticks the "child name + age" hint for the bedtime sub-genre
- [ ] Typing "gore" ticks the "one BIG idea" hint for sci-fi (tokenization finds "gore" as a noun, not actually the right hint — edge case worth testing)
- [ ] Unit tests for every populated sub-genre: pick 3 sample inputs per sub-genre and pin the expected tick set
- [ ] Pure function — no React, no DOM, testable in isolation

---

## Deliverable 3: Typewriter vs Input Tick Routing

**Design**: the `IntentSection` decides which coverage function to call based on `promptText.length`:

```tsx
const tickedIndexes = promptText.length === 0
  ? computeTickedHintIndexes(scenario, coachedPlaceholder.length) // existing
  : computeHintCoverageFromInput(scenario, promptText) // new
```

**Rationale**: the two paths serve different purposes:
- Empty state: "let me show you what great looks like" (typewriter demonstrates)
- Typing state: "let me tell you how you're doing" (input grades itself)

Switching at `promptText.length === 0` is a single-expression change; behavior for every existing test case stays identical.

### Acceptance Criteria
- [ ] Empty state → typewriter-driven ticks (identical to current behavior)
- [ ] Typing state → input-driven ticks
- [ ] Switching between empty ↔ typing smoothly (no tick flicker)

---

## Deliverable 4: "Coverage Meter" Header

Above the chip row, render a small header that translates the tick count into a plain-English progress signal:

```
"Suggested: include a line or two about"  [existing header]
────────────────────────────────────
 [3 of 5] ● ● ● ○ ○              [new progress bar]
```

**Plain-English copy** by tick count:

| Covered | Copy |
|---|---|
| 0/5 | "Your input could be more specific — tap a chip to see examples" |
| 1–2 | "Getting there — 2 more chips to cover" |
| 3 | "Strong input — 2 more chips to unlock the best output" |
| 4 | "Almost perfect — one more chip" |
| 5 | "Looking great — ready to submit" |

This gives the user a **scalar signal** on top of the binary per-chip ticks, so they know how close they are without having to count chips themselves.

### Acceptance Criteria
- [ ] Progress bar visible only when a scenario is active
- [ ] Copy updates live as user types
- [ ] Non-blocking: user can submit with 0 ticks; this is a coaching layer, not a gate
- [ ] Tracked via analytics (`scenario_coverage_at_submit: 0..5`) to validate the feature's value over time

---

## Out of Scope

- **LLM-based coverage evaluation** — too expensive per keystroke; the heuristic is good enough for a teaching layer. Future upgrade candidate only if usage data shows heuristic false-positive rate > 15%.
- **Sub-genre auto-classification from typing** — explicitly deferred in the sub-genre-calibration proposal (`done/R2-scenario-guidance-sub-genre-calibration.md`); manual chip selection has proven to work. Unblocked by this PR, not addressed here.
- **Real-time LLM rewrite suggestions** — would be the next level ("here's how to make this input stronger"). Cost and latency prohibitive; separate future proposal.
- **Per-scenario schema field for explicit keywords** — tempting to add `coverageKeywords: string[][]` to `ScenarioGuidance` alongside `coverageOffsets`, but the tokenize-label+gloss approach avoids 29 × ~5 = ~145 hand-curated lists. If the heuristic proves insufficient, a follow-up can add the explicit field.

---

## Research Direction (Before Implementation)

The user's standing ask says: *"Do deeper research per scenario/type in the system before deciding what guidance/examples to show."*

The existing sub-genre calibration already ran 29 deep-research agents (one per populated sub-genre) for the **exemplars**. This proposal does NOT require the same depth — we're reusing those exemplars + glosses, not creating new ones. The research layer here is:

1. **Validate token extraction** on the 29 existing hints: run the tokenizer across every `schemaHint` and `hintGlosses` entry, manually verify the keyword lists are sensible for 3 representative sub-genres (bedtime, business-deep-dive, medical-step1).
2. **Calibrate the coverage threshold** (≥ 2 tokens vs ≥ 3): run a handful of synthetic inputs against every sub-genre, measure false-positive + false-negative rates, pick the threshold that minimizes false-positives while keeping named-anchor coverage.
3. **Test on user-submitted inputs** captured in analytics since the sub-genre-calibration ship (`trackScenarioHintClicked` emissions indicate engagement; we can correlate with downstream content quality).

No new deep-research agents needed. The research is about the _coverage heuristic_, not the _examples_.

---

## Effort & Sequence

| Phase | Work | Effort |
|---|---|---|
| 1 | Build + unit-test `computeHintCoverageFromInput` against 3 sub-genres | ~0.5 day |
| 2 | Wire it into `IntentSection` (chip row always-visible + route ticks) | ~0.5 day |
| 3 | Coverage meter header + plain-English copy | ~0.5 day |
| 4 | Analytics emit (`scenario_coverage_at_submit`) + full sub-genre test coverage | ~1 day |
| 5 | Calibrate threshold from staged inputs + iterate | ~2 days |

**Total**: ~1 week, single frontend PR chain (2-3 PRs depending on how Phase 3 + Phase 5 split).

## Success Metrics

- **Primary**: Average `scenario_coverage_at_submit` rises from baseline (currently untracked) to ≥ 3 of 5 within 2 weeks of ship.
- **Secondary**: Follow-up question rate on first-submission drops measurably (correlation, not causation — report alongside, not as the metric itself).
- **Guardrail**: False-positive rate on the heuristic < 15% (user-covered hints that didn't actually lead to better output).
