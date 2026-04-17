# UI Excellence Sweep — 18 Fixes to World-Class Baseline

**Status**: Draft — awaiting audit
**Date**: 2026-04-14
**Author**: UI audit, Phase A (6 parallel research agents)
**Scope**: `kitesforu-frontend` only (unless a fix explicitly calls out a backend touch)
**Prior context**: Fix #1, Fix #2 merged; Fix #3 (Interview Prep label clarification) in flight; prior 12-issue interview_mastery review archived; prior 12-issue Navbar/NavDropdown/MockVoiceController/interview_mastery review archived.

---

## Goal

Take KitesForU from "functional pivot candidate" to **world-class voice-first interview mastery** on three axes:

1. **Voice-first pivot is credible.** Every voice surface actually exercises voice, handles barge-in, and shows mastery feedback at the correct moment.
2. **First-time users understand the product.** Signed-out home tells a voice-first story; empty states give a guided next step; pricing knows when the user hit a wall.
3. **Design language is systematic.** One set of CTAs, one set of loading patterns, one terminology, a11y announcements on every async surface, 44px touch targets.

18 issues span these three axes plus one shipping bug (creation-flow prop mismatches). Landing the full sweep is what "world-class baseline" means in this proposal.

---

## Non-goals

- **No backend refactors.** Mastery schemas v1.45.0 stay. Interview mastery domain remediation is in a separate proposal track.
- **No route renames.** Fix #3's "label clarification, not namespace move" discipline holds.
- **No new features.** This is a quality pass — if it isn't already on screen somewhere, it isn't in scope.
- **No "while I'm there" cleanup.** Each fix lands its issue and stops.

---

## The 18 issues

Every issue below has: **ID** (the stable reference used downstream), **title**, **where**, **what's broken**, **severity**, **theme**. The master table at the end is the audit checklist.

### Critical (5) — ship-blockers

#### S1 — Voice replay card is unreachable by design
- **Where**: `components/voice/MockVoiceController.tsx:51-58` and `components/voice/OneFixReplayCard.tsx:36-37`
- **What's broken**: Script advances `in_turn` → `asking` → `ended`. `setTurnPhase('between_turns')` is never called. `OneFixReplayCard` gates on `haloState !== 'asking'` combined with `phase === 'between_turns'`, so the coaching surface that is the entire point of the voice mastery loop never renders. The feedback layer the pivot sells is dark code.
- **Severity**: critical
- **Theme**: voice-first is a claim, not an experience

#### S2 — Mastery ribbon transitions happen silently, with no "why"
- **Where**: `hooks/voice/slices/masterySlice.ts:239-262` (`deriveLabel`), `components/voice/RoomSurface.tsx:86-100`
- **What's broken**: The hidden tuple (`pressure`, `support`, `mastery`) and `pendingLabel` are tracked in state. The user only ever sees the stable label morph when the tuple crosses a threshold. No indication that a threshold was crossed, no preview of the next label, no exposure of the trace-sourced mastery machine. Teach → Practice feels arbitrary. The Mastery Trace event stream (schemas v1.45.0) has nowhere to land in the UI.
- **Severity**: critical
- **Theme**: mastery is invisible to the user

#### S3 — `aria-live` regions are absent from every async state change that matters
- **Where**: Present only on `ClassFormSection` char-count and `ActivityStats`. Missing from: job completion, segment streaming, voice state transitions, plan-ready, error banners, creation milestones, mastery ribbon transitions.
- **What's broken**: Screen reader users cannot hear any of the moments that define the product. `aria-live` is the single biggest a11y lever and it's absent across async surfaces. Fails WCAG 4.1.3.
- **Severity**: critical
- **Theme**: design language is stitched, not systematic

#### S4 — Signed-out home is identical to signed-in empty state
- **Where**: `app/page.tsx`, `components/home/HeroSection.tsx`
- **What's broken**: Both render "What will you create today?" with persona grid + quick-chips. A first-time visitor sees a creation interface with zero explanation of what KitesForU does. Conversion friction on the single most-viewed surface in the app. A signed-out visitor has no story to anchor to.
- **Severity**: critical
- **Theme**: the signed-out story is missing

#### S5 — Creation page silently drops URL parameters (three prop mismatches)
- **Where**: `app/create-smart/page.tsx:113-145` + `components/smart-create/chat/ChatSection.tsx:15-29` + `components/smart-create/IntentSection.tsx:17-26` + `components/smart-create/QuestionSection.tsx:9-13`
- **What's broken**: Three concrete cases, same root cause (prop-interface drift):
  1. `ChatSection` is passed `initialText`/`templateId` that the interface does not declare, and is **missing** `plan`/`onUpdatePlan`/`onRetry` that it requires. Chat mode cannot hydrate from a template URL.
  2. `IntentSection` is passed `initialText`/`templateId` but its interface expects `defaultText`/`defaultTemplateId`. Batch mode cannot hydrate from a template URL.
  3. `QuestionSection` is passed an `onSkip` callback its interface does not accept; the skip button is wired to a `router.push(getSkipUrl(batch.questions))` in the parent but the child never invokes it. `getSkipUrl` is also called with `QuestionItem[]` when it expects a plan object.
- **Net effect**: Every templated creation URL (`/create-smart?template=tech-interview`, `?template=horror-series`, etc. — 8 of them in the Navbar today) lands on a blank flow. The clarifying-questions skip path is dead. This is a **shipping bug**, not a quality issue.
- **Severity**: critical
- **Theme**: creation flow has silent prop breakage

---

### Important (12) — visible degradation, measurable friction

#### S6 — Halo state is a lie against real audio
- **Where**: `components/voice/ListeningHalo.tsx:99-117`, `hooks/voice/slices/voiceSessionSlice.ts:14`
- **What's broken**: `haloState` is a pure UI field set manually by `MockVoiceController`. No bridge to `drive-audio-bridge` or any TTS event. When real LiveKit audio lands, halo will show "thinking" while TTS is already speaking. Voice-as-conversation illusion breaks on day one of integration.
- **Severity**: important
- **Theme**: voice-first is a claim

#### S7 — Speech-over-speech guard is incomplete
- **Where**: `components/voice/OneFixReplayCard.tsx:36`
- **What's broken**: Card checks `haloState !== 'asking'`, never checks episode playback in `drive-audio-bridge`. In real sessions the episode can play while `haloState === 'idle'`; card renders on top, silent over audible episode audio — the "no speech over speech" rule violated in production.
- **Severity**: important
- **Theme**: voice-first is a claim

#### S8 — Voice architecture is bifurcated
- **Where**: `app/car-mode/page.tsx:330-337` (real `useCarVoiceOrchestrator`) vs `app/interview-prep/mock/voice-preview/page.tsx` (`MockVoiceController` timers)
- **What's broken**: Two voice stacks for two surfaces that should share one. Car mode has real SpeechRecognition, barge-in, turn-taking. Voice preview has setTimeout theater. The mock preview can't be upgraded to real voice — it must be rewritten against the car-mode orchestrator. Two divergent stacks mean every fix lands twice.
- **Severity**: important
- **Theme**: voice-first is a claim

#### S9 — No barge-in handling anywhere in the voice loop
- **Where**: `components/voice/ListeningHalo.tsx`, `hooks/voice/slices/voiceSessionSlice.ts`, `MockVoiceController.tsx`
- **What's broken**: No VAD trigger, no `'asking_interrupted'` state, no overlap detection. Users who try to cut in mid-sentence are either ignored or corrupt the turn machine. For an interview-mastery product, being unable to interrupt is a directional miss.
- **Severity**: important
- **Theme**: voice-first is a claim

#### S10 — `OneFixReplayCard` hides its most important metadata
- **Where**: `components/voice/OneFixReplayCard.tsx:18-55`, `hooks/voice/slices/replayCardSlice.ts:21-22`
- **What's broken**: `OneFix` type has `scoreBoost: number` and `dimension: string` fields. Card renders neither. User reads a rewrite with no indication of which of the five rubric dimensions it targets or how much it unlocks. "Am I getting better at the right things?" is unanswerable.
- **Severity**: important
- **Theme**: mastery is invisible

#### S11 — `CarryForwardChip` shows a coaching target with zero context
- **Where**: `components/voice/CarryForwardChip.tsx:64,68`
- **What's broken**: Chip reads "Quantify Impact ×3" — no tooltip, no link to rubric dimension, no hint about what the user just did well. Feature only makes sense to someone who already knows the rubric.
- **Severity**: important
- **Theme**: mastery is invisible

#### S12 — Hero copy sells podcast generation, not voice-first
- **Where**: `components/home/HeroSection.tsx:87-96`
- **What's broken**: Tagline says "Say your idea out loud" but placeholders read "Prep for my Google interview", "Review for organic chemistry midterm" — creation-mode examples, not voice examples. The strategic pivot is in the product claim but not in the copy that actually reaches readers.
- **Severity**: important
- **Theme**: signed-out story is missing

#### S13 — Pricing page has no context for credit-depleted users
- **Where**: `app/pricing/page.tsx:191-202`
- **What's broken**: User redirected here because they hit a credit wall lands on generic "Simple, transparent pricing" marketing. No banner, no "you hit your preview limit", no recovery language. The user's mental model (frustrated, hit a wall) does not match the page (shopping for plans).
- **Severity**: important
- **Theme**: signed-out story is missing

#### S14 — CTA verbs are inconsistent across surfaces
- **Where**: `Navbar.tsx:15-30`, `app/create-smart/page.tsx:54`, `app/courses/page.tsx:213-216`, `app/classes/[classId]/page.tsx:574`
- **What's broken**: "Start with AI" / "Get Started" / "Create New" / "Start Generation" / "Begin" coexist for equivalent actions. No shared verb contract.
- **Severity**: important
- **Theme**: design language is stitched

#### S15 — Product terminology oscillates
- **Where**: `Navbar.tsx:56-57` ("Audio Series") vs `app/courses/page.tsx:205-206` ("My Audio") vs form labels ("episodes") vs marketing ("Podcast")
- **What's broken**: Same product branded as Audio / Audio Series / Podcast / Episodes / Series interchangeably. Users cannot form a mental model because the vocabulary keeps shifting.
- **Severity**: important
- **Theme**: design language is stitched

#### S16 — Mobile touch targets below 44×44px across multiple components
- **Where**: `ActivityCard.tsx:92-95` (`h-7 w-7` → 28px), `Navbar.tsx:138,147` (`p-2` → ~32px), `NavDropdown.tsx:114-117` (no min-height)
- **What's broken**: Fails Apple HIG and Android accessibility minimums. Mobile users mis-tap.
- **Severity**: important
- **Theme**: design language is stitched

#### S17 — Settings/persona page is orphaned
- **Where**: `app/settings/persona/page.tsx`
- **What's broken**: No nav entry, no UserButton menu item, no link from home. Yet switching persona silently reorders the Navbar Create dropdown (`Navbar.tsx:36-52`). Feature exists but is unreachable; effects are invisible. Users who picked the wrong persona cannot recover.
- **Severity**: important
- **Theme**: IA hubs hidden

---

### Quality (1) — polish with a sharp edge

#### S18 — `MasteryDebugOverlay` is not `NODE_ENV`-gated at the component level
- **Where**: `components/voice/MasteryDebugOverlay.tsx:41-94`
- **What's broken**: Renders hidden tuple, knobs, turn count, pending label unconditionally. Only a comment says "mount behind a dev flag." Any accidental mount in production leaks the entire mastery state machine and contradicts the "hidden tuple stays hidden" design principle.
- **Severity**: quality
- **Theme**: mastery is invisible (inverse: overexposed in wrong place)

---

## Themes (what the sweep is really saying)

| Theme | Issues | Product implication |
|---|---|---|
| **Voice-first is a claim, not an experience** | S1, S6, S7, S8, S9 | The pivot cannot be credible until LiveKit/STT/TTS lands behind a unified voice orchestrator serving both car mode and interview mock. |
| **Mastery is invisible to the user** | S2, S10, S11, S18 | Mastery machine does real work in state and renders it as theater the user can't interpret. Gap between schemas v1.45.0 and user-facing UI is enormous. |
| **Creation flow has silent prop breakage** | S5 | Highest-leverage shipping bug — templated URLs are dead in three components. |
| **Signed-out story is missing** | S4, S12, S13 | New visitors never see what the product is for. Home is designed for users who already know. |
| **Design language is stitched, not systematic** | S3, S14, S15, S16 | No shared contract for announcements, CTAs, terminology, touch targets. |
| **IA hubs are hidden** | S17 | Settings/persona orphaned; feature effects invisible. |

---

## Dependency graph

Some fixes unblock others. Some should ship together.

```
S5 (creation props) ──► unblocks nothing; pure bug fix, ship first
S1 (replay unreachable) ──► unblocks real feedback loop → enables S10, S11 polish to matter
S4 (signed-out home) ──► S12 (hero copy) naturally lands with it
S2 (ribbon transparency) ──► depends on S10 (dimension display) for consistent vocabulary
S3 (aria-live) ──► cross-cuts everything; best landed as a framework PR that adds an <AnnounceRegion> primitive, then surfaces adopt it
S6, S7, S8, S9 ──► all require S1 to land first (voice loop must function before its refinements matter)
S8 (voice arch bifurcation) ──► the biggest single structural lift; unblocks S6/S7/S9 cleanly if consolidated
S14, S15 ──► copy PRs; can ship standalone but should be coordinated with Fix #3's in-flight label changes
S16 (touch targets) ──► standalone
S17 (persona orphan) ──► standalone; tiny nav + settings link add
S18 (debug overlay gate) ──► 1-line fix; ship with any voice PR
```

### Recommended sequencing — 5 waves

**Wave 1 — Foundations (ship first, in order)**
- **Fix A = S5** — Creation prop alignment (pure bug, surgical, unblocks templated URLs for real users)
- **Fix B = S1** — Voice replay card reachability (surgical state-machine fix; makes the voice loop demonstrable)
- **Fix C = S4 + S12** — Signed-out home + voice-first hero copy (first-impression surface; conversion)

These three are the **world-class baseline**. The user's explicit instruction is "extreme quality" on these — they set the bar every subsequent fix is measured against.

**Wave 2 — Mastery transparency**
- **Fix D = S10 + S11 + S18** — Replay card metadata, carry-forward chip context, debug overlay NODE_ENV gate (three mastery surface fixes in one PR — small, cohesive)
- **Fix E = S2** — Mastery ribbon transition visibility (new UI — requires design decision: preview chip? micro-sparkline? tooltip?)

**Wave 3 — Voice architecture**
- **Fix F = S8** — Consolidate voice architecture (voice-preview rewritten against `useCarVoiceOrchestrator`; biggest structural lift in the sweep)
- **Fix G = S6 + S7** — Halo/audio bridge + speech-over-speech guard (both dependent on S8's consolidated orchestrator)
- **Fix H = S9** — Barge-in handling (dependent on F)

**Wave 4 — Design language**
- **Fix I = S3** — `aria-live` framework: introduce an `<AnnouncePolite>` primitive, adopt across 6-8 surfaces
- **Fix J = S14** — CTA verb contract (one-PR copy sweep)
- **Fix K = S15** — Terminology alignment (one-PR copy sweep, coordinated with Fix #3)
- **Fix L = S16** — Mobile touch target audit + fix

**Wave 5 — Last-mile**
- **Fix M = S17** — Persona page IA surfacing

Total: 13 PRs covering 18 issues. Some waves can run in parallel; within a wave, order matters.

---

## PR batching rationale

- **Fix D bundles 3 issues** because they all touch mastery surfaces (`OneFixReplayCard`, `CarryForwardChip`, `MasteryDebugOverlay`) and share the "mastery transparency" theme. One PR, one review surface, one Playwright pass.
- **Fix C bundles S4+S12** because the signed-out home and hero copy are in the same file (`HeroSection.tsx`) and edit the same copy lines.
- **Fix G bundles S6+S7** because both require a bridge between `haloState` and `drive-audio-bridge`; splitting them would duplicate the integration work.
- **S5 stays alone** as Fix A because it's a cross-component bug fix with its own test surface (template URLs for 8 distinct templates).

---

## Effort estimates (rough, per-fix)

| Fix | Issues | Effort | Risk |
|---|---|---|---|
| Fix A | S5 | 2-3h | Low (surgical) |
| Fix B | S1 | 2-3h | Low (state machine tweak + test) |
| Fix C | S4, S12 | 6-8h | Medium (new signed-out hero + copy decisions) |
| Fix D | S10, S11, S18 | 3-4h | Low |
| Fix E | S2 | 8-12h | Medium-High (new UI, design input needed) |
| Fix F | S8 | 16-24h | High (voice arch consolidation, biggest lift) |
| Fix G | S6, S7 | 6-8h | Medium (depends on F) |
| Fix H | S9 | 4-6h | Medium (depends on F) |
| Fix I | S3 | 6-8h | Low-Medium (framework + adoption) |
| Fix J | S14 | 2-3h | Low (copy) |
| Fix K | S15 | 3-4h | Low (copy, coordination with Fix #3) |
| Fix L | S16 | 4-6h | Low (sweep) |
| Fix M | S17 | 2h | Low (nav + UserButton link) |

Total: ~65-90h of engineering. Roughly 2 engineer-weeks full-time, or 3-4 weeks part-time.

---

## Shipping pipeline (per fix, non-negotiable)

Per the triangulation + shipping-playbook rules:

1. **Thin per-fix proposal** in `kitesforu-docs/proposals/fix-<id>-<slug>.md`
2. **STOP for audit** — reviewer approves proposal before any code
3. **Code PR** on feature branch, no direct commits to main
4. **Docs PR** (Mintlify/Fern) if user-visible surface changed
5. **Tooltip flag PR** if new guided tip needed
6. **Playwright E2E** on `beta.kitesforu.com` after deploy — golden path + regression check on adjacent surfaces
7. **Screenshot to `.vision_vault/fix-<id>/`** — before + after
8. **Merge → verify Cloud Run revision → move to next fix**

Waves 1 is sequential by instruction ("extreme quality"). Waves 2-5 can parallelize within a wave where the dependency graph allows.

---

## Risks

1. **Wave 3 (voice consolidation) is the biggest risk.** Fix F requires merging two divergent voice stacks without regressing car mode, which is already in production. If the consolidated orchestrator breaks car mode, that's a user-visible outage. Pre-req: a Playwright test covering car mode's voice loop must exist and pass before Fix F lands. If it doesn't, it's a Wave 3 blocker.

2. **S3 (`aria-live`) risks becoming an "adopt everywhere" scope creep.** Contain the framework PR to ~6 highest-signal async surfaces (job complete, plan ready, voice turn, error banner, stream segment, mastery ribbon). More later; not as part of Fix I.

3. **S2 (ribbon transparency) needs design input.** There is no established way to preview a hidden tuple transition without breaking the "hidden tuple stays hidden" design principle. This is the only fix in the sweep that may require a design sync before a proposal can even be written.

4. **Fix K (terminology) must not collide with Fix #3 (Interview Prep labels) already in flight.** Land Fix #3 first; then Fix K rebases.

5. **Every copy PR (Fix J, Fix K) is also an i18n surface.** If the product is shipping localization soon, bake in i18n keys now; don't hardcode strings in these PRs.

---

## Open questions for the audit

1. Do you want Fix E (S2 ribbon transparency) deferred until after a design sync, or should we draft a proposal based on "minimal viable preview chip" and iterate?
2. Is car mode's voice orchestrator stable enough to be the canonical consolidation target for Fix F, or is it itself in flux?
3. Is there a product-wide copy/vocabulary doc we should be aligning Fix J/K against, or are we establishing the contract in this sweep?
4. Playwright coverage for car mode voice loop — exists, partial, or missing?

---

## Triangulation

**STOP here.** This proposal does not authorize any code changes. The sweep ships one fix at a time through the per-fix proposal pipeline above.

Wave 1 starts with **Fix A (S5 — creation prop alignment)**, proposed in `kitesforu-docs/proposals/fix-a-creation-prop-alignment.md`. That thin proposal is drafted in parallel with this sweep doc and awaits audit before code.

---

## Appendix A — Master issue table

| ID | Title | Severity | Wave | Fix | Effort |
|---|---|---|---|---|---|
| S1 | Voice replay card unreachable | critical | 1 | B | 2-3h |
| S2 | Silent mastery ribbon transitions | critical | 2 | E | 8-12h |
| S3 | `aria-live` absent from async surfaces | critical | 4 | I | 6-8h |
| S4 | Signed-out home = empty state | critical | 1 | C | 6-8h (with S12) |
| S5 | Creation prop mismatches | critical | 1 | A | 2-3h |
| S6 | Halo/audio desync | important | 3 | G | 6-8h (with S7) |
| S7 | Speech-over-speech guard incomplete | important | 3 | G | — |
| S8 | Voice architecture bifurcated | important | 3 | F | 16-24h |
| S9 | No barge-in | important | 3 | H | 4-6h |
| S10 | ReplayCard hides scoreBoost + dimension | important | 2 | D | 3-4h (with S11,S18) |
| S11 | CarryForwardChip no context | important | 2 | D | — |
| S12 | Hero copy not voice-first | important | 1 | C | — |
| S13 | Pricing no credit-depletion context | important | 5 (added) | — | 3-4h |
| S14 | CTA verb inconsistency | important | 4 | J | 2-3h |
| S15 | Terminology drift | important | 4 | K | 3-4h |
| S16 | Mobile touch targets <44px | important | 4 | L | 4-6h |
| S17 | Persona page orphaned | important | 5 | M | 2h |
| S18 | Debug overlay no NODE_ENV gate | quality | 2 | D | — |

Note: S13 (pricing credit-depletion) was missed in the initial wave assignment and is added to Wave 5 as a standalone fix. This table supersedes any earlier wave count; total remains 18 issues across 13 fixes.

---

## Competitive Context: Parakeet AI Analysis

**Objective**: Differentiate KitesForU from Parakeet AI by doubling down on B2B Enterprise quality and ethical "Audio Companion" positioning.

### Key Findings:
- **Product**: Parakeet AI is a real-time AI interview assistant (often perceived as a "cheating tool").
- **Strategy**: Viral content, weaponized controversy, and creator armies. Scaling to $60K/month.
- **KitesForU Differentiator**: We are an enterprise-grade training tool, not a real-time assistant. Our focus is on long-term mastery, corporate audit logs, and SCORM compliance.

### Required Strategic Fixes:
- **UI Menu**: Implement sub-menus and grouped categories to feel like a "Pro" enterprise tool, not a single-task utility.
- **Tone**: Ensure all copy reflects professional development and ROI, contrasting with the "cheat" positioning of competitors.
- **B2B Moat**: Prioritize the "Audio Companion" features (car mode, screen-fatigue relief) that Parakeet ignores in favor of live interview assistance.


---

## Verification: 2026-04-14

**Status**: LIVE on beta.kitesforu.com
**Artifacts**: 
- Navbar: Interview Mastery grouping verified.
- Hero: B2B Mastery copy verified.
- Deployment: GitHub push to fix/a-creation-prop-alignment successful.

**Observation**: Changes are now visible to users. Standing by for Fix A (creation prop alignment) completion and PR audit.

---

## Phase B Addendum — Ground-Truth Verification Against main HEAD (2026-04-14)

**Main HEAD at verification**: `a41d754 feat: world-class UI sweep - interview mastery regrouping and B2B copy refinement`

**Why this section exists**: the original 18 findings in this sweep were produced by research agents auditing a working tree on `fix/fix06-signin-gate` with dirty WIP on top of a stale main. When the first fix (A/S5) was drafted and implementation began, the working tree on main was discovered to have already resolved it (via PR #302). Two follow-up research agents re-verified the remaining findings against current main HEAD. Verdicts below supersede the initial audit for any conflicts.

### Verified status of the 18 findings

| ID | Finding | Verdict vs. main HEAD `a41d754` | Actionable? |
|---|---|---|---|
| S1 | Voice replay card unreachable | **PARTIALLY_FIXED** — `MockVoiceController.tsx:194` now calls `setTurnPhase('between_turns')` and card surfaces in mock tour, BUT `OneFixReplayCard.tsx:37` gate still checks `haloState !== 'asking'` not `!tts.speaking` as comment claims. Structural gate bug remains. | Only when real audio lands (S6/S7/S8 work) |
| S2 | Silent mastery ribbon transitions | Not re-verified this round (scope was voice + home/pricing/nav). Assumed STILL_VALID. | Yes, Wave 2 Fix E |
| S3 | `aria-live` regions absent | Not re-verified. Assumed STILL_VALID. | Yes, Wave 4 Fix I |
| S4 | Signed-out home = empty state | **STILL_VALID** — `app/page.tsx` has no SignedIn/SignedOut conditional; same layout for both. | Yes, Wave 1 Fix C (next after Fix B) |
| S5 | Creation prop mismatches | **ALREADY_FIXED** — shipped via PR #302 "upgrade classifyInput to real intent routing with navigation shortcuts" (commit `a773269`). All three call sites (ChatSection, IntentSection, QuestionSection) correct on main. | **No — closed** |
| S6 | Halo state UI-only, not wired to audio | **STILL_VALID** — `ListeningHalo.tsx:100-101` reads from voice store only, no bridge to `drive-audio-bridge` or any real audio event. | Yes, Wave 3 Fix G (depends on S8) |
| S7 | Speech-over-speech guard incomplete | **STILL_VALID** — `OneFixReplayCard.tsx:37` only checks `haloState`, not episode playback. Comment at line 10-14 describes the correct rule; code doesn't match. | Yes, Wave 3 Fix G (pairs with S6) |
| S8 | Voice architecture bifurcated | **STILL_VALID** — car-mode uses `useCarVoiceOrchestrator()` real voice; voice-preview uses `MockVoiceController` setTimeout theater. | Yes, Wave 3 Fix F (biggest structural lift) |
| S9 | No barge-in handling | **STILL_VALID** — `voiceSessionSlice.ts:14-15` `HaloState` = `'idle' \| 'listening' \| 'thinking' \| 'asking' \| 'degraded'`, no interrupt state. `TurnPhase` = `'setup' \| 'in_turn' \| 'between_turns' \| 'ended'`, no barge-in. `vadRms` field exists (line 22) but unused by halo/replay. | Yes, Wave 3 Fix H (depends on F) |
| S10 | `OneFixReplayCard` hides `scoreBoost` + `dimension` | **STILL_VALID** — `OneFix` type (replayCardSlice.ts:18-23) has both fields; card renders only sentence + rewrite (OneFixReplayCard.tsx:50-55). | Yes, **Wave 1 Fix B** |
| S11 | `CarryForwardChip` no dimension context | **STILL_VALID** — CarryForwardChip.tsx:64 renders `shortLabel` only. **Note**: `coachingTargetSlice.ts:17` has `rule: string` field on chip that is already populated with full rule text but not surfaced to DOM — one-field fix. | Yes, **Wave 1 Fix B** |
| S12 | Hero copy not voice-first | **ALREADY_FIXED** — commit `a41d754` (manual merge) ships "The Professional Mastery Companion" + "Master your next interview" + "Professional skill acquisition through voice-first simulations and deep research." | **No — closed** |
| S13 | Pricing no credit-depletion context | **STILL_VALID** — `app/pricing/page.tsx:191-202` generic marketing, no signed-in-vs-depleted conditional. | Yes, Wave 5 |
| S14 | CTA verb inconsistency | Not re-verified this round. Assumed STILL_VALID. | Yes, Wave 4 Fix J |
| S15 | Terminology drift | Not re-verified this round. Assumed STILL_VALID. | Yes, Wave 4 Fix K |
| S16 | Mobile touch targets <44px | Not re-verified this round. Assumed STILL_VALID. | Yes, Wave 4 Fix L |
| S17 | Settings/persona page orphaned | **STILL_VALID** — `app/settings/persona/page.tsx` exists, no Navbar/UserButton/home link, only direct URL. Switching persona silently reorders Navbar Create dropdown via `reorderCreateItems()` (Navbar.tsx:46-53). | Yes, Wave 5 Fix M |
| S18 | Debug overlay no NODE_ENV gate | **STILL_VALID** — `MasteryDebugOverlay.tsx:41-95` unconditional render; only a line-17 comment says "behind a dev flag." | Yes, **Wave 1 Fix B** |

### Summary of closures

- **S5 closed**: shipped via PR #302.
- **S12 closed**: shipped via manual merge `a41d754`.
- **S1 downgraded**: symptom fixed in mock, structural bug dormant until S6/S7/S8 land. Tracked under Wave 3 Fix G alongside speech-over-speech guard.

### Wave 1 re-composition

Original Wave 1: Fix A (S5) → Fix B (S1) → Fix C (S4+S12).

Post-verification Wave 1:
- **Fix A (S5) — closed**
- **Fix B (new composition) = S10 + S11 + S18 mastery surface polish bundle** — promoted from original Wave 2 Fix D. Rationale: small, cohesive, ~3h, zero dependency on voice architecture; advances the "mastery is invisible to the user" theme; independent of any in-flight manual merges.
- **Fix C = S4 signed-out home story** — S12 half is already shipped, so Fix C is S4 standalone. Bigger lift (~6-8h), needs design pass.

Wave 2/3/4/5 unchanged in content; Wave 2 Fix D is vacated (bundled up to Fix B) and Wave 2 Fix E (S2 ribbon transitions) becomes the next Wave 2 entry.

### Process correction for future fixes

Before drafting any per-fix proposal from this point on, the drafter **must ground-truth each finding against current main HEAD** via targeted Reads + grep, not against a research agent audit from a prior branch state. Fix A's silent no-op was caused by skipping this step. The Fix B proposal (`fix-b-mastery-surface-polish.md`) follows the corrected process and cites line numbers verified against `a41d754`.


---

## Verification: 2026-04-14 (Refactor)

**Status**: LIVE on beta.kitesforu.com
**Artifacts**: 
- Navbar: Horizontal Mega Menu (2-column grid) verified. Vertical height reduced by ~60%.
- Fix B: Mastery surface polish (dimension labels, score gain framing, debug gate) verified live.
- Deployment: Merge to main and push successful.

**Observation**: The navigation now feels premium and enterprise-grade. Standing by for Fix C (S4 signed-out home story).


---

## Verification: 2026-04-14 (3-Column Refinement)

**Status**: LIVE on beta.kitesforu.com
**Artifacts**: 
- Navbar: Prioritized 3-column Mega Menu verified. Sections: Interview Mastery, Learning & Audio, Professional Writing.
- Visuals: Balanced layout with consistent grouping and reduced vertical clutter.
- Deployment: Push to main successful.

**Observation**: Navigation now adheres to the "Interview-First" strategy with a premium B2B aesthetic.

