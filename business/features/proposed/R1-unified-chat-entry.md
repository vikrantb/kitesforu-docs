# R1 — Unified Chat as the Central Entry

**Status**: PROPOSED
**Priority**: P1
**Effort**: ~1 week (frontend routing + landing polish)
**Origin**: 2026-04-19 product-owner note after live drive-test of Car Mode Q&A.

## Problem

Today the home page scatters creation entry points across many tiles, quick-action buttons, and persona cards (Pick Your Path grid, Ideas to Audio, Study Anywhere, etc.). Each tile routes to its own specialized creation form (`/create-smart`, `/courses/create`, `/classes/create`, `/interview-prep/create`, `/car-mode`). Users have to decide *where* before they decide *what*.

The "general-purpose text box" that used to sit in the middle of the page — "say anything and we figure out the rest" — was the simplest, most powerful entry point. It has been de-emphasized as the persona grid grew.

> **Product-owner note (2026-04-19):**
> The general-purpose text box in middle that where people used to say something and it directly takes to chat thing was super powerful. I think we need that central and unified experience for most of the stuff. Only single experience where we take to that chat screen and then from there we take them to whatever specialized page when they click Create. But Car Mode is Car Mode — that option always should exist.

## Proposal

Collapse the home page's many creation entry points into **one single chat-style input** front-and-center. Every typed / voice request lands in the chat, and the chat is where the user decides whether it becomes a podcast, a course, a class, a writeup, or an interview-prep session. Car Mode gets its own dedicated always-visible button (it is a distinct consumption mode, not a creation flow).

### Goals

- One input. One chat. One "let's figure this out together" surface.
- Specialized creation pages (`/courses/create`, `/classes/create`, etc.) still exist but are reached only via the **Create** CTA on the chat screen — the chat decides scope first, then routes.
- Voice-first: the central input has a prominent mic so voice input is the primary path on mobile.
- Car Mode stays as a top-level destination with its own large, recognizable button.

### Non-goals

- Not removing the persona grid entirely. Keep it below the fold as a secondary entry for returning users who know exactly what template they want.
- Not changing the chat backend or the Smart Create planner logic. This is purely routing + landing page surface.

## Experience

### Signed-in home page (new layout)

```
┌──────────────────────────────────────────────────────────┐
│                  Good afternoon, Test.                   │
│              What do you want to create?                 │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Say or type anything — I'll figure out the rest.   │  │
│  │                                          [🎤 Mic]  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│       ─── or ───                                         │
│                                                          │
│   [  🚗 Drive (Car Mode)  ]   [  ▶︎ Continue listening ]  │
│                                                          │
│   ─── Pick a Template (secondary) ───                    │
│   [persona grid, collapsed by default, expand on click]  │
└──────────────────────────────────────────────────────────┘
```

### Flow

1. User types (or taps mic and speaks) a topic or intent into the central input.
2. Submit routes to `/create-smart` with the text prefilled and the chat auto-starts.
3. The chat planner (existing `useSmartCreateChat`) figures out what kind of artifact the user wants — podcast, course, class, writeup, interview prep, or Car Mode — and proposes a plan.
4. When the user clicks **Create** in the chat, they are routed to the specialized creation page (or, if it's Car Mode, directly into `/car-mode`).

### Car Mode as a peer entry

Car Mode is not a creation flow — it's a consumption mode with a live generator underneath. It deserves its own visible button on the home page, NOT a persona card buried in a grid. Placement: next to the central input, always visible, with a distinct colour / icon (existing orange steering-wheel icon).

### Signed-out home page

Same shape: a hero + the central input (disabled / sign-in-gated) + "Try Drive mode" CTA. Reinforces the same simple mental model before the user even signs up.

## Implementation

### Frontend (primary surface area)

- `app/page.tsx` — replace the current multi-persona home layout with:
  - Hero
  - Central chat input (new component `components/home/CentralChatLauncher.tsx` — text + mic + submit, uses `VoiceInput` primitive)
  - Car Mode button
  - Persona grid collapsed into an `<details>` or lazy-loaded section below the fold
- `components/home/CentralChatLauncher.tsx` (NEW) — owns the input state, integrates `VoiceInput`, pushes to `/create-smart?initial_text=...` on submit.
- `app/create-smart/page.tsx` — ensure `?initial_text=` is already honoured (it already is via the hook); add a small banner on landing that says "we'll help you figure out the rest" to set the expectation.
- `components/home/PersonaSection.tsx` — no behavioural change, just not rendered by default at the top of the page.
- `components/Navbar.tsx` — ensure the global "+ Create" button routes to `/create-smart` (the new unified entry), not to a picker.

### Routing

- Entry points that still go directly to their creation page (power-user shortcuts, deep links): `/courses/create`, `/classes/create`, etc. — keep accessible from the chat's Create CTA and from the collapsed persona grid.
- Car Mode: `/car-mode` retains direct-URL entry.

### Analytics

- New event `home_central_chat_submit` with `{input_length, used_voice, from_section: 'hero' | 'collapsed'}`.
- Track `chat_to_create_ratio` — how often the central input leads to a completed Create action (north-star for this feature).

### Accessibility

- Central input must be keyboard-first (autofocus on desktop; skip-focus accessible).
- Mic button needs `aria-label="Speak your idea"` and `aria-pressed` bound to listening state.
- Car Mode button must be reachable via Tab without expanding the persona grid.

## Acceptance criteria

- [ ] Signed-in home page shows one central input above the fold; persona grid is collapsed by default.
- [ ] Central input voice + text both route to `/create-smart?initial_text=...` and the chat auto-starts.
- [ ] Car Mode button is visible on home page without scrolling.
- [ ] "+ Create" in the navbar routes to `/create-smart`.
- [ ] Legacy deep links (`/courses/create`, `/classes/create`, etc.) still work.
- [ ] Signed-out home page uses the same central-input shape (input is sign-in-gated).
- [ ] Playwright: type "interview for software engineer role" into central input → lands on `/create-smart` with the chat already greeting the user about an interview-prep plan.
- [ ] Playwright: click Car Mode → lands on `/car-mode` entry screen.
- [ ] A/B ready: gate the new layout behind a `home_unified_chat_entry` feature flag so we can measure activation delta before global cutover.

## Risks + mitigations

- **Returning users who love the persona grid** — keep the grid accessible in a collapsed section; progressive disclosure, not removal. Measure open-rate of the collapsed section to decide long-term prominence.
- **Chat planner misroutes** — the current `useSmartCreateChat` already proposes plans and asks clarifying questions. If a user means "classroom compliance training" but the planner defaults to "podcast", the Create CTA should offer the alternatives explicitly. Extend the chat plan model with an "artifact type" selector the user can override.
- **Feature flag cold-start** — ship with flag OFF by default on prod, ON for staff users. Measure for one week, then flip.
- **SEO regression on home page** — retain the persona-grid content in the DOM (collapsed), no SSR removal, no meta change.

## Out of scope

- Rebuilding the chat planner itself (already works for Smart Create).
- Adding new creation artifact types.
- Changing Car Mode UX (covered separately in R1 Car Mode production hardening).

## Key file references

- `kitesforu-frontend/app/page.tsx` (home)
- `kitesforu-frontend/app/create-smart/page.tsx` (destination)
- `kitesforu-frontend/components/home/` (PersonaSection, HeroSection, HomeDashboard)
- `kitesforu-frontend/components/Navbar.tsx`
- `kitesforu-frontend/components/ui/voice-input.tsx`
- `kitesforu-frontend/hooks/useSmartCreateChat.ts`
