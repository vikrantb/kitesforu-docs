# Iteration 4 — The winner, in detail

The homepage shape is now concrete. This iteration pins copy, layout, state transitions, and the cut list.

---

## The headline

**"Audio tuned to your situation."**

Why this wins:
- 5-second legible. A visitor reads it and knows the product makes audio, and it's personalized.
- Honest. Covers both halves of the real wedge (audio generation + situation-specific adaptation), not just the interview-prep slice.
- Does not exclude audiences the product actually serves (bedtime parents, horror listeners, exam crammers, teachers).
- No jargon. No proper nouns ("Mastery Trace"). No "B2B" or "executives."

Why not alternates:
- "What do you want to hear next?" — good but quietly tilts to consumption over creation.
- "Your situation. Professionally produced. In a minute." — three noun phrases; reads like a tagline, not a promise.
- "Audio that fits your situation." — nearly identical; "tuned to" has more craft connotation than "fits."

---

## The subhead

**"Describe what you're preparing for, curious about, or falling asleep to. We turn it into a voice-first audio experience in under two minutes."**

Why:
- The three verbs (preparing / curious / falling asleep) each map to a real shipped use case: interview prep / explainers / bedtime. No user has to wonder if they count.
- "Voice-first audio experience" is honest — it names the output modality. "In under two minutes" is a hard, real, impressive claim (53s + overhead per MEMORY).

---

## The CTA

Above-the-fold primary: **`Start with a sentence`** (or a voice-mic button with the same semantic).

Why:
- "Start with a sentence" is a direction, not a category. It tells the user *how* to begin without narrowing to a content type.
- The button leads to the input being focused. The input's placeholder cycles through real examples: "A 15-minute explainer on the Fed's private-credit report" / "A gentle bedtime story about a lost mitten" / "Stripe L5 backend interview prep in 2.5 weeks" (these are shipped scenario-guidance exemplars).

Not chosen:
- "Begin your first session" — corporate.
- "Try it free" — generic SaaS.
- "Listen to a sample" — tilts to consumption; put this as a secondary link.

---

## The secondary affordance (just below the headline)

A small linked phrase: **"Or listen to a sample →"** — scrolls to the proof row.

One visual, zero friction, serves the listener-only user.

---

## The signed-out page, in full

```
┌──────────────────────────────────────────────────────────────┐
│ [Logo]  Home  Search  + Create  Pricing    [Sign in]         │  Navbar (shell)
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                                                              │
│            Audio tuned to your situation.                    │
│                                                              │
│  Describe what you're preparing for, curious about, or       │
│  falling asleep to. We turn it into a voice-first audio      │
│  experience in under two minutes.                            │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │ Tell us what you want to hear…           🎙   ▶ │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│            Or listen to a sample →                           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  Hear what KitesForU produces                                │
│                                                              │
│  [Interview Prep — 18 min]  [Bedtime — 8 min]                │
│  [Explainer — 14 min]       [Horror anthology — 24 min]      │
│  (tap to preview, no sign-in required)                       │
├──────────────────────────────────────────────────────────────┤
│  Made for your situation                                     │
│                                                              │
│  [Prepping for an interview?]      [Making bedtime stories?] │
│  → /interview-prep                  → /create-smart?… bedtime│
│                                                              │
│  [Turning notes into audio?]       [Studying for an exam?]   │
│  → /create-smart                   → /create-smart?… exam    │
├──────────────────────────────────────────────────────────────┤
│  Pricing teaser (single card)                                │
├──────────────────────────────────────────────────────────────┤
│  Footer                                                       │
└──────────────────────────────────────────────────────────────┘
```

Four zones. Every pixel has a job. No mystery copy, no proper nouns, no menu-of-features.

## The signed-in page, in full

Three states — the `HomeDashboard` switches between them:

### State A: `zero_items` (fresh user)

```
┌──────────────────────────────────────────────────────────────┐
│ [Logo]  Home  Library  Search  + Create  Pricing [Avatar]    │  Navbar
├──────────────────────────────────────────────────────────────┤
│   Good morning, {name}.                                      │
│                                                              │
│            What do you want to hear?                         │
│  ┌──────────────────────────────────────────────────┐        │
│  │ Tell us what you're preparing for…       🎙  ▶  │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│   Starter ideas:                                             │
│   · A 15-min explainer on the Fed's private-credit report    │
│   · A gentle bedtime story about a lost mitten               │
│   · Stripe L5 backend interview prep in 2.5 weeks            │
├──────────────────────────────────────────────────────────────┤
│  (nothing below the fold until they have items)              │
└──────────────────────────────────────────────────────────────┘
```

### State B: `one_in_progress` (has an unfinished listen or a generating episode)

```
┌──────────────────────────────────────────────────────────────┐
│  Good morning, {name}.                                       │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │ 📻 Continue: The Midnight Line — Ep 3 of 5       │        │
│  │ 4:21 remaining                           [▶ Play]│        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│   Or start something new…                                    │
│  ┌──────────────────────────────────────────────────┐        │
│  │ What do you want to hear next?           🎙  ▶  │        │
│  └──────────────────────────────────────────────────┘        │
├──────────────────────────────────────────────────────────────┤
│   Recent                                                     │
│  [card] [card] [card]   See all → /library                  │
└──────────────────────────────────────────────────────────────┘
```

### State C: `many_items` (power user)

```
┌──────────────────────────────────────────────────────────────┐
│  Good morning, {name}.                                       │
│                                                              │
│            What do you want to hear next?                    │
│  ┌──────────────────────────────────────────────────┐        │
│  │ Tell us what you're preparing for…       🎙  ▶  │        │
│  └──────────────────────────────────────────────────┘        │
├──────────────────────────────────────────────────────────────┤
│   Continue                                                   │
│  [in-progress card × up to 6]                                │
├──────────────────────────────────────────────────────────────┤
│   Recent                                                     │
│  [card × 3]             See all → /library                  │
└──────────────────────────────────────────────────────────────┘
```

---

## The cut list (explicit)

Each of these is currently on the homepage. Each gets moved or deleted. No exceptions.

| Element | Current location | Action |
|---|---|---|
| `SignedOutHero.tsx` — "Master your next interview" H1 | Hero of signed-out | **Moves** to `/interview-prep` landing (promoted to the interview-prep tile on home) |
| `SignedOutHero.tsx` — "B2B Mastery Companion" eyebrow | Signed-out hero | **Deleted** |
| `SignedOutHero.tsx` — 3 flagship cards (Voice-First / Interactive / Custom Series) | Signed-out below-fold | **Moves** to `/interview-prep` landing |
| `SignedOutHero.tsx` — "Mastery Loop works" 3-step explainer | Signed-out below-fold | **Moves** to `/interview-prep` landing |
| `SignedOutHero.tsx` — 4 value tiles (STAR / Adaptive / Voice-first / Mastery Trace) | Signed-out below-fold | **Moves** to `/interview-prep` landing |
| `SignedOutHero.tsx` — "Ready for the room?" final CTA | Signed-out below-fold | **Deleted** (redundant with hero CTA) |
| `FirstVisitBanner.tsx` | Signed-in above-fold | **Deleted** (redundant with hero copy) |
| `HomeDashboard.tsx` — `CreatorStatsStrip` | Signed-in above-fold | **Moves** to `/dashboard` |
| `CentralChatLauncher.tsx` — "What do you want to create today?" H1 | Signed-in above-fold | **Rewritten** to "What do you want to hear?" (or "What do you want to hear next?" in state C) |
| `app/page.tsx:63-81` — Car Mode + Continue listening peer buttons | Signed-in above-fold | **Deleted**. Car Mode added to shell (navbar + bottom-nav). Continue listening replaced by specific-item card in state B. |
| `app/page.tsx:86-115` — "Pick a template instead" `<details>` | Signed-in below-fold | **Deleted**. Persona grid moves to `/create` (already exists there). |
| `PersonaSection` grid (8 personas × template chips) | Signed-in below-fold | **Deleted** from home; already exists on `/create`. |
| `HomeDashboard.tsx` — Quick Actions section | Signed-in below-fold | **Deleted**. Persona-targeted shortcuts belong on `/create` or in the input placeholder rotation. |
| `HomeDashboard.tsx` — Recommended section | Signed-in below-fold | **Deleted**. Static-per-persona "recommendations" are a lie; earn it back when wired to real history. |
| `app/page.tsx:118-141` — 4 bottom ValueCards | Signed-in below-fold | **Deleted**. Marketing tiles for signed-in users is the cop-out. |
| `HomeCTAs.tsx` — `BottomCTAs` "Begin your next session" | Signed-in below-fold | **Deleted**. Third duplicate CTA to `/create-smart`. |
| Legacy `HeroSection.tsx` (feature flag OFF path) | Signed-in | **Deleted**. Remove the flag, remove the legacy branch. One architecture. |

After cuts:
- Signed-out file: one hero + one proof row + one situation-cards row.
- Signed-in file: one hero (state machine) + one Continue row + one Recent row.
- Every deleted element either moved to a real destination (`/interview-prep`, `/dashboard`, `/create`) or was redundant with the shell or with itself.

---

## The ONE metric to watch post-ship

**Creation-attempt rate for first-time signed-in users within 5 minutes of first visit.**

If the simplification works, this goes up. If the simplification regresses an audience (e.g. interview-prep users can't find their flow), total signups stay flat but the content-type distribution shifts — which we can watch.

Secondary instrumentation (reuse the existing `lib/analytics.ts` dispatch):
- `home_hero_input_submit` (the main primary action)
- `home_sample_played` (signed-out below-fold proof row)
- `home_continue_resumed` (signed-in state B)
- `home_situation_card_clicked` (signed-out below-fold situation tiles)
- `home_starter_prompt_selected` (signed-in state A)

---

## What iteration 5 owes

Iteration 4 pins the shape. Iteration 5 names the 3-PR implementation sequence.
