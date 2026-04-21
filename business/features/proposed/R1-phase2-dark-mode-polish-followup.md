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
- [x] **ShareContentModal QR code white rectangle** — `components/ShareContentModal.tsx` line 701 ships `bg-white rounded-lg p-3 ring-1 ring-transparent dark:ring-zinc-700 shadow-sm dark:shadow-lg dark:shadow-black/40`. Exactly the "keep bg-white + subtle dark ring" fix the proposal recommended. Re-audit 2026-04-22.
- [x] **globals.css `@apply` utilities** — all four classes (`.form-input-elegant` L215-224, `.form-label-elegant` L226-228, `.card-elegant` L232-240, `.btn-secondary-elegant` L264-274) carry full `dark:*` variants. Re-audit 2026-04-22 verified directly in globals.css.

### MEDIUM — functional but jarring

- [x] **Top-level error boundary + 404** — `app/not-found.tsx` now carries 3 `dark:` variants, `app/error.tsx` now carries 6 (re-audit 2026-04-21).
- [x] **Activity error boundary** — `app/activity/error.tsx` now carries 2 `dark:` variants (re-audit 2026-04-21).
- [x] **Create-smart hero** — re-audit 2026-04-22: lines 138/144/151 already carry dark variants (`text-gray-900 dark:text-zinc-100` / `from-brand-500 to-brand-400 dark:from-brand-400 dark:to-brand-300` / `text-gray-400 dark:text-zinc-500`). The proposal line numbers drifted and the work was silently done.
- [x] **CourseHeader residual pockets** — frontend PR #507 (2026-04-22) added dark variants to the 5 real remaining gaps: L101 Drive action button, L112 Debug action button, L187 back button, L215 first separator bullet, L272 "N/M episodes" count. Mirrored the Play-All button pattern (L155) which was already dark-aware.
- [x] **FullPlayer playback-rate button** — `components/AudioPlayer/FullPlayer.tsx:252` now carries `text-gray-600 dark:text-zinc-300` + dark hover states (re-audit 2026-04-21).
- [x] **BottomNav sign-in label** — `components/BottomNav.tsx:134` now carries `text-gray-500 dark:text-zinc-400` (re-audit 2026-04-21).
- [ ] **Shadow system (systemic)** — design decision resolved: `dark:shadow-black/40` pattern (matching globals.css `.card-elegant` + `components/voice/MasteryDebugOverlay` `shadow-black/50` precedent). Frontend PR #509 (2026-04-22) applied to the 8 highest-traffic `shadow-2xl` elevated surfaces (DistributionModal, ShareModal, CommandPalette, CelebrationOverlay, PublishModal, FullPlayer, SpeakerSlots, ShareContentModal). Remaining `shadow-lg` / `shadow-xl` on less-elevated surfaces (cards, buttons) still to sweep — track as a follow-up if dark-mode testing surfaces new gaps there.
- [ ] **Progress page residual pockets** — still [ ], not re-audited this pass.
- [x] **SignedOutHero forced-dark ambiguity** — re-audit 2026-04-22: the current file uses `text-gray-500 dark:text-zinc-400`, `text-gray-900 dark:text-zinc-100`, `text-brand-600 dark:text-brand-400` throughout — no `bg-white/5` / `bg-white/10` ambiguity remaining. The component respects the active theme like every other dark-aware page. Line numbers the proposal referenced no longer exist; the refactor that removed them also fixed the theme contract.

### LOW — polish / consistency

- [ ] **Voice input recording dot opacity** — re-audit 2026-04-22: `bg-white opacity-75` and `bg-white/90` are functionally equivalent. Dropping from backlog as cosmetic-only with no real dark-mode bug. If a future design pass wants aesthetic consistency, can be picked up then.
- [x] **Home hero gradient text tuning** — re-audit 2026-04-22: `app/page.tsx:163` (line number drifted from proposal's :43) already ships `bg-gradient-to-r from-brand-500 to-brand-400 dark:from-brand-400 dark:to-brand-300 bg-clip-text text-transparent` — exactly the fix the proposal recommended.
- [x] **Onboarding stragglers** — re-audit 2026-04-22: `app/onboarding/page.tsx` grep for light-only `text-gray-*` / `bg-gray-*` / `border-gray-*` without a matching `dark:*` returns zero hits. Every step renders correctly in dark mode. Silently shipped.
- [x] **FilterToolbar sort chevron** — frontend PR #507 (2026-04-22) replaced the hardcoded data-URL SVG with an overlay `<svg>` using `currentColor` and `text-gray-400 dark:text-zinc-500` matching the adjacent `SortIcon`. Native chevron stays hidden via `appearance-none`.
- [x] **Scrollbars** — frontend PR #508 (2026-04-22) added `html.dark`-scoped rules in globals.css using Tailwind's `theme()` function. Firefox gets `scrollbar-color` standard property; Chromium gets webkit pseudo-elements (track zinc-900, thumb zinc-700 with 3px zinc-900 padding border, hover zinc-600). Light-mode behavior unchanged.

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
