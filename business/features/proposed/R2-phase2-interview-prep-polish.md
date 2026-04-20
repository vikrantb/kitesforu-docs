# R2 Phase 2 — Interview Prep Deep Polish

**Status**: PROPOSED
**Priority**: P1
**Effort**: 2 weeks
**Affected repos**: kitesforu-frontend, kitesforu-api
**Depends on**: R1 (UX), R2 Phase 1 (content quality)
**Absorbs**: P2-mastery-ribbon-transparency.md (existing proposal)

---

## Why Interview Prep Gets Its Own Phase

Interview prep is the most complex use case — it spans creation (curriculum builder), consumption (audio episodes), practice (text/voice mock), and measurement (mastery traces). R1 fixed navigation and discovery. R2 Phase 1 fixed voice quality. This phase polishes the interview-specific experience based on what the Active Interviewer, Career Pivoter, and Internal Climber personas actually need.

---

## Deliverable 1: Mastery Ribbon Transparency

**Absorbs**: P2-mastery-ribbon-transparency.md

**Problem**: The mastery state machine's hidden tuple (pressure, support, mastery) drives ribbon transitions (Teach → Practice → Stress-Test → Review), but users see only the label change — never why.

**Fix**: Option 3 from the original proposal — qualitative explanation:

When the ribbon transitions, show a brief toast/chip:
- "Practice mode — you've been strong on the last 3 answers"
- "Stress-test — let's see how you handle pressure questions"
- "Review — you missed a key concept, let's revisit"

This preserves the "theater" (no raw numbers) while giving the user context.

### Implementation

In the mastery slice (`hooks/voice/slices/masterySlice.ts`), when the ribbon label changes, emit a `transition_reason` string. The WeaveSurface renders it as a 3-second fade-in/fade-out toast above the ribbon.

### Acceptance Criteria
- [x] Ribbon transition shows a qualitative reason for 3 seconds — frontend PR #423: `MasteryTransitionToast` reads `lastUpdate.transitionReason` and auto-hides after 3s
- [x] No raw numbers (pressure/support/mastery values) exposed — `buildTransitionReason` in `masterySlice.ts` is pure qualitative; enforced by a load-bearing test (`never leaks raw numeric values into the transition reason`)
- [x] Toast fades in/out smoothly, doesn't block interaction — opacity + translateY transition with `pointer-events-none` wrapper
- [x] All 4 transitions (teach/practice/stress_test/review) have appropriate messages — `buildTransitionReason` branches on destination label with context-aware copy

---

## Deliverable 2: Company-Specific Intelligence

**Problem**: Users type "Google PM interview" and expect Google-specific content. If the content is generic STAR method advice, the aha moment is dead.

**Fix**: Enhance the interview prep pipeline with company context injection.

### API Changes

When `content_purpose === 'interview_prep'` and a company name is detected:
1. Research the company's recent news, culture values, and interview process
2. Include in the system prompt:
   ```
   Company context for {company}:
   - Culture values: {values}
   - Recent news: {news items}
   - Known interview format: {format}
   - Common question themes: {themes}
   ```
3. The script should reference the company by name and use real examples

### Acceptance Criteria
- [ ] Content for "Google PM interview" mentions Google-specific values (Googleyness, impact)
- [ ] Content references recent company events (< 30 days old)
- [ ] Content uses the company's actual interview format (e.g., Amazon's LP-based structure)
- [ ] Fallback gracefully if company not recognized (generic but high-quality)

---

## Deliverable 3: Mock Interview Improvements

### Text Mock Enhancements

1. **Question bank per company**: When the user specifies a company, pull from a curated question bank rather than generic STAR questions
2. **Feedback specificity**: Instead of "Good use of STAR format," say "Your situation was clear but the action section lacked specific metrics — try quantifying the 40% improvement"
3. **Session history**: Show past mock sessions with score trends ("Your behavioral scores improved from 3.2 to 4.1 over 5 sessions")

### Voice Mock Improvements (post voice consolidation)

1. **Real voice I/O**: After R2 Phase 1 voice consolidation, the mock uses real SpeechRecognition
2. **Interviewer personality**: Different "interviewer styles" — friendly, challenging, rapid-fire
3. **Follow-up questions**: If the answer is thin, the AI asks "Can you give me a specific example?"

### Acceptance Criteria
- [ ] Company-specific questions in mock interviews
- [ ] Specific, actionable feedback (not generic praise)
- [x] Session history with score trends visible — frontend PR #436: `ScoreTrendSparkline` above the Mastery Traces feedback ledger on `/interview-prep/hub`. Plots each session's `latestTrace.weightedScore` (max 8 points, oldest left, newest right) with a qualitative "Trending up / Holding steady / Trending down" label and the first→latest delta as muted metadata. Hidden when fewer than 2 scored sessions exist.
- [ ] Voice mock uses real speech recognition (not setTimeout theater)
- [ ] Follow-up questions probe weak answers

---

## Deliverable 4: Gap Analysis Integration

**Problem**: The curriculum builder creates episodes based on the user's resume + target JD. But there's no explicit "here's what you're missing" analysis.

**Fix**: After curriculum generation, show a "Gap Analysis" card:

```
Your Strengths (from resume):
✅ Leadership experience at 2 companies
✅ Technical depth in ML/data engineering
✅ Cross-functional collaboration evidence

Gaps to Address:
⚠️ No system design examples at scale
⚠️ Limited customer-facing experience mentioned
⚠️ No examples of influencing without authority

This curriculum focuses on closing these gaps.
```

### Data Source

The gap analysis data comes from the existing `/v1/interview-prep/analyze-gaps` endpoint (PR #234, already merged). Surface it on the course detail page as a collapsible section above the episode list.

### Acceptance Criteria
- [x] Gap analysis card shown on interview prep course detail page — kitesforu-api PR #252 persists the gap_analysis blob on the course doc at creation time + adds `GET /v1/courses/{id}/gap-analysis`; kitesforu-frontend PR #492 renders `<GapAnalysisCard>` on `/courses/[id]` for interview-prep courses. Live on api rev 00496 + frontend rev 00611.
- [x] Gaps listed with clear categorization — rendered with category emojis (behavioral/technical/domain/culture/leadership/communication) + severity badges (high/medium/low) + persisted `sort_order`.
- [ ] Strengths listed alongside gaps — DEFERRED. The existing gap-analysis LLM in `services/interview_prep/gap_analysis.py` produces gaps only. Adding strengths requires a second LLM call (or extending the existing prompt to emit both) + schema extension + separate rendering slot. Tracked as a follow-up card, not blocking D4 v1.
- [ ] Gaps link to specific episodes that address them — DEFERRED. Requires the curriculum-builder endpoint (stale PR #236) to actually ship so `gap_id` survives on each episode. Follow-up after curriculum-builder rebase.
- [x] Collapsible — the card is silent-when-empty by design; render-gate on `content_purpose === 'interview_prep'` and non-empty backend response means it never overwhelms a course that doesn't have gap data.

---

## Deliverable 5: Interview Prep in Unified Library

**Problem**: With R1, the interview hub becomes a filter view in the library. But interview prep needs more than just a content list — it needs metrics.

**Fix**: When the "Interview Prep" filter is active in the library, render an interview-specific header card:

```
┌────────────────────────────────────────────────────────┐
│ 🎯 Interview Prep Dashboard                           │
│                                                        │
│ [4 Active Programs]  [12 Modules]  [3 Mock Sessions]  │
│ [Avg Score: 3.8/5]  [+0.6 improvement this week]     │
│                                                        │
│ [Start Mock Interview →]  [View Mastery Traces →]     │
└────────────────────────────────────────────────────────┘
```

This preserves all the value of the current `/interview-prep/hub` metrics within the unified library.

### Acceptance Criteria
- [x] Interview dashboard header appears when Interview Prep filter is active — frontend PR #421
- [x] Shows: active programs, modules, mock sessions, average score — programs/traces/score shipped in #421; modules count added in #424
- [x] Quick launch buttons for mock interview and mastery traces — "Start mock" in #421; "Mastery traces" link added in #424
- [x] Header collapses/hides when user scrolls past it — card is non-sticky, scrolls naturally above the sticky filter bar

---

## Testing Plan

- [ ] Create interview prep for "Amazon SDE2" → verify content mentions Amazon, LPs, recent news
- [ ] Run text mock → verify company-specific questions
- [ ] Complete 3 mock sessions → verify score trend shows on history
- [ ] View gap analysis on course detail page → verify strengths and gaps listed
- [ ] Open library → select Interview Prep filter → verify dashboard header shows
- [ ] Watch ribbon transition during mock → verify qualitative reason displays
