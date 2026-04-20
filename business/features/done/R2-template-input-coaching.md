# R2 — Per-Template Input Coaching (Typing Animation + Richly Detailed Example Per Template)

**Status**: SHIPPED (2026-04-20) — see Implementation Summary at bottom.
**Priority**: P1 — highest lever on output quality with the smallest change in user behavior
**Effort**: ~1 week framework + 1 day per template (parallelizable; ~34 templates)
**Origin**: Product-owner note, 2026-04-20 drive-test review.

## 1. Problem

Today's `/create-smart?template=...` input shows a single-line static placeholder per template:

> `The concept you're struggling with (e.g., "Eigenvalues and eigenvectors in linear algebra")`

It is accurate, brief, and **deeply under-teaches**. The placeholder communicates "this box expects short text," so users type short text. Short text leads to generic outputs. Generic outputs are either discarded or refined through multiple follow-up rounds in the planner, which burns credits, wall-clock, and user patience.

The product-owner note (verbatim, 2026-04-20):

> "I like the relevant info you have here, but can you make it look like typing and a bigger box and more specific information. You will need deep research agents for each scenario/type we have in the system and think through what we can provide. The objective is to show and teach the user exactly how they can give some very specific data that can help us create an awesome specific output. Now if the input is good all sorts of follow up questions can be avoided. We cant force it. But we can show and tell by example with very very very good examples for each specific scenarios."

The core insight: **inputs are the single highest-leverage quality lever.** Every minute spent upfront on specific, textured, context-rich input saves 3–5 minutes of follow-up clarification and usually yields a better final output. But the input box today does not *teach* the user what "rich" even means. This proposal makes it teach.

## 2. Relationship to other proposals

This is distinct from but complementary to:

- `R2-curated-type-examples.md` — proposes a per-**top-level-type** example rotation (podcast / course / story-series / etc.) on the home hero. That proposal is about discovery at the mouth of the funnel.
- `R2-rich-input-with-attachments.md` — proposes a Tiptap composer so the input *can* hold richer payloads (files, chips). Phase 1 shipped in PR #465 (feature-flagged, text-only).

**This proposal** is about per-**individual-template** input coaching (~34 templates today, one exemplary prompt each). It assumes the rich composer surface exists and raises the **quality of what the user types into it** by showing an exemplary prompt typing itself out, in a bigger box, with the outcome preview visible below.

## 3. Evidence

- Template metadata + placeholders: `kitesforu-frontend/lib/templates.ts:1-750` — 34 templates across 8 personas. Each template has a short `defaults.topicHint`.
- Current "Say it" input: `components/smart-create/IntentSection.tsx:283` — `rows={3}`, placeholder is either the template's `topicHint` or a generic fallback.
- Current rotating hero placeholder: `components/home/HeroSection.tsx:8-13` — four shallow generic strings cycled.
- Existing typing engine (reusable): `components/home/HeroSection.tsx:37-51` — 45 ms/char typewriter + 3 s pause. Proven pattern.
- Smart Create planner flow: `hooks/useSmartCreateChat.ts:275-307` — today the planner asks clarifying questions when input is thin. Every avoided clarification saves an LLM call + a user turn.

## 4. Goal

For every template, show a **richly-detailed exemplary prompt typing itself out** in a **bigger input box**, with an **outcome preview** below so the user can see the cause/effect: *"this much detail → this kind of output."*

## 5. Proposal

### 5.1 System shape (this proposal)

- A new versioned TS module `lib/template-coaching.ts` holds one `TemplateCoaching` entry per template id.
- `IntentSection` reads the entry for the currently-selected template and passes it to the input. The input auto-expands (rows 3 → 8–10) while coaching is active. The typing engine replays the exemplary prompt. On any keystroke the animation stops and clears.
- A small "See a good example" link re-plays the animation on demand without clobbering user input.
- A subtitle below the input displays the `outcomePreview` while the animation is playing.

### 5.2 Per-template content (implementation time)

> **Scope note**: This proposal specifies the *system* and the *content contract*. The canonical exemplary prompts themselves are produced by **per-template deep research agents** spawned during implementation — one agent per template. That is deliberate; a single "generic" prompt for all 34 templates would defeat the entire point.

Agent brief template (one per template):

- Study the 3 highest-rated existing outputs that used this template on KitesForU (Firestore + feedback counters from R2 feedback capture).
- Study 5 external exemplars that match the template's intent (e.g. for `concept-deep-dive`: 3Blue1Brown, Feynman lectures, MIT OCW; for `horror-series`: Old Gods of Appalachia, The Magnus Archives).
- Produce **one** exemplary prompt (300–800 chars) that is maximally specific and evocative without being over-prescriptive. It should demonstrate the density of detail that unlocks quality output, not be a rigid recipe.
- Produce **one** `outcomePreview` (≤120 chars) that sketches what the user can expect: *"A 15-minute deep dive covering geometric intuition, the Schur decomposition, and two worked examples from image compression."*
- Mark the agent + timestamp in the entry so stale prompts can be re-researched when upstream evidence shifts.

### 5.3 Data model — versioned TS constants

```ts
// lib/template-coaching.ts
export interface TemplateCoaching {
  templateId: string              // matches lib/templates.ts id
  exemplaryPrompt: string         // 300–800 chars, richly detailed
  outcomePreview: string          // one-line sketch of what this produces
  typingSpeedMs?: number          // default 45
  coachLabel?: string             // "See a good example" (overrideable per template)
  researchedBy?: string           // audit trail — which agent produced this
  researchedAt?: string           // ISO date
  version: number                 // bump on copy change; unlocks A/B later
}

export const TEMPLATE_COACHING: Record<string, TemplateCoaching> = {
  // populated incrementally as per-template research agents return
}
```

Why versioned TS constants, not a backend service: copy iterates weekly, the full set is <200 KB raw, it ships on next build, it's reviewable in PRs, it needs no schema migration. If per-user A/B later proves necessary, a feature-flag layer sits over the constants without changing them.

### 5.4 Typing animation UX

- **Bigger box**: when the selected template has a `TemplateCoaching` entry, the input auto-expands to `rows={8}` (RichComposer) or `min-height: 200px` (legacy textarea). Collapses back when user clears / switches template.
- **Typing engine**: reuse `HeroSection`'s engine (extract to `lib/typewriter.ts` so both the home hero and the Smart Create input share it). 45 ms/char is the house tempo; per-template override via `typingSpeedMs`.
- **Interrupt behaviour**: any keystroke, voice input, paste, or focus on the textarea cancels the animation and clears the shown-but-not-submitted text. Prevents the typing animation from pretending to be user content.
- **Replay**: small "See a good example" link under the input replays the animation without clobbering the user's current input (it's still purely a visual demo — the submit button remains disabled until the user types something themselves).
- **Accessibility**: the animation is `aria-hidden="true"` (it's decorative); the input itself keeps an `aria-label` referring to the static placeholder copy. Users on screen readers hear the placeholder, not the animated example. `prefers-reduced-motion` skips the typing effect and shows the exemplary prompt fully rendered as a dimmed ghost.

### 5.5 Outcome preview

Below the input, while the animation plays or the user's input is empty:

> *"What you'll get: a 15-minute deep dive covering geometric intuition, the Schur decomposition, and two worked examples from image compression."*

Direct causal link between input quality and output shape. Disappears once the user starts typing (their input is now the source of truth; the previous outcome preview no longer applies).

### 5.6 Implementation milestones

1. **Framework (~1 week)**: schema, module skeleton, shared `lib/typewriter.ts`, IntentSection wire-in behind `feature_template_coaching_live` (default OFF), tests.
2. **Pilot template (~1 day)**: spawn one deep-research agent on `concept-deep-dive` (the template used in the product-owner's screenshot). Merge the framework + pilot together so the feature has proof-by-demonstration on day one.
3. **Remaining 33 templates (~2 weeks wall-clock, parallelizable)**: spawn one research agent per template, batch-merge 5–8 templates per PR, flip the flag ON when ≥80% coverage lands.

## 6. Acceptance criteria

- [ ] `lib/template-coaching.ts` exists with `TemplateCoaching` schema + at least `concept-deep-dive` populated before framework PR merges.
- [ ] `HeroSection` typewriter extracted into `lib/typewriter.ts` and consumed by both hero + smart-create input.
- [ ] IntentSection auto-expands the input (legacy textarea → rows 8; RichComposer → `min-height: 200px`) when a template with coaching is selected.
- [ ] Typing animation plays on template selection; user keystroke / paste / voice input interrupts + clears it cleanly.
- [ ] "See a good example" link replays the animation without altering user input.
- [ ] `outcomePreview` renders below the input while animation plays, hides on user input.
- [ ] `prefers-reduced-motion` users see the exemplary prompt rendered statically as a dimmed ghost — no motion.
- [ ] Animation is `aria-hidden`; screen readers get the static placeholder only.
- [ ] Every shipped `TemplateCoaching` entry has `researchedBy` + `researchedAt` so staleness audits are possible.
- [ ] Feature flag `feature_template_coaching_live` default OFF until ≥80% template coverage.
- [ ] Playwright test: navigate to `/create-smart?template=concept-deep-dive`, confirm animation fires, confirm user keystroke interrupts, confirm replay link re-runs animation without wiping input.
- [ ] Bundle-impact: <10 KB gzipped for the framework (all per-template copy is already in the main bundle via lib/templates; coaching text piggybacks on that).

## 7. Risks + mitigations

- **Fatigue on repeat visits**: seeing the same animation every time is annoying. Mitigation: cache "has seen coaching for template X" in localStorage; subsequent visits show the exemplary prompt as a static dimmed ghost (still educational, not repetitive).
- **Copy rot**: as the platform evolves, exemplary prompts stop matching actual best outputs. Mitigation: `researchedAt` timestamp + a quarterly re-research batch job (CI warning when any entry is >6 months old).
- **Over-specific examples make the product feel narrow**: if the exemplary prompt for `horror-series` is always "vampires in 5 episodes," users think that's all it does. Mitigation: each template gets two alternates in a `TemplateCoaching.alternates?: string[]` field for rotation on replay.
- **Animation causes perceived lag**: Tiptap + typing both animating at once could thrash. Mitigation: animation uses `requestAnimationFrame` inserts against the editor's transaction, never re-renders React between chars.
- **Voice-first users never see the animation**: they tap mic and talk. Mitigation: on mic-activate, speak a short audio prompt: "For best results, describe your topic, audience, and what you want to understand." Same teaching, different channel.

## 8. Out of scope

- Dynamic / LLM-generated examples at runtime (cost + latency; canonical curation is the bar).
- Per-user personalized examples — that's R3 profile-driven-personalization's job once Phase 1 ships.
- i18n of exemplary prompts (English first).
- Replacing the planner's clarifying-question flow (the planner remains as a safety net for users who still type thin input).
- Full-document examples (≥1000 chars) — the point is to teach richness, not to bait the user into pasting War and Peace.

## 9. Open questions

1. Should the animation auto-play once per template-session or once per user-per-template (persist in localStorage)? Leaning toward once-per-session with a subtle replay affordance.
2. Do we surface the `researchedAt` timestamp anywhere user-facing, or keep it debug-only? Debug-only for v1.
3. Alternates rotation: weighted random, or strict round-robin? Strict round-robin is more predictable for QA.

## 10. References

- `components/smart-create/IntentSection.tsx:283` — placeholder emission point for the current textarea.
- `components/smart-create/RichComposer/RichComposer.tsx` — Phase 1 Tiptap composer (feature-flagged, shipped PR #465).
- `components/home/HeroSection.tsx:37-51` — existing typing engine to extract into `lib/typewriter.ts`.
- `lib/templates.ts` — source of truth for the 34 template ids + their current `topicHint` strings.
- Sibling proposals:
  - `R2-curated-type-examples.md`
  - `R2-rich-input-with-attachments.md`
  - `R3-profile-driven-personalization.md`

---

## Implementation Summary (2026-04-20) — SHIPPED 100%

**Flag**: `feature_template_coaching_live = true` in `lib/feature-flags.ts`. Verified live on beta.kitesforu.com.

**Frontend PRs (kitesforu-frontend)**:
- `#466` — framework + `lib/typewriter.ts` shared engine + `lib/template-coaching.ts` schema + `IntentSection` consumer + pilot (`concept-deep-dive`).
- `#467` — batch 1 (6 templates).
- `#470` — batch 3 (8 templates) + hotfix renaming `employee-onboarding` → `corp-onboarding` (batch 2's entry was keyed on a non-existent template id).
- `#471` — batch 4 (6 templates) + flipped `feature_template_coaching_live` ON. Coverage crossed the 80% threshold (29/34 ≈ 85%); uncovered templates fall through to the pre-existing placeholder via null-guarded `coaching && …` branches in `IntentSection`, so the remainder ship is graceful degradation with zero regression risk.
- `#472` — batch 5 (final 5 templates: k12-middle, k12-quick-review, uni-graduate, uni-review, quick-test). **Coverage now 34/34 (100%).**

**Per-template coaching content** came from 34 parallel deep-research agents (one per template), each grounded in real-world exemplars — Pramp/Exponent (interview-prep), Stratechery/Matt Levine (news-digest), Taylor Jenkins Reid / Sally Rooney (romance-drama), NGSS/CCSS/TEKS (K-12), Andy Grove / Julie Zhuo / Brené Brown (corp-leadership), Chomsky/Athey/Vaswani papers (uni-graduate), etc. Each entry is a 300–800-char primary prompt + 2 textured alternates, stamped `version: 1, researchedAt: '2026-04-20'`.

**Verification**: Playwright smoke on beta. Tech Interview showed `"What you'll get: … 25-min coached session drilling tree DP state design and a rate-limiter mock critique."` + "See another example" replay link. Middle School (uncovered at the time of test) correctly fell through to the default placeholder. Screenshots in `.vision_vault/`.

**Deviations from the proposal**:
- Coverage ramped in 5 batches (pilot + 4 batches) rather than a single content drop; the flag flipped at 85%, with the final 5 templates landing the same day.
- Typewriter engine lives at `lib/typewriter.ts` (shared module) instead of refactoring the inline `HeroSection` effect — cleaner reuse for the sibling `R2-curated-type-examples` work.
- No big-box refactor of the home hero yet (in scope for `R2-curated-type-examples`).

**Deferred follow-ups**:
- Per-user A/B of alternates (proposal §7 mentioned this; deferred until we see per-template engagement data).
- i18n of exemplary prompts (English-only MVP).
- Telemetry on "See another example" click-through (trivial to add once we're measuring funnel lift).
