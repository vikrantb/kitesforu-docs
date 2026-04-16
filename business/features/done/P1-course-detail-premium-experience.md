# P1 — Course Detail Premium Experience

**Status**: PROPOSED
**Priority**: P1 (high — the course detail page is the primary consumption surface)
**Affected repos**: kitesforu-frontend
**Key files**: app/courses/[courseId]/page.tsx, components/courses/detail/*

## Problem

The course detail page is functional but doesn't reflect the premium quality of what the backend now produces. The pipeline ships voice persona, genre-aware expressions, variable pauses, naturalness post-processing, comedy timing, and broadcast-standard LUFS — but the frontend shows a plain episode list with basic play buttons. Users can't see or feel the quality innovations.

## User story

As a user who generated a Custom Interview Series, I want the course detail page to feel like a premium audio product — showing me the voice persona, episode quality indicators, genre theming, and a listening experience that matches the production quality of the audio itself.

## Scope

### Episode cards
- Show the **voice persona name and avatar** (e.g., "Hosted by Dr. Sarah Chen") per episode
- Show **genre tag** (comedy, horror, educational, etc.) with genre-appropriate color theming
- Show **episode duration** prominently (formatted as "12 min" not raw seconds)
- Show **generation quality score** if available from the quality gate

### Player experience
- **Progressive loading indicator**: show which segments are generated vs pending (SSE streaming is already wired — surface it visually)
- **Waveform visualization** or at minimum a progress bar that feels premium (not the basic HTML5 audio scrubber)
- **Playback speed controls**: 0.75x, 1x, 1.25x, 1.5x, 2x
- **Skip 15s forward/back buttons** (standard podcast UX)

### Course header
- Show the **persona** that hosts the series
- Show **total duration** (sum of all episodes)
- Show **completion percentage** (episodes listened vs total)
- **Genre-themed gradient** on the header (horror = dark, comedy = warm, educational = blue)

### Follow-up questions (in-episode)
- Surface the **Quick Question overlay** from Car Mode inside the course player
- Users should be able to ask "wait, explain that part again" mid-listen and get a voice response

## Acceptance criteria

- [ ] Episode cards show persona name + genre tag + duration
- [ ] Player has speed controls + skip buttons
- [ ] Course header shows total duration + completion %
- [ ] Genre theming applies to header gradient
- [ ] Quick Question overlay works from the course detail player
- [ ] All new interactive elements hit 44px touch target minimum
- [ ] Screen reader announces episode transitions and player state changes

## Dependencies

- Voice persona data must be persisted in the episode Firestore document (check if `stages.job-audio.persona` is already written)
- Genre profile data must be available from the episode or course document
- SSE streaming segments are already wired in the frontend (hooks/useStudioStreaming.ts)

## Effort estimate

8-12 hours across 3-4 PRs.
