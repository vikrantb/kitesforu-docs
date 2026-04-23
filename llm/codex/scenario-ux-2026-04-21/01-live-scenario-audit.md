# Live Scenario Audit — 2026-04-21

## Method

I used the beta environment and the shared `kitetest` credentials to walk the product as a user, then compared the live behavior against the intended scenarios in:

- `kitetest/tests/staging/user-scenarios.spec.ts`
- `business/features/proposed/R1-phase1-ux-foundation.md`
- `business/features/proposed/R1-phase3-ux-creation-home.md`
- `business/features/proposed/R2-phase2-interview-prep-polish.md`
- `business/features/done/R3-profile-driven-personalization.md`

## North-star judgment

The product should feel like:

- "I want to prep for an interview"
- "I want to create a class"
- "I want to train my team"
- "I want to continue learning"

It too often feels like:

- "Should I use create-smart or classes/create?"
- "Why does corporate training say classroom?"
- "Why is my teacher home pushing interview prep?"
- "Why am I blocked by onboarding before I can do the thing I came to do?"

## Scenario 1 — Fresh user onboarding

### Intended scenario

The user should quickly identify their lane, give just enough context, and start doing real work.

### Live behavior

- A fresh user is hard-routed to `/onboarding` from `/`, `/dashboard`, `/classes`, `/library`, and `/interview-prep`.
- The onboarding persona set is broad and sensible: interview prep, quick audio, study, creative stories, K-12, university, corporate, self-paced.
- The user cannot inspect the rest of the product first and decide later.

### Product judgment

The onboarding content is not the problem. The **hard gate** is the problem.

This is acceptable for creation-heavy paths, but risky for:

- student join flows
- returning invited users
- users arriving with one concrete intent and low patience

### What feels wrong

The product asks for identity before it has earned enough trust.

## Scenario 2 — Teacher creates a class

### Intended scenario

Teacher should be able to:

1. start from home
2. create a class quickly
3. understand what happens next
4. share it with students

### Live behavior

- The actual class creation path is still `classes/create`.
- The page title is **Create a Classroom**.
- The path is coherent once the teacher is there.
- The teacher's signed-in home is not coherent for the teacher scenario.

Observed home emphasis for `teacher1`:

- recent content and class drafts
- quick actions: `Quick Mock`, `Tech Interview`, `Behavioral Prep`

### Product judgment

The home page is not acting like a teacher's control tower. It is acting like a mixed activity wall with interview-prep bias.

That creates two problems:

1. the teacher's highest-value next step is not obvious
2. the teacher has to mentally switch from "my classroom work" to "generic KitesForU capabilities"

### What feels wrong

The teacher has already told us who they are, but the product still behaves like it is not sure.

## Scenario 3 — Corporate / HR user

### Intended scenario

Corporate user should feel like they are creating training, onboarding, compliance, or enablement content for teams.

### Live behavior

Home for `hrperson1` is much closer to the right shape:

- `Corporate Training`
- `Team Update`
- `Newsletter`
- `Quick Briefing`

But the create flow is still the same `classes/create` surface with:

- `Create a Classroom`
- `Audio Lessons Your Students Will Love`
- `University-Grade Audio Lectures`

### Product judgment

This is one of the clearest scenario mismatches in the product.

The corporate persona is reasonably recognized on home, then immediately shoved back into an education-first mental model the moment they try to create.

### What feels wrong

The product says "we know you're corporate" and then the next page says "welcome to classrooms."

## Scenario 4 — Student learning

### Intended scenario

Student should be able to:

1. join class
2. continue lessons
3. take quizzes
4. move between lessons easily

### Live behavior

- `student1` and `student2` were both routed to onboarding instead of a usable student experience.
- This means the real student-learning scenario is fragile unless onboarding has already been completed.
- The scenario spec expects join, learn, and quiz paths to be accessible; the live entry state is still identity-gated.

### Product judgment

This is dangerous because a student journey is often invitation-led and low-intent. Students are not trying to "set up a profile." They are trying to get into class.

### What feels wrong

The product makes the student earn the right to learn.

## Scenario 5 — Library / classes / dashboard navigation

### Intended scenario

Phase 1 says the system should simplify around:

- `/`
- `/library`
- `+ Create`

and redirect old routes cleanly.

### Live behavior

- `/classes` correctly redirects to `/library?type=class`
- `/dashboard` effectively collapses to `/`
- the top nav is simplified
- but the internal language is still mixed:
  - `Library`
  - `My Classes`
  - `Classrooms`
  - `Create a Classroom`
  - content-type tabs

### Product judgment

The routing simplification landed technically, but the **mental model simplification is incomplete**.

The user still sees several competing concepts for where work lives and how to start it.

## Scenario 6 — Interview prep

### Intended scenario

Interview prep should feel like a specialized, high-value product, not a generic course builder.

### Live behavior

The interview-prep landing page is one of the best surfaces in the product:

- clear options
- clear promise
- clear distinctions between text, voice, and series creation

### Product judgment

This is important because it shows the right answer:

- scenario-first
- concrete outcomes
- low ambiguity

Interview prep feels usable because it is built around the user's job, not around internal content objects.

## Cross-cutting trust issue — sign-in landing bug

I was able to reproduce a logged-in bad landing after sign-in:

- URL remained `/sign-in`
- page heading was `Page not found`
- user was clearly authenticated in nav state

This is a high-priority trust issue. It breaks the journey before the product can even help.

## Main findings

1. The biggest usability problem is **journey fragmentation**, not visual clutter.
2. The product is strongest where it is **scenario-native**.
3. The product is weakest where specialized journeys leak internal content-type architecture.
4. Home is not yet a reliable **what should I do next?** surface.
5. Onboarding is overused as a hard gate.
6. Corporate and teacher flows still collapse back into an education/classroom frame too often.
