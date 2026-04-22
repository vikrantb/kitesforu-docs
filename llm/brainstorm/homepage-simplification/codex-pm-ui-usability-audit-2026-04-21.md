# Codex PM UI Usability Audit

Date: 2026-04-21
Environment: `https://beta.kitesforu.com`
User used for audit: `test@kitesforu.com`

## What I looked at

- Signed-out homepage
- Signed-in homepage
- `create-smart`
- `interview-prep`
- `library`

This is a product/UX audit, not a code audit.

## Executive view

The product has real strengths:

- The visual system is distinctive.
- The interview-prep surface is materially clearer than the rest of the app.
- The core idea of "describe what you want and we figure out the right format" is strong.

The biggest problem is not lack of features. It is lack of hierarchy.

Right now the app often asks the user to understand:

- the overall product category
- the right creation mode
- the right content type
- their recent work
- quick actions
- recommended actions
- templates
- library state

all at once.

That creates a "smart but noisy" experience. Beauty is in simplicity, and today too many surfaces compete for attention at the same time.

## The core product question

The UI currently sends two different stories:

1. KitesForU is an interview-mastery product.
2. KitesForU is a broad AI creation workspace for audio, classes, writing, and more.

Both may be true, but the interface is trying to say both at once.

This is the root usability issue.

If the product wants to be broad, the homepage needs a simple organizing principle.
If the product wants to lead with interview prep, then the rest of the product should feel like expansion paths, not equal-weight choices.

## What is confusing today

### 1. Signed-out homepage mixes one strong promise with too many product branches

The hero is sharp:

- "Master your next interview"
- "The B2B Mastery Companion"

But the same page also exposes:

- search
- create
- pricing
- three different interview modes
- footer links for quick audio, writeups, classrooms, writing, audio series

The user gets a focused headline and an unfocused information architecture.

### 2. Signed-in homepage is trying to be four pages at once

The signed-in homepage currently behaves like:

- a dashboard
- a launcher
- a recommendation engine
- a resume-work surface

all on one page.

Specific symptoms:

- welcome banner
- creation footprint metrics
- recent items
- quick actions
- recommended for you
- a large create composer
- more starting cards below it

Each block is individually reasonable. Together they compete.

### 3. The create surface is strategically right but operationally under-guided

`/create-smart` is probably the right long-term center of gravity.

But today it feels too empty and too subtle:

- the main input is large but under-explained
- the tabs (`Say it`, `I have notes`, `Starting points`) are too quiet
- the primary action does not feel strong enough
- the user is not given enough confidence about what "good input" looks like

This is a page with a good idea and a weak handoff.

### 4. The library is dense and hard to scan

The library has real content, but it feels like a database view more than a workspace.

Specific problems:

- too many repeated or near-duplicate items
- mixed content types with weak visual separation
- long titles dominate the page
- filters are present but not visually strong enough
- the most important state is not obvious: what should I resume, fix, or ignore?

### 5. Some trust signals are accidentally weak

Examples:

- landing on a logged-in 404 after sign-in is a major trust hit
- `999,999 credits` looks fake or placeholder-like
- decorative kites sometimes compete with content instead of supporting it
- tooltips can obscure important text

These are small individually, but together they make the experience feel less calm.

## What good user experience should feel like

For KitesForU, good UX should feel:

- calm
- obvious
- confidence-building
- personalized without being chaotic
- fast to start
- easy to resume

The user should always know:

1. What is this page for?
2. What is the best next action?
3. What happens if I click it?
4. Where is my work?

If a page cannot answer those four questions in a glance, it is too complicated.

## Product direction: the simplest good version

The best direction is likely:

- one primary action
- one strong secondary action
- one resume/in-progress area
- everything else demoted or deferred

In other words:

- the homepage should be a decision reducer, not a catalog
- the create page should be the canonical input surface
- the library should be the canonical management surface

## Concrete ideas

### Idea A: Split the signed-out and signed-in jobs more aggressively

Signed-out homepage:

- sell one clear promise
- show one primary CTA
- explain the product in one path
- optionally show two secondary modalities, not many

Signed-in homepage:

- stop being a marketing page
- become a workspace home

### Idea B: Make the signed-in homepage a "Start or Resume" page

Above the fold should contain only:

- one personalized primary CTA
- one resume card if something is in progress
- one alternate path chooser

Everything else goes below the fold or to dedicated pages.

Example structure:

1. Primary CTA:
   - `Start with AI`
2. Resume CTA:
   - `Continue your latest interview prep`
   - or `Continue your last audio series`
3. Simple path picker:
   - `Interview prep`
   - `Create learning content`
   - `Quick audio briefing`

Not six sections. Not twelve links.

### Idea C: Turn `create-smart` into the real front door

This page should become dramatically more guided.

It needs:

- better example prompts
- clearer mode explanation
- stronger attachment affordance
- visible examples by scenario
- a more obvious primary action

The page should teach the user how to succeed, not just wait for input.

### Idea D: Make the library task-oriented, not inventory-oriented

Reframe the library into:

- In progress
- Ready to use
- Drafts
- Archived / older

And visually group by actionability, not only by content type.

The first question in the library should be:

- what should I continue?

not:

- how many mixed assets exist in my account?

### Idea E: Use interview-prep as the quality benchmark

The interview-prep page is the clearest surface in the current product set.

Why it works better:

- one clear promise
- a bounded set of choices
- simple comparison between the choices
- supportive detail under the options

That pattern should be reused elsewhere.

## High-priority issues

### P0 issues

- Logged-in post-auth path can land on a 404 state.
- Homepage has no single dominant next action once signed in.
- Product positioning is split between "interview mastery" and "general AI creation workspace."

### P1 issues

- `create-smart` is under-guided and too sparse for first-use confidence.
- Library is too dense and too repetitive to support fast resumption.
- The signed-in homepage repeats creation entry points in too many different modules.

### P2 issues

- Visual decoration sometimes adds motion/noise without adding clarity.
- Credits and some small trust signals feel placeholder-like.
- Footer and global navigation contribute to option overload on already busy pages.

## Recommended prioritization

### 1. Simplify the signed-in homepage first

This has the highest leverage because it affects every returning user.

Target outcome:

- one primary CTA
- one resume block
- one lightweight path chooser
- move metrics and extra recommendations lower or elsewhere

### 2. Improve `create-smart` second

This is likely the best long-term canonical entry point, but it needs stronger scaffolding.

Target outcome:

- better input examples
- clearer scenario-specific prompts
- stronger "what happens next" explanation
- less empty feeling

### 3. Reframe the library around resumption

Target outcome:

- easier scanning
- better grouping
- fewer repeated cards
- faster "continue working" behavior

### 4. Resolve the product-story conflict

Leadership/product decision:

- Is KitesForU led by interview mastery with adjacent creation tools?
- Or is it a broader creation platform where interview prep is the flagship use case?

The UI will keep feeling confused until this is explicit.

## A good future state

The elegant version of KitesForU would feel like this:

- I arrive and immediately understand the best next action.
- The app remembers what I was doing and offers to continue it.
- If I want to create something new, there is one obvious place to start.
- The app helps me give good input instead of making me guess.
- My library feels like a workspace, not a dump of everything ever created.

That is the usability bar worth aiming for.
