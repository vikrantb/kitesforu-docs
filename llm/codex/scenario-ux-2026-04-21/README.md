# Scenario UX Review — 2026-04-21

This folder contains a scenario-based product review of the beta experience, grounded in:

- live walkthroughs on `https://beta.kitesforu.com`
- the staging scenario tests in `kitetest/tests/staging/user-scenarios.spec.ts`
- the existing business proposals in `business/features/`

This is intentionally **not** a visual-density audit. The question here is simpler:

> For the main user scenarios we say we support, does the actual product make the next step obvious and trustworthy?

## What I tested live

- Fresh / not-yet-onboarded user
- Teacher persona
- Student accounts
- Corporate / HR persona
- Returning generic user

## Core conclusion

The product is strongest when it behaves like a **clear scenario product**:

- interview prep
- create a class
- learn from a class
- create corporate training

The product gets confusing when it falls back into a **content-type product**:

- course vs class vs writeup vs podcast
- `create-smart` vs `classes/create` vs `interview-prep`
- home vs dashboard vs library vs legacy class mental models

The main usability problem is not "too many pixels." It is that the user is often asked to understand the product's internal architecture before the product understands the user's job to be done.

## Files

- `01-live-scenario-audit.md`
  Scenario-by-scenario findings from the live beta.
- `02-plan-home-as-scenario-launcher.md`
  How the signed-in home should become a start/resume surface.
- `03-plan-unified-create-router.md`
  How to remove creation fragmentation without flattening specialized flows.
- `04-plan-onboarding-and-entry-gates.md`
  How to stop onboarding and route gates from breaking the journey.
- `05-plan-classroom-and-learning-journey.md`
  How the teacher/student flow should feel end to end.
- `06-prioritized-roadmap.md`
  Recommended order of implementation.
