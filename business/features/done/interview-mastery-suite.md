# Interview Mastery Suite

**Status**: DONE
**Shipped**: April 12-15, 2026
**Repos**: kitesforu-frontend, kitesforu-api, kitesforu-workers
**PRs**: #285, #298-316 (frontend), API routes, workers mastery PRs C1-C4

## What it does

Three complementary products for interview preparation, connected by a shared Mastery Trace:

1. **Interactive Mock Interview** — Text-based STAR-method drills with 5-dimension rubric scoring and Elo-adaptive difficulty
2. **Voice-First Simulation** — Audio interview room with listening halo, coaching chips, and one-fix replay cards
3. **Custom Interview Series** — Upload resume + target role → generates a multi-episode audio prep course

## Key capabilities

- **5-dimension rubric**: specificity, quantified impact, role-outcome clarity, ownership, level-appropriate
- **Elo adaptation**: difficulty adjusts per seniority, seeded per role
- **One-fix coaching**: after every answer, one highest-impact improvement suggestion
- **Mastery Trace**: event-sourced progress tracking (schemas v1.45.0)
- **Voice-first concierge**: voice input is the primary entry point for the creation flow
- **Free-tier course support**: 30-second preview episodes for free accounts
- **Progressive disclosure**: upload/URL options hidden behind a toggle
- **Auto-advance sections**: 600ms debounce when section becomes valid

## User impact

Executives can prep for interviews via voice, text, or audio — whichever fits the moment. All three products feed a single progress trace so improvement is visible across modalities.
