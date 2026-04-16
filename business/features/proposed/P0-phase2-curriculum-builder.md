# P0 Phase 2 — Curriculum Builder + Preview

**Status**: PROPOSED (implement after Phase 1)
**Priority**: P0
**Affected repos**: kitesforu-frontend, kitesforu-api, kitesforu-course-workers
**Depends on**: Phase 1 (gap analysis produces the input)
**Estimate**: 16-22 hours

---

## The User Journey (Phase 2)

After the user confirms their focus areas in Phase 1's gap analysis:

```
┌─ Step 4: Your Prep Plan (NEW) ───────────────────────┐
│                                                       │
│  "Building your curriculum..."                        │
│  (loading — AI generating drill-structured episodes)  │
│                                                       │
│  → Shows the full curriculum preview:                 │
│                                                       │
│  ┌─ Episode 1 ─────────────────── Behavioral ──────┐ │
│  │ Leadership Transition Drill                      │ │
│  │ Practice describing a time you led a team        │ │
│  │ through a complex shift change. Calibrated to    │ │
│  │ Mayo Clinic's leadership expectations.           │ │
│  │ [Edit] [Remove]                       ↕ drag    │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─ Episode 2 ─────────────────── Technical ───────┐ │
│  │ EHR Systems Deep Dive                           │ │
│  │ Walk through clinical information system         │ │
│  │ workflows. Focused on Epic — the system          │ │
│  │ specifically mentioned in the JD.                │ │
│  │ [Edit] [Remove]                       ↕ drag    │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─ Episode 3 ──────────────────── Behavioral ─────┐ │
│  │ Conflict Resolution Scenarios                    │ │
│  │ Real charge nurse conflict scenarios: staffing   │ │
│  │ disputes, patient family escalations, physician  │ │
│  │ disagreements. STAR-structured practice.         │ │
│  │ [Edit] [Remove]                       ↕ drag    │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  [+ Add another episode]                              │
│                                                       │
│  ── Settings (collapsed by default) ──                │
│     Episode length: [10 min ▾]                        │
│     Voice style: [Professional coaching ▾]            │
│     Total estimated time: ~50 min                     │
│                                                       │
│  ── Quick path: "Generate my prep course →"           │
│  ── Deep path: edit any episode, reorder, add/remove  │
│                                                       │
└──────────────────────────────────────────────────────┘
```

## UX Principles

### Least friction
- Curriculum auto-generates from the confirmed gaps — no user input needed
- Default settings (10 min episodes, professional coaching voice) are pre-selected
- "Generate my prep course" is ONE CLICK from the preview
- Episode count matches the recommended count from gap analysis

### Total control
- Every episode title + description is editable inline
- Episodes are drag-to-reorder
- Remove any episode, add new ones
- Settings panel (collapsed) lets user adjust duration, voice style, and other params
- Each episode shows its interview type badge (Behavioral, Technical, Culture) — changeable

### Beautiful
- Episode cards with interview-type color-coded badges
- Drag handles for reorder
- Inline edit mode (click title → becomes input, blur → saves)
- Collapsed settings with progressive disclosure

---

## Data Model

### CurriculumDraft

```typescript
interface DraftEpisode {
  id: string
  title: string
  description: string
  interview_type: 'behavioral' | 'technical' | 'system_design' | 'culture_fit' | 'case_study' | 'mock_final'
  gap_id: string                    // links back to the Gap it addresses
  drill_structure: string           // "STAR prompts" | "problem walkthrough" | "values alignment" | "full simulation"
  sort_order: number
  estimated_duration_min: number
}

interface CurriculumDraft {
  episodes: DraftEpisode[]
  total_duration_min: number
  recommended_voice_style: string    // "Professional coaching" | "Warm mentor" | etc.
  summary: string                    // One-sentence curriculum overview
}
```

### API

- `POST /v1/interview-prep/build-curriculum` — takes Gap[] + CandidateProfile + TargetProfile → returns CurriculumDraft
- `POST /v1/interview-prep/generate` — takes finalized CurriculumDraft → kicks off course generation with specialized prompts

---

## Implementation Steps

1. **Shared types**: DraftEpisode, CurriculumDraft in shared/schemas.ts (30 min)
2. **Curriculum builder API endpoint** (API repo, 8-10h — LLM prompt engineering for drill structure)
3. **CurriculumPreviewSection component** (frontend, 6-8h — editable episode cards, drag reorder, settings)
4. **Generate endpoint** that threads resume + JD + gaps into each episode's prompt (API/workers, 4-6h)
5. **Wire into create page** as Step 4 (frontend, 2h)

---

## What Changes in the Workers

The course orchestrator's `_build_episode_instructions` method currently sends generic context. Phase 2 changes it to include:

```python
# Per-episode context for interview prep
parts.append(f"CANDIDATE RESUME: {resume_summary}")
parts.append(f"TARGET ROLE: {target_role} at {company}")
parts.append(f"GAP BEING DRILLED: {gap_description}")
parts.append(f"INTERVIEW TYPE: {interview_type}")
parts.append(f"DRILL STRUCTURE: This is a {drill_structure}, not a lecture.")
parts.append(f"Include 3 practice prompts where the listener pauses and answers out loud.")
```

This is the change that transforms generic explainers into targeted drills.
