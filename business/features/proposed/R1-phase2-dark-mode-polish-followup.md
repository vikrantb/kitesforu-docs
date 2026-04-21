# R1 Phase 2 D4 — Dark Mode Polish Follow-Up

**Status**: PROPOSED
**Priority**: P1 (user-reported; several high-traffic surfaces are visibly broken)
**Effort**: 1–2 weeks, batched into ~8 PRs
**Origin**: User feedback 2026-04-19: "some of the ux is not good for dark mode"

## Context

The original R1 P2 D4 pass landed ~20 PRs covering ~68 components with the convention `light-class dark:zinc-equivalent`, brand/semantic tints using `/10..30` alpha variants. A follow-up audit found that several high-traffic surfaces received zero dark-mode variants, and a few systemic gaps (Clerk auth modals, SVG literal strokes, `shadow-*` without dark counterparts) exist across the app.

This document tracks what the audit found and the planned PR sequence to close it.

## Audit methodology

1. Grepped `app/`, `components/`, and the shared UI library for light-only Tailwind classes (`bg-white`, `bg-gray-*`, `bg-slate-*`, `text-gray-*`, `border-*`, `hover:bg-*`, etc.) that lacked a corresponding `dark:` sibling.
2. Counted light-only vs `dark:` hits per file to surface coverage gaps.
3. Hand-verified that always-dark components (Car Mode / Drive, QuickQuestionOverlay, IntroPlaybackOverlay, CourseDebugPanel) remain intentionally dark-only and are NOT flagged as regressions.

## Punch list

### HIGH — broken in dark mode, users will notice

*Verified 2026-04-21: most HIGH items turned out to be already shipped in prior dark-mode passes, just never marked. Re-audit brought the remaining real gaps down to two (QR rectangle + globals.css utilities).*

- [x] **Clerk auth modals** — shipped via `components/ThemedClerkProvider.tsx`: reads the resolved theme via `useTheme` and pipes a zinc/brand `appearance.variables` map into `ClerkProvider` (zinc-950 background, zinc-50 text, brand-500 accent). Chosen over `@clerk/themes` baseTheme to keep palette consistent with Tailwind tokens without the extra dependency.
- [x] **Interview prep hub** — `app/interview-prep/hub/page.tsx` now carries 41 `dark:` variants across cards, filter pills, empty state, CTAs (re-audit 2026-04-21).
- [x] **Activity directory sweep** — 6 of 8 files now carry dark variants (ActivityCard=21, ActiveNowSection=11, FilterToolbar=16, SeriesCard=16, CompactAudioCard=4, ContentSection=4). `ActivityStats.tsx` was the last remaining literal-class file — fixed in the same PR as this doc update. `EmptyState.tsx` intentionally uses semantic tokens (`bg-card`, `text-foreground`, `border-border`) which are theme-aware by design; no `dark:` prefix needed.
- [x] **Public share route — courses** — `app/courses/shared/[token]/page.tsx` now carries 36 `dark:` variants (re-audit 2026-04-21).
- [x] **Public share route — classes** — `app/classes/join/[shareToken]/page.tsx` now carries 30 `dark:` variants (re-audit 2026-04-21).
- [x] **Classes explore/library** — `app/classes/library/page.tsx` now carries 17 `dark:` variants (re-audit 2026-04-21).
- [x] **Legacy podcast create form** — `app/courses/create/page.tsx` now carries 26 `dark:` variants (re-audit 2026-04-21).
- [x] **Signed-in home bottom CTA + ValueCards** — `app/page.tsx:85-134` now carries 6 `dark:` variants including the brand-gradient section + the four ValueCards (re-audit 2026-04-21).
- [x] **Course progress ring stroke** — `components/courses/detail/CourseHeader.tsx` no longer contains `stroke="#e5e7eb"`; the ring uses theme-aware tokens (re-audit 2026-04-21).
- [ ] **ShareContentModal QR code white rectangle** — `components/ShareContentModal.tsx:681`. QR codes need white backing for scannability, but a bright white block on zinc-950 looks like a bug. Fix: keep `bg-white`, but add a padded container with `dark:bg-zinc-200` or a subtle border/ring so the white block reads as an asset, not a glitch.
- [ ] **globals.css `@apply` utilities** — `app/globals.css:214-266` has four light-only utility classes (`.form-input-elegant`, `.form-label-elegant`, `.card-elegant`, `.btn-secondary-elegant`) with no `.dark` overrides. Verify references; either delete if unused or add `.dark` rules.

### MEDIUM — functional but jarring

- [x] **Top-level error boundary + 404** — `app/not-found.tsx` now carries 3 `dark:` variants, `app/error.tsx` now carries 6 (re-audit 2026-04-21).
- [x] **Activity error boundary** — `app/activity/error.tsx` now carries 2 `dark:` variants (re-audit 2026-04-21).
- [ ] **Create-smart hero** — `app/create-smart/page.tsx:138,144,151,241`. The primary creation flow's `<h1>`, tagline, and "Advanced options" underline link lack dark variants on what is otherwise a dark-aware page.
- [ ] **CourseHeader residual pockets** — `components/courses/detail/CourseHeader.tsx:94,105,138,166,223,247`. Action buttons, separator bullets `text-gray-300`, metadata line lack dark variants in an otherwise dark-aware file.
- [x] **FullPlayer playback-rate button** — `components/AudioPlayer/FullPlayer.tsx:252` now carries `text-gray-600 dark:text-zinc-300` + dark hover states (re-audit 2026-04-21).
- [x] **BottomNav sign-in label** — `components/BottomNav.tsx:134` now carries `text-gray-500 dark:text-zinc-400` (re-audit 2026-04-21).
- [ ] **Shadow system (systemic)** — 77 `shadow-lg`/`shadow-xl`/`shadow-2xl` occurrences across 47 files, **zero** `dark:shadow-*` anywhere. Black translucent shadows on dark surfaces are nearly invisible; bright-styled shadows look muddy. Design-system decision: either add `dark:shadow-black/40` pattern, or replace with `dark:shadow-none dark:ring-1 dark:ring-white/5`. Apply to top ~20 modal / elevated components as one PR.
- [ ] **Progress page residual pockets** — `app/progress/[jobId]/page.tsx` — 27 light vs 35 dark. Spot-check sweep.
- [ ] **SignedOutHero forced-dark ambiguity** — `components/home/SignedOutHero.tsx` uses `bg-white/5`/`bg-white/10` opacity variants. If it is intentionally always-dark, make it explicit with a `data-theme="dark"` wrapper.

### LOW — polish / consistency

- [ ] **Voice input recording dot opacity** — `components/ui/voice-input.tsx:184,185,336,337` use literal `bg-white opacity-75`. Consider `bg-white/90` for consistency.
- [ ] **Home hero gradient text tuning** — `app/page.tsx:43` gradient endpoints are tuned for light background. Consider `dark:from-brand-400 dark:to-brand-300` for better contrast on zinc-950.
- [ ] **Onboarding stragglers** — `app/onboarding/page.tsx` has 15 light vs 16 dark — near-parity, worth a once-over since it's a first-run surface.
- [ ] **FilterToolbar sort chevron** — `components/activity/FilterToolbar.tsx:319` inlines a data URL SVG with hardcoded stroke `%239ca3af`. Replace with a lucide `<ChevronDown>` + `text-zinc-400` class.
- [ ] **Scrollbars** — `app/globals.css` has no `::-webkit-scrollbar` rules. Default browser scrollbars on Linux/Chrome can look bright in dark mode. Add dark-aware scrollbar rules.

## Recommended PR sequence

1. **Infra (high leverage)** — Clerk `appearance` wiring. Fixes every auth surface in one shot.
2. **Interview prep hub** — single large file, ~100 class additions.
3. **Activity directory sweep** — 6 files, one systemic dark pass.
4. **Public share routes** — `courses/shared/[token]` + `classes/join/[shareToken]`. Decide theme policy first.
5. **Error boundaries + 404** — small fixups to `not-found.tsx`, `error.tsx`, `activity/error.tsx`.
6. **Home signed-in CTA + ValueCards** — targeted fix to `app/page.tsx`.
7. **Misc targeted fixes** — CourseHeader SVG stroke, FullPlayer:252, BottomNav:134, create-smart hero text, ShareContentModal QR backing.
8. **Shadow system (design decision)** — add `dark:shadow-*` pattern as system-wide convention.
9. **Legacy `courses/create` decision** — redirect or dark-pass.
10. **LOW-tier polish sweep** — voice-input opacity, home gradient tuning, onboarding, scrollbars, FilterToolbar chevron.

## Acceptance criteria

- [ ] Every route in the signed-in navbar renders correctly in dark mode (library, interview prep hub, create, classes library, courses, activity).
- [ ] Public share routes either render correctly in dark mode or are intentionally forced-light with a `data-theme` marker.
- [ ] Clerk auth modals follow the active theme.
- [ ] Error and 404 pages render correctly in dark mode.
- [ ] Audit re-run finds no new high-severity light-only surfaces.
- [ ] Always-dark components (Drive, QuickQuestionOverlay, etc.) remain intentional and marked.

## Out of scope

- Full WCAG AA audit via automated tooling (deferred to a separate accessibility initiative).
- RTL / i18n CSS foundations (tracked under R3 D4).
- Redesign of forced-dark surfaces (Drive, car-mode) — those are intentional.
