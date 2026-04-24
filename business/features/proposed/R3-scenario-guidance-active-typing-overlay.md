# R3 — Scenario Guidance: Active-Typing Overlay (keystone refactor)

**Status**: PROPOSED
**Priority**: P1 (user-deferred for "after current in-flight work complete"; that condition is now met with the scenario-guidance v2 data-layer sweep at 29/29)
**Effort**: ~2 days of focused UI work + ~1 day QA + a small number of tests
**Affected repos**: kitesforu-frontend
**Depends on**: R2-scenario-guidance-v2-density-and-preemption (data layer shipped 2026-04-21→24; UI layer mostly shipped; looped caret shimmer explicitly deferred on that proposal pending this refactor)
**Origin**: 2026-04-20 product-owner standing follow-up — *"improve the scenario/type guidance UI so it feels more like active typing, uses a bigger box, and gives much more specific example input."* R2-v2 closed the data layer + the "bigger box" + "more specific example" pieces. This proposal closes the remaining "feels more like active typing" piece via an architectural refactor that also unlocks several adjacent polishes.

---

## 1. One-paragraph thesis

The scenario-guidance composer currently renders its typing-animated exemplar into the plain `<textarea placeholder>` attribute (see `components/smart-create/IntentSection.tsx:706-710`). That attribute is a single opaque string — no per-character styling, no inline decorations, no cursor control, no HTML. R2-v2 pushed that medium as far as it can go: typewriter animation via string mutation, chip-pulse signals rendered BELOW the textarea, fade-to-next alternate cycling. The remaining "active typing" wins — looped caret shimmer at the end of the exemplar, inline clarifier highlights overlaid on the specific exemplar words that prove a clarifier, a smooth fade-out as the user starts typing (rather than the current on/off placeholder snap) — all require rendering the exemplar as real DOM. This proposal extracts an `<ActiveTypingOverlay>` component that layers the exemplar behind a transparent textarea, unlocks those polishes, and leaves every existing v2 feature intact.

## 2. What's shipped (grounding, not aspiration)

Confirmed by grep + read on 2026-04-24:

- `IntentSection.tsx:175` — `coachedPlaceholder` state drives the typewriter via `runTypewriter` (lib/typewriter.ts).
- `IntentSection.tsx:706-710` — plain textarea path renders `placeholder={coachedPlaceholder}` when scenario+empty-input.
- `IntentSection.tsx:674-699` — Tiptap-based RichComposer path behind `feature_rich_composer_live` (currently `false` per `lib/feature-flags.ts:149`).
- Scenario v2 features already live: `resolveBoxSizeRows` (#544), idle-fade-to-next cycling (#544), chip pulse at caret offset (#564), "your turn" nudge (#563), mobile 8-row cap (#564), analytics (#564).
- `boxSizeRows` explicit override wins when set; falls back to exemplar-length auto-resolver.

## 3. Why the current `<textarea placeholder>` approach caps out

| Desired UX | Possible with `placeholder` attribute? |
| --- | --- |
| Typewriter animation (char-by-char reveal) | Yes — mutate the string. Shipped. |
| Chip pulse on the nearest clarifier offset | Yes — chips are rendered OUTSIDE the textarea (already shipped). |
| **Looped caret shimmer at end of typed exemplar** | **No** — the attribute renders as a single grey string; per-char styling, cursor glyph with CSS animation, and per-char opacity transitions are all impossible on the attribute itself. |
| **Inline clarifier highlights** (underline the specific substring of the exemplar that proves a clarifier) | **No** — attributes are plain strings; HTML is stripped. |
| **Smooth fade-out as user types** | **No** — placeholder is binary (shows when empty, hidden when not). Today's UX is a sharp snap. |
| **Micro-jitter / breathing cursor** | **No** — no DOM node means no CSS animation target. |

Conclusion: every remaining active-typing polish the product-owner asked for requires real DOM behind the input surface, not the `placeholder` attribute. That's the architectural keystone.

## 4. Proposed architecture — `<ActiveTypingOverlay>`

A small overlay component layered behind a textarea whose own text-color matches the background. Pattern is common in autocomplete UIs (GitHub's inline suggestions, Copilot's ghost text).

```
┌─────────────────────────────────────────────┐
│  <div class="relative">                     │
│    ┌───────────────────────────────────┐    │
│    │ <ActiveTypingOverlay              │    │
│    │   aria-hidden                     │    │
│    │   typedText={coachedPlaceholder}  │    │
│    │   userTextLength={promptText.len} │    │
│    │   cursorVisible={!isTyping}       │    │
│    │   clarifierMarkers={...}          │    │
│    │ />                                │    │
│    │  ↑ absolute, same font/padding    │    │
│    │    as the textarea, pointer-events:none│
│    └───────────────────────────────────┘    │
│    ┌───────────────────────────────────┐    │
│    │ <textarea                         │    │
│    │   value={promptText}              │    │
│    │   placeholder=""  ← cleared       │    │
│    │   className="..."                 │    │
│    │ />                                │    │
│    │  ↑ sits on top, transparent-ish   │    │
│    │    background so overlay shows    │    │
│    └───────────────────────────────────┘    │
│  </div>                                     │
└─────────────────────────────────────────────┘
```

**The overlay renders:**
- The current `coachedPlaceholder` text as grey DOM text (same typography as the textarea — not a visual glitch).
- A live `▎` caret element at the end of the typed string — CSS `@keyframes caretShimmer { 0%,50%,100% { opacity: 1 } 25%,75% { opacity: 0.3 } }` with `animation-duration: 1.2s`.
- Inline clarifier markers — when a `clarifiersCovered` entry has a `substringMatch` field (new optional), the overlay wraps that substring in a subtle `border-b border-dotted border-brand-300/60` span that fades in as the typewriter reaches that offset. If no substring is provided (most current v2 entries), the overlay skips inline markers and the below-the-box chip-pulse behavior continues to do the work.
- Opacity controlled by `Math.max(0, 1 - userTextLength / FADE_START_CHARS)` where `FADE_START_CHARS = 24` — the overlay gracefully fades over the user's first ~24 characters instead of snapping away.

**The textarea:**
- `placeholder` cleared to empty string.
- `className` adds `bg-transparent` so the overlay underneath remains visible.
- `value` is the user's actual input. When empty, the overlay reads as the "placeholder"; when typed, the user's text layers on top (visually clean because the overlay fades as they type).

**User input rendering:**
- The user's typed text uses the standard textarea color/weight — no change from today.
- The overlay's typed-exemplar text uses the existing placeholder grey (`placeholder-gray-400` / `dark:placeholder-zinc-500`), so visually this looks identical to the current state when the input is empty.

## 5. What this unlocks (in addition to #4's baseline)

1. **Looped caret shimmer** — was explicitly deferred in R2-v2 for this refactor. Now trivially a CSS keyframe on the overlay's `<span aria-hidden>▎</span>`.
2. **Smooth user-typing fade** — overlay opacity ramps from 1 → 0 across the first 24 characters the user types. Today's snap-off feels abrupt; the ramp reads as "the coach steps back as you step in."
3. **Inline clarifier highlights** (phase-2 within this proposal) — exemplars that carry `substringMatch` spans get subtle underline on the substring, fading in in sync with the typewriter offset. Pairs with the existing chip-pulse rather than replaces it.
4. **Reduced-motion friendliness** — the whole overlay collapses to a static grey string under `prefers-reduced-motion`, matching today's behavior exactly. No regression for users who opt out of motion.
5. **Dark-mode parity for free** — overlay inherits the same `dark:placeholder-zinc-500` class the current textarea carries.

## 6. Scope decisions (IN and OUT)

**IN scope:**
- `<ActiveTypingOverlay>` component extracted to `components/smart-create/ActiveTypingOverlay.tsx` (single file, small).
- Wire it into the plain-textarea path in `IntentSection.tsx:701-720`.
- Looped caret shimmer + smooth user-typing fade (core UX gains).
- Optional `substringMatch` field on `ScenarioGuidance.schemaHints[n]` for phase-2 inline highlights; not required for every entry — backwards-compatible default behavior is no inline highlight.
- `prefers-reduced-motion` handling (overlay renders static, matches today).
- `feature_scenario_guidance_active_typing_overlay` flag, defaulted OFF; flip ON after first-50-creations sanity review.
- Jest unit tests for the overlay component + an integration test that the textarea path renders the overlay when the feature flag is on.

**OUT of scope:**
- RichComposer path (`feature_rich_composer_live=true`). That path is currently off in production. The Tiptap-based richer composer has its own Placeholder extension; per-char styling there would use ProseMirror decorations, a fundamentally different implementation. Phase-2 proposal after `feature_rich_composer_live` ships.
- Smart "specificity suggestions" based on the user's live input (separate proposal territory).
- Adding/rewriting exemplars. Data layer is 29/29 complete.
- Multi-line caret positioning edge cases under IME composition / RTL — follow-up if issues surface in post-deploy Playwright.

## 7. Implementation plan

### Phase A — overlay + looped caret (load-bearing piece)

1. `components/smart-create/ActiveTypingOverlay.tsx` — new component. Props: `{ typedText, userTextLength, cursorVisible, reducedMotion, className }`. Renders a `div` with the typed text + a `<span aria-hidden>▎</span>` at the end. Uses `tailwind-merge` to reuse the textarea's font/padding classes.
2. `IntentSection.tsx` — wrap the textarea in a `<div className="relative">`; mount the overlay above; clear the `placeholder` attribute when flag is on; add `bg-transparent` to the textarea; wire `cursorVisible = !isTyping && promptText.length === 0`.
3. CSS keyframes in `globals.css` or inline Tailwind arbitrary value — `@keyframes caret-shimmer`. Respect `motion-reduce:animate-none`.
4. Feature-flag gate in `lib/feature-flags.ts` (`feature_scenario_guidance_active_typing_overlay: false`).
5. Unit tests: overlay renders when empty, cursor visible when not typing, fades as `userTextLength` grows, reduced-motion variant disables the shimmer animation.

### Phase B — smooth user-typing fade

1. Derive overlay opacity from `userTextLength` across the first `FADE_START_CHARS = 24` characters.
2. Unit test: opacity at 0 chars = 1.0, at 12 chars ≈ 0.5, at 24+ chars = 0.
3. E2E verification: manually type in the composer on a scenario-loaded page; confirm smooth fade vs snap-off.

### Phase C — optional inline clarifier highlights

1. Extend `SchemaHint` type (in `lib/scenario-guidance.ts`) with optional `substringMatch?: string`.
2. Only scenarios that the researcher has annotated with substring matches render the inline highlight; the rest continue to show below-box chips unchanged.
3. When a substring match is present AND the typewriter has reached a char offset past the substring's end, apply a `border-b border-dotted border-brand-300/60` to that span in the overlay. Chips continue to work below-box.
4. Unit test: given a schemaHint with `substringMatch: "Stripe L5"`, the overlay renders a highlighted span around "Stripe L5" once the typewriter reaches that offset.
5. Seed: 2–3 scenarios get `substringMatch` annotations as a pilot (interview-prep/stripe, exam-prep/medical-step1, corporate-training/cybersecurity) to validate the shape. Remaining scenarios stay without the annotation — phase-2 rollout.

## 8. Acceptance criteria

- [ ] `ActiveTypingOverlay` component exists at `components/smart-create/ActiveTypingOverlay.tsx`; < 150 lines, single responsibility.
- [ ] IntentSection's plain-textarea path uses the overlay when `feature_scenario_guidance_active_typing_overlay` is on; falls back to the current placeholder behavior when off. Both paths render identically when the input is empty.
- [ ] Looped caret shimmer visible at the end of the typed exemplar after the typewriter completes; disabled under `prefers-reduced-motion`.
- [ ] User typing fades the overlay smoothly over the first ~24 characters; no snap.
- [ ] Schema-hint chip pulse (R2-v2) continues to work identically below the box.
- [ ] `alternate_shown` / `your_turn_shown` analytics continue to fire identically.
- [ ] Mobile 8-row cap still enforced when `isNarrowViewport`.
- [ ] Dark-mode parity maintained — overlay visible and legible under the `dark` class.
- [ ] Jest: overlay + integration tests added; full suite green.
- [ ] `tsc --noEmit` clean, `pnpm lint` no new warnings.
- [ ] **Post-deploy Playwright verification on beta.kitesforu.com** per CLAUDE.md rule 11: open `/create-smart` with an interview-prep scenario loaded, wait for typewriter to complete, verify caret shimmer visible; start typing and confirm smooth overlay fade.
- [ ] Phase C: at least 2 scenarios seeded with `substringMatch` annotations; inline-highlight test pins the contract.

## 9. Risk / rollback

| Risk | Likelihood | Mitigation | Rollback |
|---|---|---|---|
| Overlay + textarea drift in font/padding (visual jitter) | Medium | Reuse the textarea's className for the overlay typography; single Tailwind class set, applied to both. Snapshot-style jest test that renders empty state and asserts the overlay's text matches the coachedPlaceholder exactly. | Feature-flag off → revert to placeholder. |
| IME composition / RTL input misaligns the caret | Low | Caret element lives at `::after` of the last visible span, so it follows natural text flow including RTL. IME composition already handled by textarea itself; overlay only renders when user hasn't typed. | Feature-flag off; ship IME fix if needed. |
| Autofill / password-manager overlays interfere | Low | The composer isn't an auth field; browsers shouldn't trigger autofill. Test with Chrome, Safari, Firefox. | Feature-flag off. |
| Performance regression on long exemplars | Low | Overlay is static DOM; no animation loops for the text itself; only the caret element animates via CSS. | N/A — tuning is local. |
| User finds the shimmer distracting | Medium | Flag stays OFF in prod at first. Expand rollout after a 50-creation sanity review with `content_created` rate holding. If a regression appears, flip off. | One flag flip. |

Rollback strategy: `feature_scenario_guidance_active_typing_overlay` defaults to `false`. The existing `placeholder`-based path remains code-resident throughout and is the fallback branch; removing the feature is one flag flip.

## 10. CI/CD implications

- **Frontend-only.** No api, workers, schemas, or docs-repo changes (beyond this proposal + its eventual shiplog update).
- **Local CI** (per the 2026-04-22 local-ci thread): pnpm type-check, lint, test. New overlay tests will run.
- **GitHub CI** is billing-blocked (per the 2026-04-22 handoff); local CI is primary gate.
- **Deploy**: standard main-push Cloud Run redeploy.
- **Post-deploy**: Playwright verification on beta per CLAUDE.md rule 11. Runtime feature (animation + placeholder substitution) needs browser testing — `tsc` cannot validate motion or visual layering.

## 11. What NOT to do in this thread

- Don't merge the overlay refactor into the RichComposer (Tiptap) path in the same PR. The Tiptap path is behind a different flag, has a different extension model, and needs its own decorations-based implementation.
- Don't add new scenarios, clarifiers, or exemplars. Data layer is 29/29 complete via R2-v2.
- Don't refactor `runTypewriter` or `lib/typewriter.ts`. The overlay consumes its output — no upstream change needed.
- Don't move the chip-pulse/below-box rendering into the overlay. Chips stay below as redundant signal, especially important on mobile where the overlay may be partially obscured by keyboard.
- Don't add a real-LLM gate to this proposal. That's an R2-v2 optional AC and belongs in its own env-flagged CI job.

## 12. Ship log

- **2026-04-24 docs this proposal** — scope captured; awaits Codex audit before code per triangulation rule.
- *(next)* PR 1 frontend — `ActiveTypingOverlay.tsx` component + feature-flag gate + phase-A tests.
- *(next)* PR 2 frontend — phase-B smooth-fade + phase-C inline clarifier highlights + pilot scenarios seeded.
- *(next)* beta Playwright verification per CLAUDE.md rule 11.

## 13. Sources

- `kitesforu-frontend/components/smart-create/IntentSection.tsx:175, 701-720` — current placeholder-based rendering.
- `kitesforu-frontend/lib/typewriter.ts` — `runTypewriter` char-streaming implementation (unchanged by this proposal).
- `kitesforu-frontend/lib/feature-flags.ts:76, 149` — `feature_rich_composer_live` currently off (primary path is the plain textarea this proposal targets).
- `business/features/proposed/R2-scenario-guidance-v2-density-and-preemption.md:166` — the partial AC ("Placeholder feels 'active'") that explicitly deferred the looped shimmer pending this refactor.
- User standing follow-up (2026-04-20): "improve the scenario/type guidance UI so it feels more like active typing."
