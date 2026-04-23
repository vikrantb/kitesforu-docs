# Prioritized Roadmap

## P0 — Must fix now

### 1. Fix post-sign-in bad landing

If sign-in can leave the user authenticated on `/sign-in` with `Page not found`, fix that before almost anything else.

Why:

- breaks trust immediately
- makes the product feel unstable
- poisons first-run experience

### 2. Stop hard-gating key routes behind onboarding

At minimum, do not hard-gate:

- class join
- class learning
- library
- interview-prep landing

Why:

- invitation-led and student-led journeys are fragile
- onboarding should not outrank intent

## P1 — Biggest usability win

### 3. Rebuild signed-in home around Start / Resume

Why:

- fixes the daily decision problem
- helps every persona
- makes the product feel simpler immediately

### 4. Unify create entry at the job-to-be-done level

Why:

- removes route guessing
- preserves specialized flows
- reduces product-architecture leakage

## P2 — High-value scenario corrections

### 5. Remove classroom language from corporate creation

Why:

- current mismatch is obvious and credibility-damaging
- small copy / routing changes could create a large perceived improvement

### 6. Make teacher home teacher-first

Why:

- current quick actions skew too hard toward interview prep
- class creation, review, and sharing should dominate

### 7. Make student entry and continue-learning flow explicit

Why:

- the student journey is where extra friction hurts most
- student motivation is low compared to teacher/admin motivation

## P3 — Structural cleanup

### 8. Normalize naming and route mental models

Resolve the mixed language between:

- dashboard vs home
- library vs classes
- classrooms vs class cards
- create-smart vs create

### 9. Make "recommended" truly role-aware or remove it

Why:

- generic recommendations add noise
- weak recommendations make the product feel less intelligent than it is

## Recommended sequencing

### Phase 1

- auth landing fix
- onboarding gate reduction
- home rewrite

### Phase 2

- unified create router
- corporate copy/frame split
- teacher-first home actions

### Phase 3

- student-specific resume/join/continue improvements
- naming cleanup
- recommendation tightening

## Final recommendation

If only one strategic move gets made soon, make it this:

> Turn the product into a scenario-first system with one obvious next step per persona.

That will do more for usability than another round of local UI polish.
