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

- [ ] **Clerk auth modals** — `app/layout.tsx` wraps `<ClerkProvider>` without an `appearance` prop. Sign-in / sign-up / user-button modals render fully white on top of a dark shell. Fix: import `dark` from `@clerk/themes` and bind `appearance={{ baseTheme: resolvedTheme === 'dark' ? dark : undefined }}` via a client wrapper using `next-themes`.
- [ ] **Interview prep hub** — `app/interview-prep/hub/page.tsx` has ~100 light classes and zero `dark:` variants. Every card, filter pill, empty state, pagination border, and CTA is hard-coded light; the sticky backdrop (`bg-white/85`) and executive Badges will glow against a dark page. Needs a full pass-through dark pass.
- [ ] **Activity directory sweep** — `components/activity/` (6 files: `ActivityCard.tsx`, `FilterToolbar.tsx`, `SeriesCard.tsx`, `ActiveNowSection.tsx`, `CompactAudioCard.tsx`, `ContentSection.tsx`) are completely undarkened. One systemic PR.
- [ ] **Public share route — courses** — `app/courses/shared/[token]/page.tsx` has 36 light classes, only 3 `dark:` hits. External invitees on dark OS see a broken view. Decide theme policy: dark-aware or force `data-theme="light"` on the root.
- [ ] **Public share route — classes** — `app/classes/join/[shareToken]/page.tsx` has 25 light, 0 `dark:`. Same fix direction as the course share route.
- [ ] **Classes explore/library** — `app/classes/library/page.tsx` has 14 light, 0 `dark:`. Search input, filter selects, empty state, and all course cards are light-only.
- [ ] **Legacy podcast create form** — `app/courses/create/page.tsx` has 20 light, 0 `dark:`. Style selector cards, persona selector cards, summary rows all light-only. Decide: dark-pass or redirect to `/create-smart` (matching the existing `/create` + `/create2` redirect pattern).
- [ ] **Signed-in home bottom CTA + ValueCards** — `app/page.tsx:85-134`. The `<section>` with `bg-gradient-to-br from-brand-50 to-brand-100` and the four ValueCards have no dark variants. Fix: `dark:from-brand-500/10 dark:to-brand-500/5 dark:border-brand-500/20` and `dark:bg-zinc-900/60 dark:border-zinc-800 dark:text-zinc-100` on ValueCards.
- [ ] **Course progress ring stroke** — `components/courses/detail/CourseHeader.tsx:201` uses hardcoded `stroke="#e5e7eb"` (gray-200). The ring track disappears on dark shell. Fix: `stroke="currentColor"` with `className="text-gray-200 dark:text-zinc-800"` on the SVG.
- [ ] **ShareContentModal QR code white rectangle** — `components/ShareContentModal.tsx:681`. QR codes need white backing for scannability, but a bright white block on zinc-950 looks like a bug. Fix: keep `bg-white`, but add a padded container with `dark:bg-zinc-200` or a subtle border/ring so the white block reads as an asset, not a glitch.
- [ ] **globals.css `@apply` utilities** — `app/globals.css:214-266` has four light-only utility classes (`.form-input-elegant`, `.form-label-elegant`, `.card-elegant`, `.btn-secondary-elegant`) with no `.dark` overrides. Verify references; either delete if unused or add `.dark` rules.

### MEDIUM — functional but jarring

- [ ] **Top-level error boundary + 404** — `app/not-found.tsx` (2 light, 0 dark) and `app/error.tsx` (3 light, 0 dark). Users hitting errors in dark mode see white-text-on-whiter-background.
- [ ] **Activity error boundary** — `app/activity/error.tsx` (2 light, 0 dark). Same fix.
- [ ] **Create-smart hero** — `app/create-smart/page.tsx:138,144,151,241`. The primary creation flow's `<h1>`, tagline, and "Advanced options" underline link lack dark variants on what is otherwise a dark-aware page.
- [ ] **CourseHeader residual pockets** — `components/courses/detail/CourseHeader.tsx:94,105,138,166,223,247`. Action buttons, separator bullets `text-gray-300`, metadata line lack dark variants in an otherwise dark-aware file.
- [ ] **FullPlayer playback-rate button** — `components/AudioPlayer/FullPlayer.tsx:252`. The single light-only control in an otherwise-darkened player.
- [ ] **BottomNav sign-in label** — `components/BottomNav.tsx:134`. "Sign in" label's `text-gray-500` lacks a dark variant; sibling labels have one.
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
