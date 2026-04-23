# Preliminary Prioritization

This is the initial point of view before a second critique pass.

## Priority test

The first things to do should be the changes that most improve these four questions:

1. Does the user know where to start?
2. Does the user know what to do next?
3. Does the user understand what the system is doing?
4. Can the user keep moving without translating product architecture?

## Wave 1 — Highest leverage

### 1. Stop interrupting intent with avoidable setup or dead ends

**Includes**

- fix any bad post-auth landing / dead-end states
- let join / invited / continuation paths bypass generic onboarding
- defer profile capture until it is relevant

**Why this is first**

Nothing damages trust faster than blocking a motivated user before they reach value.

### 2. Rebuild signed-in home around Start / Resume

**Includes**

- state-aware home behavior
- one dominant next step
- clearer distinction between resume, review, and start new

**Why this is first**

This simplifies the daily experience for almost every returning user, regardless of persona.

### 3. Introduce one canonical create entry by user job

**Includes**

- a single top-level create route
- job-to-flow routing underneath
- fewer architecture-exposing route decisions

**Why this is first**

This removes a major recurring comprehension tax across multiple scenarios.

### 4. Clean up page naming and subsequent-page framing

**Includes**

- titles
- route labels
- CTA language
- stage framing on later pages

**Why this is first**

Bad naming quietly multiplies confusion across the whole system. It is a small change with outsized cognitive impact.

## Wave 2 — Strong next layer

### 5. Strengthen prompt guidance as confidence scaffolding

**Includes**

- scenario-specific guidance v2
- clarifier preemption
- better defaults + examples
- optional second-step capture only when it materially helps

**Why now**

This improves creation quality and lowers frustration, but it works best once route and scenario clarity are improved.

### 6. Add explicit post-create stage and progress states

**Includes**

- generation stages
- clear "what happens next"
- resume / review / share handoff

**Why now**

This improves trust in the long-running AI parts of the product and reduces perceived flakiness.

### 7. Make teacher / student / corporate workflows feel native

**Includes**

- teacher control tower
- student continue-learning surface
- corporate training mode language and handoffs

**Why now**

This is where scenario credibility becomes real, especially beyond interview prep.

## Wave 3 — Valuable, but after the core gets simpler

### 8. Replace generic recommendations with justified recommendations

**Includes**

- "because you are doing X"
- role-aware and state-aware suggestions

### 9. Refine library into a better operational surface

**Includes**

- better scenario overlays
- stronger resume and review cues

### 10. Expand personalization where it improves decisions rather than just feeling clever

**Includes**

- better defaults
- better ordering
- stronger relevance

## What should not lead the roadmap

These may still matter, but they should not be mistaken for the highest-value UX work:

- isolated cosmetic polish
- adding more homepage modules
- deeper recommendation widgets before the core journey is clear
- copy-only improvements that do not change the actual decision burden

## Working final recommendation

If KitesForU wants the most meaningful UX shift, the roadmap should start with:

1. remove interruption before value
2. simplify the home decision
3. unify creation at the job level
4. rename and reframe subsequent pages around real tasks
5. improve prompt confidence and progress trust

That sequence attacks the deepest causes of confusion rather than merely making the existing complexity prettier.
