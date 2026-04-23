# Research-Grounded Principles

## What "real UX improvement" means here

For KitesForU, real UX improvement does **not** mean:

- adding more cards
- polishing spacing without reducing uncertainty
- improving one screen while leaving the journey fragmented

It **does** mean:

- making the next step more obvious
- lowering the amount of translation the user has to do
- reducing the chance that a motivated user gets interrupted, misrouted, or stalled
- helping the user trust both the system and their own choices

## Principle 1 — Start with the user's job, not the product's structure

GOV.UK's design principles are still the cleanest articulation of this:

- start with user needs
- do less
- do the hard work to make it simple

That matters here because KitesForU currently exposes too much of its own architecture:

- `create-smart` vs `classes/create` vs `interview-prep`
- library vs classes vs home vs dashboard mental models
- corporate language that collapses back into classroom language

The user does not care which subsystem they are entering. They care about the job:

- prep for an interview
- create a class
- train a team
- continue learning

**KitesForU implication**:
The product should route by user job first, then let the internals decide the best flow.

**Sources**

- GOV.UK Government Design Principles
  https://www.gov.uk/guidance/government-design-principles

## Principle 2 — Journeys must join up coherently end to end

GOV.UK's service manual makes the point explicitly: users get stuck when the different parts of the journey do not join up coherently, and teams should map the whole problem rather than optimize disconnected transactions.

This matters because KitesForU has multiple places where the local screen is reasonable but the end-to-end flow is not:

- sign in -> unexpected landing state
- home -> create -> creation detail -> share / continue
- invite / student intent -> onboarding wall
- corporate landing -> classroom-language creation flow

**KitesForU implication**:
Any UX work that improves one screen but leaves the handoff confusing is only partial progress. The real unit of improvement is the whole scenario loop.

**Sources**

- GOV.UK Service Manual, map and understand a user's whole problem
  https://www.gov.uk/service-manual/design/map-a-users-whole-problem

## Principle 3 — Use the user's language, not internal jargon

Nielsen Norman Group's heuristics are blunt about this:

- the design should speak the user's language
- use familiar words and concepts rather than internal jargon

Baymard's research reinforces the same point from a different angle: unclear or jargon-heavy labels cause users to avoid or misunderstand options, and can directly contribute to abandonment.

This is highly relevant to KitesForU:

- "classroom" is wrong for many corporate scenarios
- "library" is a storage concept, not always the user's task concept
- "create-smart" is a product-internal name, not a user intent
- route names and page titles often do not tell the user what they can do next

**KitesForU implication**:
Naming and page framing are not cosmetic. They are part of task comprehension. If the label is wrong, the user either hesitates or picks the wrong path.

**Sources**

- NN/g usability heuristics summary
  https://www.nngroup.com/articles/ten-usability-heuristics/
- Baymard, unclear filtering terminology causes confusion and abandonment
  https://baymard.com/blog/explain-industry-specific-filters

## Principle 4 — Prefer recognition over recall

NN/g's "recognition rather than recall" heuristic is one of the most useful lenses for AI products.

People should not have to remember:

- which route they need
- which fields matter for a strong prompt
- what stage their class or interview series is in
- what the system is currently doing

They should be able to recognize those things from the interface itself.

This is especially important in KitesForU because many flows are cognitively heavy already:

- creation prompts require specific input
- interview prep involves multiple moving parts
- classroom flows span generation, review, share, join, and progress

**KitesForU implication**:
The interface should surface:

- in-context examples instead of expecting the user to infer how to prompt well
- explicit stage labels instead of implicit status
- clear resume targets instead of forcing memory of where work lives

**Sources**

- NN/g, recognition rather than recall
  https://www.nngroup.com/articles/recognition-and-recall/

## Principle 5 — Progressive disclosure is how you manage complexity without dumbing the product down

Apple's design guidance frames progressive disclosure as a way to manage complexity and simplify decision making: show the common choices first, reveal more only when needed.

This matters because KitesForU is genuinely a broad product. The answer is not to flatten it into one generic flow. The answer is to reveal the right level of complexity at the right moment.

Examples:

- one top-level create entry, then specialized flows underneath
- one dominant next action on home, not six competing ones
- light profile capture at the point it becomes relevant, not all up front
- contextual guidance in creation, not a generic tutorial wall before use

**KitesForU implication**:
The product should feel simple at the top and powerful underneath, not complicated everywhere.

**Sources**

- Apple WWDC 2017, Essential Design Principles
  https://developer.apple.com/videos/play/wwdc2017/802

## Principle 6 — Information scent decides whether users keep going

NN/g's information-foraging model is very useful here: users follow paths that look promising relative to the time and effort required.

In KitesForU, that means users quickly judge:

- does this card seem like the thing I need?
- does this page title suggest the right outcome?
- does this route look like it leads to my actual goal?
- is this generation status worth waiting for?

If the scent is weak, they hesitate or bounce.

**KitesForU implication**:
Every high-traffic decision point should make the reward legible:

- what this path is for
- why it is relevant to this user
- what happens next if they click

Weak scent is a hidden tax. It produces uncertainty before any obvious UI bug appears.

**Sources**

- NN/g, information foraging
  https://www.nngroup.com/articles/information-foraging/

## Principle 7 — Visibility of system status is a trust feature, not a cosmetic one

NN/g's first heuristic is visibility of system status: people need timely feedback about what is going on.

In AI products this becomes even more important because users cannot directly see the work. They are forced to infer:

- did the system understand me?
- is it doing anything?
- how far along is it?
- what do I do while it works?

**KitesForU implication**:
Real UX improvement must include better operational trust:

- clear post-create state
- clear generation stage
- explicit next action while waiting
- clear resumption when work is ready

Without this, even a technically-good backend feels unreliable.

**Sources**

- NN/g usability heuristics summary
  https://www.nngroup.com/articles/ten-usability-heuristics/

## Principle 8 — Every extra unit of information competes with the thing that matters

NN/g's minimalist-design heuristic is often misunderstood as "make it pretty." The real point is sharper: every extra unit of information competes with relevant units of information.

For KitesForU, that means the problem is often not visual density alone. The problem is competitive messaging:

- several different "next steps" on one page
- dashboard information competing with creation actions
- recommendations competing with resume-work
- generic feature promotion competing with persona-specific intent

**KitesForU implication**:
The UX should minimize competition between actions. A page should not make the user negotiate between multiple equally-loud interpretations of what it is for.

**Sources**

- NN/g usability heuristics summary
  https://www.nngroup.com/articles/ten-usability-heuristics/

## Synthesis for KitesForU

Taken together, the principles above point to one strong conclusion:

> KitesForU should be designed as a scenario-first, state-aware, low-translation product.

In practical terms that means:

- scenario-first: the product understands whether the user is teaching, learning, interviewing, training, or storytelling
- state-aware: the product knows whether the user is starting, waiting, resuming, reviewing, sharing, or learning
- low-translation: the user does not have to decode route names, object types, or hidden system status to keep moving

That is the lens the rest of this package uses.
