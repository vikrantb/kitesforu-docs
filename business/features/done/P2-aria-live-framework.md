# P2 — Aria-Live Announcement Framework

**Status**: PROPOSED
**Priority**: P2
**Affected repos**: kitesforu-frontend
**Maps to**: UI Excellence Sweep S3

## Problem

Screen reader users cannot hear async state changes (job completion, segment streaming, voice state transitions, plan ready, error banners, mastery ribbon transitions). Only `ClassFormSection` char-count and `ActivityStats` have `aria-live` regions.

## Scope

- Create a reusable `<AnnouncePolite>` primitive component
- Adopt it across the 6 highest-signal async surfaces:
  1. Job/episode completion
  2. Plan ready (Smart Create)
  3. Voice state transitions (halo)
  4. Error banners
  5. Streaming segment ready
  6. Mastery ribbon transition

## Acceptance criteria

- [ ] `<AnnouncePolite>` component exists and is tested
- [ ] Adopted in 6+ async surfaces
- [ ] VoiceOver on Safari announces each surface change
- [ ] No visual change — purely screen reader

## Effort estimate

6-8 hours, 2 PRs (component + adoption).
