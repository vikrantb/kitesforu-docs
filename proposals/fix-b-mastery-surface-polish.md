# Fix B — Mastery Surface Polish Bundle (S10 + S11 + S18)

**Status**: Draft — awaiting audit
**Date**: 2026-04-14
**Part of**: UI Excellence Sweep, Wave 1, Fix B (promoted from original Wave 2 Fix D)
**Maps to**: Issues S10, S11, S18 in `ui-excellence-sweep.md`
**Severity**: Important × 2 (S10, S11) + Quality × 1 (S18)
**Depends on**: nothing
**Blocks**: nothing

---

## Why this promoted from Wave 2 to Wave 1

The original Wave 1 plan was:
- Fix A = S5 creation prop mismatches
- Fix B = S1 voice replay card reachability
- Fix C = S4 + S12 signed-out home + voice-first hero copy

**Post-verification state of main HEAD (`a41d754`):**
- **S5 shipped** via PR #302 "upgrade classifyInput to real intent routing" — Fix A is a no-op.
- **S12 shipped** via your manual `a41d754` commit (Mastery regrouping + B2B hero copy). Fix C's hero half is a no-op.
- **S1 symptom is gone**: `MockVoiceController.tsx:194` now calls `setTurnPhase('between_turns')`, and the card surfaces in the mock tour. But the gate semantics are **structurally unchanged** — `OneFixReplayCard.tsx:37` still checks `haloState !== 'asking'` when the comment at line 11 says the rule is `replayCard && !tts.speaking`. That's a real bug, but it only manifests once real audio is wired — i.e., it's **dormant until S6 (halo-audio bridge) + S8 (voice consolidation)** land. Fixing the gate without the bridge would be polish without outcome change.

So the highest-value Wave 1 next step is a **small, cohesive, velocity-positive bundle that advances the "mastery is invisible to the user" theme without touching voice architecture**. That's the S10 + S11 + S18 triplet. Three surface-level fixes, one PR, ~3h of work, immediately visible.

Fix C (S4 signed-out home) is next after this — bigger lift, needs a design pass, best done as a dedicated PR.

---

## Problem

The mastery machine does real work in state and renders it as theater the user can't read or interpret. Three concrete instances, all independent, all tiny:

### S10 — `OneFixReplayCard` hides its most important metadata
- **Where**: `components/voice/OneFixReplayCard.tsx:50-55`, `hooks/voice/slices/replayCardSlice.ts:18-23`
- **What's broken**: `OneFix` has `dimension: RubricDimension` and `scoreBoost: number`. The card renders only `sentence` and `rewrite`. The user reads a rewrite with no indication of which of the five rubric dimensions it targets or how much it unlocks. "Am I improving on the right things?" is unanswerable.

### S11 — `CarryForwardChip` shows a coaching target with zero context
- **Where**: `components/voice/CarryForwardChip.tsx:64`
- **What's broken**: Chip renders `chip.shortLabel` only (e.g., "Quantify Impact"). No `title`, no `aria-label`, no link to the rubric rule. **But the data already has it**: `coachingTargetSlice.ts:17` defines `rule: string` on the chip — the full coaching rule the coach picked. It's just not surfaced to the DOM. One-field fix.

### S18 — `MasteryDebugOverlay` has no `NODE_ENV` gate at the component level
- **Where**: `components/voice/MasteryDebugOverlay.tsx:41-94`
- **What's broken**: Component body renders unconditionally. Line 17 comment says "Mount on the preview harness or behind a dev flag on real sessions" — but the comment is only honored by callers, not the component itself. Any accidental mount in production leaks the hidden tuple, knobs, and state machine internals to real users. One-line fix that closes a real information-leak vector.

---

## Goal

Three concrete changes that make the mastery loop legible without breaking its aesthetic ("whisper," "never modal, never audible," "hidden tuple stays hidden"). Zero regressions on the existing surfaces.

---

## Non-goals

- **Not changing `OneFixReplayCard`'s gate semantics** — that is S7, and it depends on S6 (halo-audio bridge) which depends on S8 (voice consolidation). Out of scope.
- **Not turning `CarryForwardChip` into a modal** — comment at line 12 says "Never modal, never audible, never scored." Tooltip/aria only. No click interaction, no popup, no info panel.
- **Not gating `MasteryDebugOverlay` with a runtime flag** — keep it to `process.env.NODE_ENV`. A query-param flag is considered in alternatives but not recommended for this PR.
- **Not adding new mastery surfaces.** Every change in this PR touches existing rendered components only.
- **No new test frameworks, no refactor of the voice store, no reorganization of the slice tree.**

---

## Design

### Change 1 — `OneFixReplayCard` renders dimension + scoreBoost

**Add a small header line above the sentence**, in the existing whisper aesthetic. Uppercase tracking, muted color, tabular numerics for the score.

**Dimension copy** — add a pure helper in the component file (or in `replayCardSlice.ts` as a small export) mapping the `RubricDimension` enum to human-readable labels:

```ts
const DIMENSION_LABEL: Record<RubricDimension, string> = {
  specificity: 'Specificity',
  quantified_impact: 'Quantified Impact',
  role_outcome_clarity: 'Role & Outcome Clarity',
  ownership: 'Ownership',
  level_appropriate: 'Level-Appropriate',
}
```

**Score rendering** — `scoreBoost` is a 0-1 float per the sweep audit (the mock uses 0.8). Render as a percentage gain: `+${Math.round(scoreBoost * 100)} pts`. Example: `+80 pts`. Or if the team prefers decimals, `+0.8`. **Recommend: percentage gain** — readable, intuitive, no confusion with Elo points.

**Layout** — add one line above the sentence block, same max-width, same backdrop:

```tsx
<div className="mb-2 flex items-center justify-between text-[9px] font-semibold uppercase tracking-[0.18em] text-neutral-400/80">
  <span>{DIMENSION_LABEL[replayCard.dimension]}</span>
  <span className="tabular-nums text-emerald-300/80">
    +{Math.round(replayCard.scoreBoost * 100)} pts
  </span>
</div>
<p className="text-[15px] font-normal leading-relaxed text-neutral-300">
  {replayCard.sentence}
</p>
<p className="mt-3 text-[17px] font-medium leading-snug tracking-tight text-white">
  &ldquo;{replayCard.rewrite}&rdquo;
</p>
```

Respects the whisper: small text, muted, no grid, no header. The dimension label is an eyebrow, not a banner.

### Change 2 — `CarryForwardChip` gets `title` + `aria-label` from `chip.rule`

**One-field change**. `coachingTargetSlice.ts:17` already has `rule: string`. Wire it to the outer motion.div:

```tsx
<motion.div
  key={chip.chipId}
  layout
  title={chip.rule}
  aria-label={`Coaching target: ${chip.rule}`}
  initial={...}
  ...
>
```

- `title` gives a native browser tooltip on hover — respects "never modal" because tooltips are OS-level, not DOM overlays.
- `aria-label` gives AT users the full rule text instead of just the shortLabel.
- No visual change. Exactly zero new pixels on screen. The chip still renders `shortLabel` as the only visible text.

If the coaching rule text isn't user-friendly (e.g., it's internal language like `"rule_quantify_impact_v2"`), add a second helper mapping rules to user-facing copy. **Investigation**: check what's actually in `chip.rule` during a real session by inspecting the backend payload — if it's already user-facing text, ship this as-is; if it's an ID, add the mapping layer in this PR.

### Change 3 — `MasteryDebugOverlay` NODE_ENV gate

**Hooks-safe guard**. Rules of hooks require hooks to run in the same order every render, so the production early-return must come **after** the `useVoiceStore` calls, not before them:

```ts
export function MasteryDebugOverlay(): JSX.Element {
  const hidden = useVoiceStore((s) => s.hidden)
  const label = useVoiceStore((s) => s.label)
  const knobs = useVoiceStore((s) => s.knobs)
  const tone = TONE_DOT[knobs.questionTone]

  if (process.env.NODE_ENV === 'production') {
    return <></>
  }

  return (
    // ...existing JSX unchanged
  )
}
```

Three-line change, hooks rule preserved, guarantees production users never see the overlay even if a caller accidentally mounts it.

**Why not also a `?debug=1` query-param flag?** Out of scope. Filed as follow-up: "allow `?debug=1` to enable overlay in production for bug-hunting." Not in this PR.

---

## Files changed

| File | Change | Est. lines |
|---|---|---|
| `components/voice/OneFixReplayCard.tsx` | Add `DIMENSION_LABEL` map + header line with dimension + scoreBoost | ~18 |
| `components/voice/CarryForwardChip.tsx` | Add `title` + `aria-label` wired to `chip.rule` (maybe +1 helper map if `rule` is an ID) | 2-8 |
| `components/voice/MasteryDebugOverlay.tsx` | Hooks-safe NODE_ENV production gate | 3 |
| `__tests__/voice/OneFixReplayCard.test.tsx` | New — dimension/scoreBoost renders, gate still works | ~60 |
| `__tests__/voice/CarryForwardChip.test.tsx` | New — title/aria-label reflect rule | ~30 |
| `__tests__/voice/MasteryDebugOverlay.test.tsx` | New — returns empty fragment in production NODE_ENV | ~25 |

Total: ~140 lines across 6 files. Single PR.

---

## Test plan

### Pre-flight
- [ ] `pnpm type-check` — clean on touched files.
- [ ] `pnpm lint` — clean.
- [ ] `pnpm build` — clean.

### Unit tests (Jest + @testing-library/react)
- [ ] `OneFixReplayCard`: mount with a mock voice store that returns `replayCard = { sentence, rewrite, dimension: 'quantified_impact', scoreBoost: 0.8 }` and `haloState = 'idle'` — assert "Quantified Impact" appears, assert "+80 pts" appears, assert sentence and rewrite still render.
- [ ] `OneFixReplayCard`: same fixture but `haloState = 'asking'` — assert nothing renders (gate still honored).
- [ ] `OneFixReplayCard`: all five `RubricDimension` values render their correct human-readable labels.
- [ ] `CarryForwardChip`: mount with a chip whose `rule` is `"Quantify every impact with a number"` — assert the motion.div has `title="Quantify every impact with a number"` and `aria-label="Coaching target: Quantify every impact with a number"`.
- [ ] `CarryForwardChip`: mount with `chip = null` — assert nothing renders (existing behavior preserved).
- [ ] `MasteryDebugOverlay`: set `process.env.NODE_ENV = 'production'`, mount — assert empty fragment, zero DOM output, zero voice-store subscription regressions.
- [ ] `MasteryDebugOverlay`: set `process.env.NODE_ENV = 'development'`, mount — assert full overlay renders (existing behavior).

### Playwright E2E on `beta.kitesforu.com` after deploy
- [ ] Navigate to `/interview-prep/mock/voice-preview`, start the simulation, advance to the "Between turns" step, confirm the one-fix card surfaces with a dimension label and a `+N pts` gain indicator. Screenshot.
- [ ] In the same session, confirm the `CarryForwardChip` appears in the corner with a full-rule tooltip on hover. Screenshot.
- [ ] Confirm `MasteryDebugOverlay` does NOT render on `beta.kitesforu.com` (production deployment) — inspect DOM, confirm absence.
- [ ] Regression: home page, pricing page, nav dropdown all unchanged.

### Screenshots
- [ ] `.vision_vault/fix-b/` — before + after for all three surfaces.

---

## Rollout

Per shipping-playbook memory — one atomic unit:

1. **Code PR** — branch `fix/b-mastery-surface-polish` in `kitesforu-frontend`.
2. **Docs PR** — if there's a help-center article on "How to read your mastery feedback," update it to mention the dimension labels and score gain indicator. Otherwise skip.
3. **Tooltip flag PR** — not required. Changes are static surface polish, no new guided tip.

Merge → verify Cloud Run revision → Playwright → move to Fix C (S4 signed-out home) next.

---

## Alternatives considered

1. **Fix C (S4 signed-out home) as Fix B instead.** Bigger impact, but 6-8h with design decisions (what does the signed-out home say? Parakeet-aligned? Where does the explainer go?) — slower velocity, needs a design sync. Better to land this small bundle first while the signed-out home gets proper design attention.

2. **Fix S1 gate semantics properly** (align `OneFixReplayCard` guard with real audio state). Would require wiring to `drive-audio-bridge` or a new "tts speaking" field in the voice store. Dormant bug today — symptom fixed in mock, only matters once real audio lands. Wave 3 Fix G territory.

3. **Bundle S10 + S11 + S18 with S2 (ribbon transitions)**. Rejected — S2 requires design input for a "pending label preview" UI element; that's Wave 2 Fix E with an open design question. Keep this bundle tight.

4. **Render dimension as an icon instead of a text label.** Rejected — five rubric dimensions don't map cleanly to icons, and icons without labels are an a11y regression.

5. **Rename `scoreBoost` to `scoreGain` or `impactDelta`.** Rejected — scope creep, touches the backend contract, separate refactor PR.

---

## Risks

1. **`chip.rule` may contain non-user-friendly text.** If the backend sends internal rule IDs (`rule_quantify_impact_v2`) instead of human-readable rules, the tooltip will be cryptic. **Mitigation**: inspect an actual chip payload during implementation (`console.log` in the preview harness or read a fixture). If IDs, add a `RULE_LABEL` map in the component file that falls back to `shortLabel` if the rule is unknown. Flag this in the PR description.

2. **`OneFixReplayCard`'s whisper aesthetic**. Adding a header line with two uppercase-tracking elements might tip the card from "whisper" into "grid/card." **Mitigation**: the designed header uses `text-[9px]` and `text-neutral-400/80` — smaller and more muted than the sentence. Visual check against the existing screenshot before merge.

3. **`+80 pts` gain number may be misleading.** If `scoreBoost` is actually a 0-1 confidence score from a downstream ML model, multiplying by 100 and labeling it "pts" is wrong framing. **Mitigation**: read the backend spec for `scoreBoost` before shipping. If it's not "points gained on the rubric dimension," rename the label to `confidence`, `improvement`, or whatever the spec says. This is a copy decision that needs a 30-second review.

4. **`MasteryDebugOverlay` production gate might break the preview harness** if the preview harness itself runs under `NODE_ENV !== 'development'` (e.g., a staging build). **Mitigation**: if the preview page is accessed from a non-dev build, gate the overlay on `process.env.NEXT_PUBLIC_ENABLE_DEBUG_OVERLAY === 'true'` instead of NODE_ENV. Check the preview page's build chain during implementation.

5. **Tests for `OneFixReplayCard` require mocking `useVoiceStore`.** Voice store is a Zustand slice — mockable via `jest.mock('@/hooks/voice/useVoiceStore')`. Precedent exists in the repo — reuse the pattern from whichever test file already mocks Zustand.

---

## Open questions for audit

1. **Score framing** — is `scoreBoost` actually a 0-1 rubric gain, or something else? If the latter, the copy needs a different label. Name the correct user-facing framing before implementation.
2. **`chip.rule` content** — is it human-readable text or an internal ID? If ID, do we need a mapping layer in Fix B, or should the backend rewrite it?
3. **Preview harness NODE_ENV** — does the preview page ever run under a non-development build that still needs the debug overlay visible? If yes, use a `NEXT_PUBLIC_` flag instead of `NODE_ENV`.
4. **Wave 1 fix ordering after B** — is Fix C still S4 signed-out home, or do you want to deprioritize it in favor of a different Wave 2 fix?

---

## Triangulation

**STOP here.** This proposal does not authorize any code changes. Awaiting audit.

Pre-flight ground-truth verification complete: all three findings (S10, S11, S18) verified against main HEAD `a41d754` with specific line numbers. This proposal will not repeat the Fix A stale-read mistake.

Next in the Wave 1 sequence after Fix B merges: **Fix C (S4 — signed-out home story)**, to be drafted after this one ships and reviewed against the latest main.
