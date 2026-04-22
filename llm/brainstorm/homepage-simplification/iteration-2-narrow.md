# Iteration 2 — Narrow to survivors

Four candidates survive iteration 1 critique. This pass puts each through first-order pressure-testing against the README's 6 evaluation criteria.

Scorecard (1-5 per criterion; higher = better fit for KitesForU):

| Criterion | D (split-by-auth) | F (situation-first) | G (resume-first) | D+F (compose) |
|---|---|---|---|---|
| 1. 5-second "what + do" | 3 | 5 | 4 (signed-in only) | 5 |
| 2. Exactly one above-fold primary | 4 | 5 | 5 | 5 |
| 3. "If in shell, not on home" | 5 | 4 | 5 | 5 |
| 4. Honest language | 3 | 5 | 4 | 5 |
| 5. Legible first flow | 4 | 4 | 5 | 5 |
| 6. Beautiful-simple | 4 | 5 | 4 | 5 |
| **Total** | 23 | 28 | 27 | 30 |

## Why candidates D and F compose rather than compete

D is an *architectural* recommendation: signed-out and signed-in should not share components. F is a *positioning* recommendation: the hero promise should be situation-first, not interview-first. These are orthogonal. The strongest candidate is **D+F**: architectural split with situation-first positioning on both halves.

G (resume-first) is a valuable ingredient inside the D+F signed-in branch — it answers *what goes in the signed-in hero when the user has an in-progress item*. It's not a standalone homepage; it's the primary-action state of the signed-in dashboard.

B (show-don't-tell) is a valuable *below-the-fold* signed-out section — a single row of 3-4 real sample episodes that prove the promise. It's not a hero; it's proof.

## The D+F+G+B composition

| Zone | Signed-out | Signed-in |
|---|---|---|
| Hero (above fold) | F: one promise + one input + one CTA | G (if in-progress) or F's input (otherwise) |
| Proof (just below fold) | B: 3-4 sample audio cards | "Continue" row (specific items, not links to `/library`) |
| Deep surfaces | Linked to `/pricing`, `/help`, `/whats-new` in footer only | Below-fold: small "Recent" strip (3 items max), then nothing |
| Deleted entirely | BottomCTAs, persona grid, Mastery-Loop explainer, 4 ValueCards, FirstVisitBanner, CreatorStatsStrip (moved to `/dashboard`) | Same list |

## What this composition still owes its reader

1. **The hero copy, for real.** Agent 4 drafted three options; iteration 3 picks one.
2. **The signed-in state machine.** G (resume-first) vs F (fresh input) depends on user state — how does the component decide?
3. **The signed-out CTA discipline.** "Start with a sentence" vs "Listen to a sample" — which leads? Both are honest, both fit the wedge.
4. **What specifically replaces the persona grid.** Agent 3 says it moves to `/create`. Iteration 3 sketches the handoff.

## Loser-candidates: one-line verdicts

- **A (input-only)**: too spartan for a signed-out stranger. Works for Perplexity because "search" is already a known verb; "voice-first audio for your situation" is not.
- **C (persona lanes)**: forces a binary choice the KitesForU user doesn't want. Creator-consumers slide between modes; a forced lane taxes everyone.
- **E (interview-forward)**: doubles down on the miscast Agent 4 diagnosed. The interview-prep-only hero is what the page currently does and why it's broken.
- **H (chat-is-the-page)**: collapses `/` into `/create-smart`, erasing the returning-user dashboard job. Signed-in users need a next-action surface, not a blank chat.

## Going into iteration 3

D+F+G+B survives. Iteration 3 stress-tests it against:
- Mobile vs desktop (the bottom-nav `+` already competes with the hero on mobile).
- Signed-in with 0 items vs 1 item vs many items (the dashboard has to degrade gracefully).
- A user who only wants to listen (consumer-lane) — does the hero serve them?
- A user who only wants to interview-prep (the current-PM audience) — does the hero demote them unacceptably?
