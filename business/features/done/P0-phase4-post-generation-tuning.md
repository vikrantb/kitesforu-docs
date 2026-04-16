# P0 Phase 4 — Post-Generation Review & Tuning

**Status**: PROPOSED (implement after Phase 3)
**Priority**: P0
**Affected repos**: kitesforu-frontend, kitesforu-api
**Depends on**: Phase 3 (episodes must be drill-structured for tuning to be meaningful)
**Estimate**: 8-12 hours

---

## What This Phase Does

After the course generates, the user can review, adjust, and request targeted regeneration of specific episodes — without regenerating the entire course.

## User Journey

```
┌─ Course Detail Page (after generation) ──────────────┐
│                                                       │
│  Your Prep Plan: Charge Nurse at Mayo Clinic          │
│  5 episodes · 47 min · Generated from your resume     │
│                                                       │
│  [Listen All]  [Share]  [Edit Plan]                   │
│                                                       │
│  Episode 1: Leadership Transition Drill  ▶ ⟳         │
│    "Great coverage of the ICU surge scenario. I'd     │
│     rate this 4/5 for relevance to your background."  │
│    [Regenerate with different focus]                   │
│    [Add a follow-up episode on this gap]               │
│                                                       │
│  Episode 2: EHR Systems Deep Dive        ▶ ⟳         │
│    [Regenerate] [Skip — I know this well]              │
│                                                       │
│  ── After listening ──                                │
│                                                       │
│  "How was that episode?"                              │
│    [Too easy] [Just right] [Too hard]                 │
│    [Not relevant to my interview]                     │
│                                                       │
│  → Feedback adjusts future regeneration params         │
│                                                       │
└──────────────────────────────────────────────────────┘
```

## Key Capabilities

### Per-episode actions
- **Regenerate**: re-run a single episode with the same gap but different questions/scenarios
- **Regenerate with different focus**: change what aspect of the gap is drilled
- **Skip**: mark as "I know this well" — adjusts the curriculum emphasis
- **Add follow-up**: generate an additional episode that goes deeper on the same gap

### Post-listen feedback
- Quick reaction: Too easy / Just right / Too hard / Not relevant
- Feeds back into the curriculum builder for future sessions
- Over time, builds a preference model per user (stored in Mastery Trace)

### Curriculum-level adjustments
- "Add 2 more behavioral episodes" — generates new episodes from existing gaps
- "Focus more on technical, less on behavioral" — reweights the curriculum
- "I got the job — now prep me for the first 90 days" — pivots the entire curriculum to a new goal

## Implementation Steps

1. Per-episode regeneration endpoint: `POST /v1/interview-prep/regenerate-episode` (API, 4-6h)
2. Post-listen feedback UI on course detail page (frontend, 3-4h)
3. "Add follow-up episode" flow (frontend + API, 3-4h)
4. Feedback storage in Mastery Trace events (workers, 2h)

## Why This Matters

This is what makes the product **sticky**. The first generation gets the user 80% of the way. Tuning gets them to 95%. The feedback loop means every subsequent course is better calibrated. Users come back because the system learns what they need.
