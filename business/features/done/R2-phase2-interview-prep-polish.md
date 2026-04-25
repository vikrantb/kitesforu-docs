# R2 Phase 2 — Interview Prep Deep Polish

**Status**: DONE (2026-04-24 — D1–D5 substantively shipped; voice-mock real STT deferred to voice consolidation thread)
**Priority**: P1
**Effort**: 2 weeks
**Affected repos**: kitesforu-frontend, kitesforu-api
**Depends on**: R1 (UX), R2 Phase 1 (content quality)
**Absorbs**: P2-mastery-ribbon-transparency.md (existing proposal)

## Implementation summary (2026-04-24 close-out)

- **D1 — Mastery ribbon transparency**: `MasteryTransitionToast` (frontend PR #423) renders qualitative reason for 3s. `buildTransitionReason` is pure-qualitative; load-bearing test pins "never leaks raw numeric values".
- **D2 — Company-specific intelligence**: 3-PR chain (api #273 curriculum-builder + #274 gap-analysis + #275 Tavily-backed `company_research.py`). Canonical anchors per company (Amazon LPs, Google Googleyness + GCA, Meta coding-heavy, Microsoft collab-design, Apple craft, Netflix F&R, McKinsey/BCG/Bain case structure). 5s hard timeout, never-raises, never-invent. Activates when `TAVILY_API_KEY` set; falls back to training-data behavior otherwise.
- **D3 — Mock interview improvements**: Company-specific question framing (schemas #77 + api #276); evaluation prompt emits `top_fix` / `rewrite` / `followup_question`; `ScoreTrendSparkline` (frontend PR #436) above the Mastery Traces ledger; follow-up questions probe weak answers when any dimension scores ≤ 2.0 or a STAR component is absent.
- **D4 — Gap analysis integration**: api #252 persists + endpoint, frontend #492 ships `<GapAnalysisCard>` ABOVE `EpisodeList` per §4 (corrected via #590 + order-pin test on 2026-04-24); api #254 + frontend #494 add strengths section first; api #255 + frontend #496 link gaps to specific draft episode numbers.
- **D5 — Interview Prep in unified library**: frontend #421 + #424 — dashboard header (active programs / modules / mock sessions / avg score) appears when Interview Prep filter is active; quick-launch buttons; non-sticky scroll behavior.

**Deferred to a dedicated thread (out of scope for this phase's closure)**:
- **Voice mock real speech recognition** (D3 final AC): blocked on R2 Phase 1 D1 voice architecture consolidation. Will land alongside the LiveKit adapter replacement of `MockVoiceController`.

This proposal is closed for shipping purposes.

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
- [x] Content for "Google PM interview" mentions Google-specific values (Googleyness, impact) — api PR #273 (curriculum-builder) + PR #274 (gap-analysis) add explicit "Company-specific tailoring" rules to the system prompt with canonical anchors (Amazon LPs, Google Googleyness + GCA, Meta coding-heavy, Microsoft collaborative-design, Apple craft, Netflix F&R, McKinsey/BCG/Bain case structure). Both LLM paths (gap cards → curriculum episode titles → workers composer) now weave these cues naturally.
- [x] Content references recent company events (< 30 days old) — api PR #275 adds `services/interview_prep/company_research.py` (Tavily-backed, 5s hard timeout, top-3 + 400-char cap, NEVER-RAISES contract). Both gap_analysis and curriculum_builder await `fetch_recent_events(target_company_name)` before the LLM call and render a "RECENT COMPANY CONTEXT (last 30 days)" prompt block with a "never invent detail beyond what is cited here" scope directive. 12 new tests cover every failure mode (missing key / HTTP error / timeout / malformed body / empty results) + happy path + prompt shape. **Requires `TAVILY_API_KEY` in Cloud Run secrets to activate**; unset → returns None → training-data-only behavior preserved (AC not regressed).
- [x] Content uses the company's actual interview format (e.g., Amazon's LP-based structure) — same PR #273/#274 anchor list explicitly names the public interview formats; prompt prefers "Walk through a 'Deliver Results' STAR for Amazon" over generic behavioral framing when target is Amazon.
- [x] Fallback gracefully if company not recognized (generic but high-quality) — shipped end-to-end across all three paths: prompt clause ("If target company is not well-known, fall back to strong generic structure — do not guess"), company_research silent-None on Tavily failure, NEVER-invent clause prevents LLM from inventing company facts.

**D2 STATUS — ALL 4 ACs SHIPPED.** Three-PR chain complete: api #273 (curriculum-builder prompt) + #274 (gap-analysis prompt) + #275 (recent-events research step).

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
- [x] Company-specific questions in mock interviews — schemas PR #77 (v1.57.0) added `GenerateConceptsRequest.target_company`; api PR #276 threads it through `teaching.generator` → `teaching.prompt` with the same company-tailoring guidance used by curriculum-builder (#273) and gap-analysis (#274). System prompt names canonical anchors (Amazon Leadership Principles, Google Googleyness + GCA, Meta coding-heavy, Microsoft collaborative-design, Apple craft, Netflix F&R, McKinsey/BCG/Bain case-structure), forbids inventing private rubrics, and falls back to strong generic framing when the company is not well-known. Teach-phase concepts now weave company-specific framing into titles + `why_this_role`. Full question-bank tagging (per-company curated questions in the Elo-selected bank) remains a follow-up only if usage data shows the Teach-phase signal isn't sufficient.
- [x] Specific, actionable feedback (not generic praise) — shipped via the evaluation prompt in `services/mock_interview/evaluation/prompt.py`: LLM produces `top_fix` ("one concrete specific fix, max 200 chars"), `rewrite` ("one concrete rewrite of the weakest sentence, max 300 chars"), and `followup_question`. UI renders all three in `TurnFeedbackCard` + `CoachFeedbackCard`. The system-message explicitly structures feedback around the 5-dim rubric (specificity, quantified_impact, ownership, role_outcome_clarity, level_appropriate), so outputs name specific dimensions rather than defaulting to generic praise.
- [x] Session history with score trends visible — frontend PR #436: `ScoreTrendSparkline` above the Mastery Traces feedback ledger on `/interview-prep/hub`. Plots each session's `latestTrace.weightedScore` (max 8 points, oldest left, newest right) with a qualitative "Trending up / Holding steady / Trending down" label and the first→latest delta as muted metadata. Hidden when fewer than 2 scored sessions exist.
- [ ] Voice mock uses real speech recognition (not setTimeout theater) — remaining; separate frontend thread (post voice-architecture-consolidation).
- [x] Follow-up questions probe weak answers — shipped end-to-end: schemas PR #72 added `TurnFeedback.followup_question` + `CoachFeedback.followup_question`; api PR #257/#258 taught the evaluation prompt to emit a probing follow-up when any dimension scored ≤ 2.0 or a STAR component is absent; frontend renders the probe in both assessment and practice feedback cards.

**D3 STATUS — 4 of 5 ACs SHIPPED.** Voice-mock real-speech-recognition remains (blocked on voice-architecture consolidation).

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
- [x] Gap analysis card shown on interview prep course detail page — kitesforu-api PR #252 persists the gap_analysis blob on the course doc at creation time + adds `GET /v1/courses/{id}/gap-analysis`; kitesforu-frontend PR #492 renders `<GapAnalysisCard>` on `/courses/[id]` for interview-prep courses. Live on api rev 00496 + frontend rev 00611. **Position corrected 2026-04-24 via frontend #590** — originally rendered below `EpisodeList` by oversight; §4 Data Source specifies the card renders ABOVE the episode list (so users see "what you already bring" + gap framing before the tactical episode list). Order pinned by `__tests__/app/course-detail-gap-analysis-position.test.ts` so a future refactor cannot silently regress it.
- [x] Gaps listed with clear categorization — rendered with category emojis (behavioral/technical/domain/culture/leadership/communication) + severity badges (high/medium/low) + persisted `sort_order`.
- [x] Strengths listed alongside gaps — kitesforu-api PR #254 extends the single LLM pass to emit 2-5 strengths with `category` / `title` / `description` / `resume_evidence` / `jd_alignment` alongside the existing 3-7 gaps. Backend prompt rules bake in "real concrete assets, no generic praise, fewer is better than fabricated". Strengths persist on `course.gap_analysis.strengths` and surface via `GET /v1/courses/{id}/gap-analysis`. kitesforu-frontend PR #494 renders a new "What you already bring" section FIRST inside the existing `<GapAnalysisCard>` (emerald palette, category emoji, JD-alignment caption). Render-gate fires on EITHER gaps OR strengths so an ultra-well-matched candidate still sees the card. Live on api rev 00498 + frontend rev 00613.
- [x] Gaps link to specific episodes that address them — end-to-end shipped 2026-04-21 (self-corrected an earlier overclaim, then closed for real). kitesforu-api PR #255 extends `CreateInterviewPrepRequest` to accept `draft_episodes: list[DraftEpisodeInput]` and persists them on `course.gap_analysis.draft_episodes` alongside gaps + strengths; `GET /v1/courses/{id}/gap-analysis` surfaces the field when present. kitesforu-frontend PR #496 sends `draft_episodes` in the submit body and renders `"Covered in: Episode 1, Episode 3"` under each gap card — episode numbers are 1-indexed from `sort_order` (course workers generate in that order, so draft # matches the eventual real episode #). Line silently omitted for legacy courses without persisted episodes and for gaps that have no matching episode (e.g. custom gap added after curriculum-build). 3 new tests; 12 total passing in `gap-analysis-card.test.tsx`. Live on api rev 00499 + frontend rev 00615.
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
