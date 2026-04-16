# P0 Phase 1 — Smart Intake + Gap Analysis

**Status**: READY FOR IMPLEMENTATION
**Priority**: P0 — implement first
**Affected repos**: kitesforu-frontend, kitesforu-api
**Depends on**: existing ResumeSection + TargetPositionSection (already shipped in concierge PRs)
**Estimate**: 14-20 hours

---

## The User Journey (Phase 1 only)

```
┌─ Step 1: Your Background ────────────────────────────┐
│  [Voice hero] Talk through your background            │
│  [Textarea] Or paste / type                          │
│  (Progressive disclosure: upload / URL)               │
│  → Auto-extracts CandidateProfile                    │
│  → Auto-advances to Step 2                           │
└──────────────────────────────────────────────────────┘
         ↓ (existing — already shipped)
┌─ Step 2: Your Target ────────────────────────────────┐
│  [Voice hero] Tell me about the role                  │
│  [Textarea] Or paste the JD                          │
│  (Progressive disclosure: URL)                        │
│  → Auto-extracts TargetProfile                       │
│  → Auto-advances to Step 3                           │
└──────────────────────────────────────────────────────┘
         ↓ (existing — already shipped)
┌─ Step 3: Your Focus (NEW) ───────────────────────────┐
│                                                       │
│  "Analyzing your background against this role..."     │
│  (loading state while gap analysis runs)              │
│                                                       │
│  → Shows 3-7 structured Gap Cards:                    │
│                                                       │
│  ☑ Leadership Transition                    HIGH      │
│    Your resume shows IC clinical work at Community    │
│    Hospital. This charge nurse role at Mayo Clinic     │
│    requires direct team leadership experience.        │
│                                                       │
│  ☑ Electronic Health Records               MEDIUM    │
│    JD mentions Epic EHR integration. Your resume      │
│    doesn't reference any EHR system by name.          │
│                                                       │
│  ☑ Behavioral: Conflict Resolution         HIGH      │
│    Charge nurse roles always probe conflict handling.  │
│    No conflict examples found in your background.     │
│                                                       │
│  ☐ Patient Safety Protocols                LOW       │
│    Minor gap — JD mentions Joint Commission;          │
│    your background implies but doesn't state this.    │
│                                                       │
│  [Voice hero] Anything else you want to practice?     │
│  [Text input for custom focus areas]                  │
│                                                       │
│  ── Quick path: "Looks good, build my plan →"         │
│  ── Deep path: uncheck gaps, drag to reorder,         │
│     add custom focus areas, adjust priority            │
│                                                       │
└──────────────────────────────────────────────────────┘
         ↓ advances to Phase 2 (Curriculum Builder)
```

## UX Principles Applied

### Least friction (fast path)
- Gap analysis runs AUTOMATICALLY the moment both profiles are extracted
- All gaps are pre-checked with AI-recommended priority order
- "Looks good, build my plan" is ONE CLICK to advance
- The entire flow from resume upload to gap confirmation can be 3 clicks: upload → auto-advance → confirm

### Total control (deep path)
- Every gap card has a checkbox — uncheck to exclude
- Drag to reorder priority (top = most episodes devoted to it)
- Severity badges (HIGH / MEDIUM / LOW) are editable
- "Add your own" voice/text input for focus areas the AI missed
- Each gap card is expandable to show the AI's reasoning

### Easy and beautiful
- Gap cards use the same Card aesthetic as the rest of the app
- Severity badges use the brand color scale (HIGH = red/orange, MEDIUM = amber, LOW = gray)
- Voice hero for adding custom focus areas — consistent with Steps 1 and 2
- Loading state for gap analysis is the same ThinkingIndicator pattern, not a spinner
- Auto-advance from Step 2 → Step 3 means the gap analysis starts before the user expects it (feels fast)

---

## Data Model

### Gap (frontend + API shared type)

```typescript
interface Gap {
  id: string                           // uuid
  category: 'behavioral' | 'technical' | 'domain' | 'culture' | 'leadership' | 'custom'
  title: string                        // "Leadership Transition"
  description: string                  // The full explanation of why this is a gap
  severity: 'high' | 'medium' | 'low'  // AI-assessed importance
  source: 'resume_gap' | 'jd_requirement' | 'industry_standard' | 'user_added'
  resume_evidence?: string             // What the resume says (or doesn't say)
  jd_evidence?: string                 // What the JD requires
  checked: boolean                     // User's selection state (default: true)
  sort_order: number                   // User can reorder
}
```

### GapAnalysisRequest (API input)

```typescript
interface GapAnalysisRequest {
  candidate_profile: CandidateProfile   // Already extracted by existing pipeline
  target_profile: TargetProfile         // Already extracted by existing pipeline
  interview_focus: string               // behavioral | technical | system_design | mixed
  user_context?: string                 // "Where I need help" text from Step 2
}
```

### GapAnalysisResponse (API output)

```typescript
interface GapAnalysisResponse {
  gaps: Gap[]                           // 3-7 structured gaps
  summary: string                       // One-sentence overview for the UI
  recommended_episode_count: number     // AI's suggestion (3-7)
  confidence: number                    // 0-1, how confident the analysis is
}
```

---

## Implementation Steps (in order)

### Step 1: Shared types (frontend)
- Add Gap, GapAnalysisRequest, GapAnalysisResponse to `shared/schemas.ts`
- 30 minutes

### Step 2: Gap analysis API endpoint (API repo)
- New route: `POST /v1/interview-prep/analyze-gaps`
- Takes CandidateProfile + TargetProfile + interview_focus
- Calls an LLM (Gemini or Claude) with a structured prompt that produces Gap[]
- Returns GapAnalysisResponse
- 6-8 hours (prompt engineering is the bulk of the work)

### Step 3: GapAnalysisSection component (frontend)
- New component: `components/interview-prep/create/GapAnalysisSection.tsx`
- CollapsibleSection wrapper (Step 3)
- Loading state (ThinkingIndicator while API runs)
- Gap cards with checkboxes, severity badges, expand/collapse for reasoning
- Drag-to-reorder (or simpler: up/down arrows)
- Voice hero for "Anything else you want to practice?"
- "Looks good" primary CTA
- 8-10 hours

### Step 4: Wire into create page
- Add GapAnalysisSection as Step 3 in the interview-prep create flow
- Trigger gap analysis API when both profiles are extracted (Step 2 completes)
- Pass gap results to the next phase (curriculum builder)
- 2-4 hours

### Step 5: Hook extension
- Extend `useInterviewPrepForm` with gap analysis state:
  - `gaps: Gap[]`
  - `gapsLoading: boolean`
  - `gapsError: string | null`
  - `toggleGap(id: string): void`
  - `reorderGaps(from: number, to: number): void`
  - `addCustomGap(title: string, description: string): void`
  - `section3Valid: boolean` (at least 1 gap checked)
- 3-4 hours

---

## What This Replaces

Today, Step 2 (Your Target) has a "Next: Review" button that goes directly to a Review section where the user hits "Generate." There's no gap analysis, no focus selection, no curriculum preview.

After Phase 1, Step 2 auto-advances to Step 3 (Your Focus) which shows the gap analysis. The "Next: Review" button is replaced by "Build my prep plan" which advances to Phase 2 (Curriculum Builder).

---

## Mock Data for Frontend-First Development

While the API endpoint is being built, the frontend can use mock gap data:

```typescript
const MOCK_GAPS: Gap[] = [
  {
    id: 'gap-1',
    category: 'leadership',
    title: 'Leadership Transition',
    description: 'Your resume demonstrates strong individual clinical skills at Community Hospital. This charge nurse role at Mayo Clinic requires direct team leadership, shift coordination, and mentoring responsibilities that aren\'t evidenced in your background.',
    severity: 'high',
    source: 'resume_gap',
    resume_evidence: 'IC clinical roles listed; no management or team lead titles',
    jd_evidence: 'JD requires: "Lead a team of 8-12 RNs, manage shift transitions, mentor new staff"',
    checked: true,
    sort_order: 0,
  },
  // ... more gaps
]
```

This allows the frontend to ship the full GapAnalysisSection UX before the API is ready. When the API ships, swap mock data for the API call.
