# P1 — Progressive Episode Streaming on Course Pages

**Status**: PROPOSED
**Priority**: P1
**Affected repos**: kitesforu-frontend
**Key files**: hooks/useStudioStreaming.ts, components/courses/detail/EpisodeList.tsx, components/courses/detail/GenerationProgress.tsx

## Problem

When a course is generating, users see a generic progress bar and must wait for ALL episodes to complete before playing any. The SSE streaming infrastructure is already built (PR #278, hooks/useStudioStreaming.ts) but only wired to the Studio page — not the course detail page.

## User story

As a user who just started generating a 10-episode series, I want to start listening to Episode 1 the moment it's ready — not wait 20 minutes for all 10 to complete.

## Scope

- Wire `useStudioStreaming` (or a similar SSE hook) into the course detail page
- As each episode completes, immediately enable its play button and show a "Ready" badge
- Show a per-episode progress indicator (generating → ready → playing → listened)
- Auto-play the first completed episode (opt-in, with a user preference)
- Show remaining time estimate based on average episode generation time

## Acceptance criteria

- [ ] Episode play button enables the moment that episode's audio_url is set
- [ ] Progress shows per-episode state (pending → generating → ready)
- [ ] First-ready episode can auto-play (with user opt-in)
- [ ] Remaining time estimate updates as each episode completes
- [ ] Works alongside Drive Mode (if user started via Drive, episodes stream to the player)

## Dependencies

- Course orchestrator already updates episode status per-episode (worker.py `_update_episode_status`)
- SSE streaming hooks exist but need a course-specific adapter
- Audio player component already supports playlist mode with auto-advance

## Effort estimate

6-8 hours, 2 PRs (hook adapter + UI wiring).
