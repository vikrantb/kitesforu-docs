# P2 — Pricing Page Credit-Depletion Context

**Status**: PROPOSED
**Priority**: P2
**Affected repos**: kitesforu-frontend
**Maps to**: UI Excellence Sweep S13

## Problem

Users who hit a credit wall get redirected to /pricing and see generic marketing ("Simple, transparent pricing"). No banner, no context, no "you hit your limit" message. The mental model gap is massive: they're frustrated, not shopping.

## Scope

- Detect when the user arrives at /pricing via a credit-depletion redirect (query param `?reason=credits` or referrer check)
- Show a contextual banner: "You've used all your preview credits. Upgrade to continue creating."
- Show their current credit balance and what they've used
- Highlight the plan that would have covered their last attempt

## Acceptance criteria

- [ ] Banner shows when user arrives via credit depletion
- [ ] Banner does NOT show for direct visits to /pricing
- [ ] Current credit balance displayed
- [ ] Recommended plan highlighted

## Effort estimate

4-6 hours, 1 PR.
