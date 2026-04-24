# Iteration 5 — Implementation priority

The winning shape from iteration 4 ships as three sequential PRs. Each is independently shippable with graceful fallback if the next PR slips.

---

## Status tracker — 2026-04-24

| PR | Status | Detail |
|---|---|---|
| PR #1 | **Live in prod** | `feature_home_signed_out_v2: true` in `lib/feature-flags.ts`. `SignedOutHeroV2.tsx` renders for all signed-out visitors. Frontend #591 pins the copy + redirect map + negative contract (21 Jest assertions) so a future editor can't silently regress the v1 positioning. |
| PR #2 | **Built + test-gated; awaiting flag flip** | Frontend #592 ships `HomeDashboardV2.tsx` + `useHomeDashboardState` hook behind `feature_home_signed_in_v2` (default OFF). Three-state machine: `zero_items` / `one_in_progress` / `many_items`. 34 new tests (19 state-machine unit + 15 render) pin the state→layout contract. Full suite 1073/1073 pass. Owner can flip the flag on beta to see the simplified shape; cleanup of legacy components ships as follow-up after flag flip. |
| PR #3 | blocked on PR #2 flag flip + cleanup | Car Mode to shell + deletion of legacy `HomeDashboard` / `CreatorStatsStrip` / `QuickActionsSection` / persona `<details>` grid / `ValueCards` / `BottomCTAs` / removal of `feature_home_unified_chat_live`. |

---

## PR #1 — The signed-out landing: cut, clarify, commit

**Scope**: replace `SignedOutHero.tsx` with the shape from iteration 4.

**Changes**:
- Rewrite `components/home/SignedOutHero.tsx` as 4 zones: hero (headline + input + sub-CTA) / sample-row / situation-cards / footer.
- Move the interview-prep hero copy (H1, 3 flagship cards, Mastery Loop, STAR/Elo/Trace value tiles) to `app/interview-prep/page.tsx` as the landing of that route. A user who clicks the "Prepping for an interview?" situation card on home arrives at the current full interview-prep pitch.
- Delete the "Ready for the room?" final CTA, the "B2B Mastery Companion" eyebrow, the "engineered for executives" language.
- New copy (final): headline / subhead / CTA / situation-card captions as written in iteration-4.

**Complexity**: single component rewrite + one new sub-component (`SampleRow`) + the interview-prep landing gets the moved content. ~250 LOC delta net-negative (the deletions outweigh the additions).

**Test plan**:
- Playwright: anonymous visitor lands on `/`, sees the new headline, clicks the situation card for interview prep, lands on `/interview-prep`, sees the preserved Mastery Loop content.
- Unit: copy assertions on the new `SignedOutHero` (fails loudly if anyone re-introduces "B2B" or "executives").

**Ships independently**: yes. Signed-in users unaffected.

**Risk**: regresses acquisition for a visitor who specifically landed expecting the interview-prep hero. Mitigation: the interview-prep situation card is prominent; URL-deep-links to `/interview-prep` from marketing continue to work.

---

## PR #2 — The signed-in dashboard: state machine + cut the marketing

**Scope**: rewrite `HomeDashboard.tsx` around the 3-state machine (zero_items / one_in_progress / many_items).

**Changes**:
- Add `useHomeDashboardState()` hook that reads the same data sources `HomeDashboard.tsx` already reads and returns one of the three states.
- Rewrite `HomeDashboard.tsx` to render based on state:
  - A: `CentralChatLauncher` + 3 starter-prompt chips (pulled from scenario-guidance exemplars — they already exist in `lib/scenario-guidance.ts`).
  - B: specific "Continue: {name}" card + compact `CentralChatLauncher` below.
  - C: `CentralChatLauncher` hero + Continue row + Recent row.
- Delete from render tree: `FirstVisitBanner`, `CreatorStatsStrip`, the peer Car Mode / Continue-listening buttons (`app/page.tsx:63-81`), the `<details>` persona grid (`app/page.tsx:86-115`), the 4 ValueCards (`app/page.tsx:118-141`), `BottomCTAs`, `QuickActionsSection`, `RecommendedSection`.
- Remove the `feature_home_unified_chat_live` feature flag — pick the unified shape, commit, delete the legacy `HeroSection.tsx` branch.
- `CentralChatLauncher` H1 changes from "What do you want to **create** today?" to "What do you want to **hear**?" (or "…next?" in state C).

**Complexity**: the biggest PR. Touches `app/page.tsx`, `HomeDashboard.tsx`, deletes 6+ components, removes a feature flag. ~1500 LOC delta net-negative.

**Test plan**:
- Playwright: sign in as a new user (0 items), see state A. Generate one item, see state B with specific continue. Generate multiple, see state C.
- Unit: state machine has 3 test cases (zero / one / many) with explicit expected components.
- Analytics: new events (`home_hero_input_submit`, `home_continue_resumed`, `home_starter_prompt_selected`) fire per iteration-4 spec.

**Ships independently**: yes, but only after PR #1 (because `app/page.tsx` is edited in both).

**Risk**: biggest behavior delta. Regression risk on the returning-power-user flow. Mitigation: keep everything that moved (persona grid to `/create`, stats strip to `/dashboard`) reachable via the shell; add temporary "Moved to X" redirects for 30 days.

---

## PR #3 — Add Car Mode to the shell + polish

**Scope**: put Car Mode in the navbar so it's no longer homepage-dependent.

**Changes**:
- Add "Car Mode" entry to `Navbar.tsx` (desktop) — small button with the 🚗 icon, next to `Search`.
- Add Car Mode tab option to `BottomNav.tsx` (mobile) — replace the "Search" tab if real-estate is tight (the `⌘K` palette is already global).
- Delete the residual Car Mode peer button code that PR #2 left behind (it's referenced in a few other places).
- Add the "Moved from home" quick redirects: visits to `/?view=dashboard` land on `/dashboard`; visits to `/?view=templates` redirect to `/create`. These are a 30-day bridge, then deleted.
- `/dashboard` gets the `CreatorStatsStrip` ported from PR #2's cut list — it earns its home as a dashboard page, not as a homepage above-fold.
- Run a final "moved to done/" pass on `R1-phase2-dark-mode-polish-followup.md` and close out the homepage simplification proposal.

**Complexity**: smallest. ~300 LOC delta.

**Test plan**:
- Playwright: Car Mode reachable from any page via navbar; bottom-nav `Car Mode` tab opens `/car-mode` on mobile.
- Redirect: `/?view=dashboard` → `/dashboard`, no flicker.
- Manual: spend 10 minutes clicking around — every "I used to reach X from home" route has a new canonical home.

**Ships independently**: only after PR #2 lands (Car Mode on homepage is still rendering in PR #2; PR #3 is the cleanup).

**Risk**: bottom-nav churn has taste consequences. If "Search" vs "Car Mode" is contentious, ship Car Mode only in the desktop navbar for v1 and defer the mobile tab swap.

---

## Shipping order + dependency

```
PR #1 (signed-out rewrite) ───┐
                              ├──→ both merged independently
PR #2 (signed-in state machine) ──┼──→ PR #3 (Car Mode in shell + polish)
                                  │
                                  └── requires PR #2 on main
```

PR #1 and PR #2 touch `app/page.tsx` but in disjoint areas (one modifies the `<SignedOut>` branch, the other modifies the `<SignedIn>` branch). They can ship in either order; merge conflicts are trivial.

---

## Estimated velocity

Per this session's proven pattern (research agents in parallel, then focused implementation):

- PR #1: ~1 day (copy rewrites are the biggest task; the component surgery is small).
- PR #2: ~2 days (state machine + deletion sweep; bulk of the complexity).
- PR #3: ~0.5 day.

Total: ~3.5 days of focused work, shippable as three merges inside a week.

---

## Rollback plan

Each PR ships with a feature flag for the first 72 hours:
- PR #1: `feature_home_signed_out_v2` — off flips back to current `SignedOutHero` component.
- PR #2: `feature_home_signed_in_v2` — off flips back to current `HomeDashboard` + legacy render tree.
- PR #3: no flag needed (cleanup only).

Flags are removed 72 hours post-merge if no regression signal. Matches the team's existing `feature_home_unified_chat_live` pattern (which is removed as part of PR #2).

---

## What we're watching post-ship

Primary metric: **creation-attempt rate for first-time signed-in users within 5 minutes of first visit**. Tracked via `home_hero_input_submit` + user-cohort join.

Target: a measurable lift. If flat, the simplification didn't regress but didn't help; we reconsider. If down, we roll back via the flag.

Secondary signals:
- `home_sample_played` / `home_situation_card_clicked` on signed-out (engagement with the new proof row)
- `home_continue_resumed` on signed-in (state B is doing its job)
- Time-to-first-generation for new signups (should drop — fewer decisions to make)

---

## What this brainstorm does NOT solve

Out of scope for the homepage simplification thread:
- `/create-smart` improvements — that surface has its own deferred thread (scenario-guidance v2 is the active one).
- The "too many content types" breadth question — the homepage simplification narrows to one honest positioning but doesn't decide whether to drop k12-lesson / university-lecture / corporate-training from the product.
- Pricing surface redesign — pricing teaser on home gets a single tile; `/pricing` overhaul is separate.
- Player surface — already solid after R1-p2.

These get their own threads if they need them.

---

## The one sentence this brainstorm converges on

> **The homepage should tell one honest promise — "Audio tuned to your situation" — with one input, one proof row, one situation grid, and a signed-in dashboard that does exactly one of three things depending on user state.** Every other block on the current homepage either moves to a surface that already exists (`/create`, `/dashboard`, `/interview-prep`) or is deleted.

Everything in this brainstorm was in service of that sentence.
