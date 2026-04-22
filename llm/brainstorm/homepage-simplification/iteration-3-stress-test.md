# Iteration 3 — Stress-test the D+F+G+B composition

Four axes. For each, either the composition holds or we adjust.

---

## Axis 1 — Mobile vs desktop

**Desktop (1440×900)**: one hero promise + one input + one sample-row below the fold fits comfortably. No friction.

**Mobile (375×812)**: the layout-level `BottomNav` already has an elevated `+ Create` button in the middle (Agent 1 called this out — `BottomNav.tsx:167-168` — `-mt-4 bg-brand-500 ring-4`). On mobile, the shell already *promises creation* louder than the page can. This is a genuine conflict:

- **Option A**: hide the hero input on mobile; let the bottom-nav `+` be the single creation action. Hero becomes headline + CTA only.
- **Option B**: keep the hero input, but demote the bottom-nav `+` on the homepage route specifically (it stays elevated on other pages).
- **Option C**: redesign the bottom-nav so the `+` is not elevated on homepage — visually yielding to the page's own primary.

Decision: **Option A for v1**. Mobile homepage hero = headline + sample-audio preview + CTA. The bottom-nav `+` is the input affordance. This obeys the "if it's in the shell, don't put it on the homepage" rule *harder* on mobile where every pixel of duplication is felt.

Desktop keeps the hero input because the `+ Create` navbar button is a smaller affordance than the mobile `+` tab.

**Composition update**: signed-out/signed-in hero input is desktop-only; mobile shows a "Tap `+` to start" nudge if the user hasn't submitted anything yet.

---

## Axis 2 — Signed-in empty-state (0 items)

A fresh-signed-in user has nothing to "Continue." G (resume-first) has no content to surface. What does the hero show?

- Default: F (situation-first input). The dashboard gracefully degrades to the creation hero.
- Below-fold: a "First time? Try these" row of 3 starter prompts — e.g. "A 10-min explainer on Treasury bills" / "A 15-min horror story set in an abandoned lighthouse" / "Interview prep for Stripe L5 backend."

This solves the "empty-grid problem" Agent 2 flagged (Pattern D fails if the grid is empty). We never show an empty grid — we show starter prompts when there's no content.

**Composition update**: `HomeDashboard` becomes a state machine with three states:
1. `zero_items` — F hero + starter prompts (no "Continue" row)
2. `one_in_progress` — G hero (specific continue card) + F input below + "Recent" row
3. `many_items` — F hero + "Continue" row (specific items) + "Recent" row. No persona grid, no stat strip, no ValueCards.

---

## Axis 3 — The listener-only user

Consider a user who only wants to consume. They don't create; they open the app to find something to listen to. Does a creation-hero serve them?

- Signed-out listener-only: this persona is thin (they'd use Spotify). But if they exist, Candidate B (sample-row below the fold) gives them an entry point. They click a sample → it plays → they see they can make more like this → they sign up.
- Signed-in listener-only: the "Continue" row is literally for them. G is their hero.

The composition serves them via the below-fold sample row (signed-out) and the Continue row (signed-in). The hero headline must not assume they want to create.

**Composition update**: signed-out hero copy must acknowledge both halves. Agent 4's draft #1 ("What do you want to hear next?") threads this needle — it sounds like consumption ("hear") while still being an input (they type). Draft #3 is more explicit ("Audio that fits your situation") and covers both create and listen equally.

---

## Axis 4 — The interview-prep-only user

The current site's *actual* highest-performing promise is "Master your next interview." Does demoting it to a sub-shelf regress acquisition?

Two considerations:

1. **Evidence**: Agent 4 found no retention data. We don't actually know that the interview-prep-only audience is the biggest. The `SignedOutHero` *copy* says so, but the product's effort (60+ audio-gen PRs vs 30+ interview-prep PRs) says the investment is bigger on audio generation.
2. **Positioning theory**: a hero that promises "everything" regresses every audience. A hero that promises one thing risks the other. The question is whether the dual-wedge promise ("audio tuned to your situation") can *carry* interview-prep as a flagship specific example.

Decision: **yes, with a prominent Interview Prep tile below the fold.** The hero is situation-first; the first below-fold card is "Prepping for an interview? → /interview-prep". That tile owns the full current `SignedOutHero` copy ("Master your next interview", STAR, Elo, Mastery Trace). Interview-prep users land on the hero, see themselves in the first card, click, and continue to the current dedicated surface. No one is demoted — the current hero *moves*, it doesn't get cut.

**Composition update**: signed-out below-fold includes a dedicated "Interview Prep" card with the current hero copy, next to 2-3 other "situation" cards (Bedtime / Study / Podcast).

---

## Composition after iteration 3

**Signed-out homepage (`/` when `<SignedOut>`):**

```
[Navbar (shell)]
  Logo · Home · Search · + Create · Pricing · Sign In

[Hero — one viewport]
  "Audio tuned to your situation."
  Subhead: "Describe what you're preparing for, curious about, or
            falling asleep to. We turn it into a voice-first audio
            experience in under two minutes."
  Input: "Tell us what you want to hear" (+ mic)
  CTA: "Listen to a sample →" (anchors to the sample row below)

[Proof row — below the fold]
  "Hear what KitesForU produces"
  [sample 1: Interview Prep — 18 min] [sample 2: Bedtime — 8 min]
  [sample 3: Explainer — 14 min]      [sample 4: Horror — 24 min]
  (audio-preview tiles, tap to play in place)

[Situation cards — row of 3-4]
  "Prepping for an interview?" → /interview-prep (owns full Mastery Trace / STAR copy here)
  "Making bedtime stories?"     → /create-smart?scenario=bedtime
  "Turning notes into audio?"   → /create-smart
  "Studying for an exam?"       → /create-smart?scenario=exam-prep

[Pricing teaser + footer]
```

**Signed-in homepage (`/` when `<SignedIn>`):**

```
[Navbar + greeting bar, reduced height]

[Hero state machine]
  Zero items:    "What do you want to hear?" input + 3 starter prompts below
  In-progress:   "Continue: [specific item] · 4:21 left" [Resume button]
                 small "Or start new" input below
  Steady state:  "What do you want to hear?" input + "Continue" row

[Below the fold, max 2 sections]
  "Continue" (specific items, max 6)
  "Recent" (3 items max, with "See all → /library")

[Nothing else]
  No persona grid (moved to /create)
  No marketing ValueCards
  No BottomCTAs
  No CreatorStatsStrip (moved to /dashboard)
  No FirstVisitBanner
```

**Shell unchanged.** Navbar + BottomNav carry existing nav.

---

## What survives iteration 3

The composition holds against all 4 axes with small adjustments:
- Mobile: hide hero input, rely on shell `+` tab
- Empty signed-in: degrade to F with starter prompts
- Listener-only: sample-row below signed-out hero; Continue row signed-in
- Interview-prep-only: dedicated card below the situation-first hero owns the interview-prep promise

This is the shape iteration 4 makes concrete.
