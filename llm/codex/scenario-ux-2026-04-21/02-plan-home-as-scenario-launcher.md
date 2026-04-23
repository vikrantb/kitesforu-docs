# Plan A — Home As Scenario Launcher

## Goal

Make signed-in home answer one question fast:

> What is the most sensible next thing for this user to do right now?

## Problem to solve

Today home is trying to be all of these at once:

- stats dashboard
- recent work shelf
- template catalog
- persona recommendation engine
- feature advertisement

That creates a second-order problem: even when the modules are good, the user still has to decide which module matters.

## Proposed home structure

### Block 1 — Resume

Always first.

If the user has anything active, show only the highest-value resume item:

- continue class generation
- continue interview prep series
- continue lesson 2 of 7
- review class before sharing

This block should feel operational, not inspirational.

### Block 2 — Start the right thing

Exactly 3 large cards, role-adaptive.

Teacher:

- Create a class
- Share / manage classes
- Continue learning content

Corporate:

- Create training
- Create a team update
- Build onboarding or compliance

Interview persona:

- Start mock interview
- Build interview series
- Review past mastery

Student:

- Continue learning
- Join a class
- Browse your library

### Block 3 — Recent work

Useful, but demoted below resume and start.

### Block 4 — Recommendations

Only if they are genuinely role-specific. If not, hide them.

## Rules

1. No home section should exist unless it helps the user take action.
2. If two blocks compete for "what next," one of them is wrong.
3. `Quick Actions` must be persona-faithful, not globally popular.
4. Home should prefer "resume" over "discover" for returning users.

## Concrete product changes

- Replace the current generic quick-action set with role-aware action sets.
- Only show `Recommended for you` when the recommendation is stronger than generic templates.
- Convert `Your creation footprint` into a secondary row or collapsible summary.
- Promote one clear operational CTA for each active scenario.

## Why this matters

A usable home page is not a summary of the product. It is a decision reducer.
