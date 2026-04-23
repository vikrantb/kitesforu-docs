# Plan C — Progressive Onboarding And Entry Gates

## Goal

Capture enough profile signal without blocking high-intent or invitation-led flows.

## Current problem

Fresh and student accounts are aggressively redirected to `/onboarding` from multiple routes.

That creates friction for users who came to do one concrete thing:

- join a class
- open a shared link
- continue learning
- inspect the product before committing

## Better rule

Onboarding should be a **booster**, not a wall.

## Proposed gating model

### Hard-gate only where needed

Hard gate:

- first creation flow if profile is truly required
- settings/profile completion when the user explicitly opens it

Do not hard-gate:

- class join links
- class learning routes
- library browse
- interview prep landing page

### Progressive capture

Collect profile detail in context:

- before first class creation: ask teaching context
- before first interview series: ask role/company/timeline
- before first corporate training flow: ask team/use case

### Lightweight fallback

If persona is unknown, default the experience and let behavior teach the system.

## Specific scenario guidance

### Student join flow

If a student opens a share link or class code:

1. let them authenticate
2. show class preview
3. let them join
4. ask profile/setup later if needed

### Returning invited user

Do not bounce them through generic onboarding before they even see the object they were invited to.

### Fresh self-serve user

Show onboarding as a short assist, but allow skip/defer.

## Additional trust fix

The post-auth bad landing on `/sign-in` + `Page not found` needs immediate fixing. It destroys confidence in the first 10 seconds.

## Why this matters

The easiest way to make a product feel unusable is to block intent with setup.
