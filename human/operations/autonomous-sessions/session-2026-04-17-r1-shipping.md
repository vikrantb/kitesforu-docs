---
name: R1 Autonomous Session Handoff (2026-04-17)
description: Shipping status after long autonomous session on R1 roadmap — what's live on beta, what's still open, where to resume.
type: project
originSessionId: 127d6e39-652b-4445-857e-bd58af3b0fbf
---
## Context

Autonomous execution of the R1-R3 roadmap from `business/features/proposed/` per user mandate on 2026-04-17. Started with reconnaissance on the (then) P1-P2 items; after discovering the docs reorg (PRs #35/#36) moved those into `done/` or `superseded/`, pivoted to shipping the actual R1 phases instead.

## Why this matters

**How to apply**: Next session should open with `gh pr list --repo=vikrantb/kitesforu-frontend --state open` and pick up from PR #339 (or whatever is open). Deferred items are explicit — don't redo them assuming they're missing.

## What shipped on beta (verified)

| Phase/Del | PR | Revision | Verified |
|---|---|---|---|
| R1 P1 D1 — schemas `content_purpose` v1.46.0 | schemas #66 | Artifact Registry | tests pass + dep published |
| R1 P1 D1 — API executors + list endpoints + backfill | api #237 | kitesforu-api-00481 | /openapi.json 200 |
| R1 P1 D1 — FE `isInterviewPrepCourse` | fe #334 | kitesforu-frontend-00453 | tsc + lint |
| R1 P1 D2 — `/library` unified page | fe #335 | kitesforu-frontend-00454 | curl /library 307 (Clerk) |
| R1 P1 D3 — redirects + Navbar + BottomNav | fe #336 | kitesforu-frontend-00455 | **5 redirects curl-verified** |
| R1 P2 D4 — dark mode infra + core surfaces | fe #337 | kitesforu-frontend-00456 | tsc + CI |
| R1 P2 D3 — Car Mode class/studio + episodeIndex | fe #338 | deploying at handoff | tsc + CI |
| R1 P3 D1 — /create landing page (slice) | fe #339 | open PR, CI in progress | tsc |

### Verified redirects (2026-04-17 ~16:19 UTC)

- `/courses` → 307 → `/library?type=course`
- `/classes` → 307 → `/library?type=class`
- `/writeups` → 307 → `/library?type=writeup`
- `/activity` → 307 → `/library`
- `/interview-prep/hub` → 307 → `/library?type=interview_prep`

## Backfill script

`kitesforu-api/scripts/backfill_content_purpose.py --dry-run | --apply` is shipped but **NOT RUN** in prod yet. Existing courses/classes/writeups/podcast_jobs in Firestore still don't have `content_purpose`. Run from a workstation with ADC:

```
python scripts/backfill_content_purpose.py --dry-run   # review
python scripts/backfill_content_purpose.py --apply
```

Logic: `courses` with `course_type=='interview_prep'` → `interview_prep`, others → `course`; `classes` → `class`; `writeups` → `writeup`; `podcast_jobs` → `podcast` except those with `parent_course_id` (course episodes stay untagged).

## Deferred items (intentionally, with reasons)

### R1 Phase 2 (3 of 5 deliverables NOT shipped)

- **D1 Persistent Audio Player** — large refactor: move `useAudioPlayer` state to module level via `drive-audio-bridge.ts` pattern, render `<PersistentPlayer>` in `app/layout.tsx`, add MiniPlayer refinement, FullPlayer bottom-sheet refinement, dynamic layout padding, MediaSession persistence. Needs a dedicated session; high iOS Safari risk.
- **D2 Speaker Visualization** — **BLOCKED on multi-repo backend change**. Per recon, workers/course-workers do NOT emit per-segment timestamps to Firestore (no `stages.job-audio.segments[].start_ms/end_ms`). Frontend can't map `currentTime → speaker` without this. Fix: workers emit segment timeline + speaker + persona_id during TTS orchestration → frontend reads it. Spans kitesforu-workers, kitesforu-course-workers, kitesforu-schemas.
- **D3 Q&A button in FullPlayer** — deferred; needs a non-Car-Mode-session episode Q&A endpoint + `useEpisodeQA` hook wrapper. Shipped the **Car Mode button expansion** (class + studio + episodeIndex) as the standalone slice.
- **D4 Dark Mode** — shipped infrastructure + core surfaces (Navbar, BottomNav, Library, AudioContentCard). **Deferred**: per-page dark polish for course/class/writeup detail pages, smart-create chat, studio, auth/onboarding. Drop-in follow-up PRs (mostly adding `dark:` classes).
- **D5 Sleep Timer & Offline** — not started. `useSleepTimer` exists but not surfaced in FullPlayer; offline download needs PWA Service Worker + Cache API.

### R1 Phase 3 (2 of 3 deliverables NOT shipped)

- **D1 Create Page** — shipped as slice (hero + template grid + blank slate). **Deferred from slice**: smart intent suggestions (needs classifier endpoint), voice input button in hero.
- **D2 Personalized Home Dashboard** — not started. Requires activity feed hook, persona-adaptive quick actions, contextual greeting, time-aware copy.
- **D3 Smart-Create classification clarification** — not started. Needs `clarify_content_purpose` SSE event in `services/smart_create/intake.py` with confidence threshold; frontend handler in `useSmartCreateChat`.

### R2 Phase 1 — Content Quality & Voice

172-line spec, not started. Absorbs P2-voice-architecture-consolidation.

### R2 Phase 2 — Interview Prep Polish

162-line spec, not started. Absorbs P2-mastery-ribbon-transparency.

### R3 — Platform Growth & Distribution

214-line spec, not started.

### Phase B — Audit `done/` features (7 issues each)

Not started. 18 items total (P0 interview prep curriculum phases, P1 course detail premium, P2 aria-live / pricing, systems like voice-persona + audio-naturalness etc). Per user mandate: identify exactly 7 new issues, potential regressions, or UX improvements per feature; fix them all; Playwright verify.

## Key recon findings to NOT redo

- Car Mode endpoint `POST /v1/car-mode/session/{session_id}/question` is reusable for episode Q&A if caller pre-populates a session-like doc. The existing `CarModeQuestionService._build_system_prompt` could be factored to take explicit context (topic/title/outline/qa_history) without a session doc.
- `QuickQuestionOverlay` (`components/drive/QuickQuestionOverlay.tsx`) is already well-decoupled from Car Mode. Takes `qq: UseQuickQuestionReturn`. Reusable as-is for any audio context providing `sessionId` + `pauseEpisode` + `resumeEpisode`.
- `useQuickQuestion` (`hooks/drive/useQuickQuestion.ts`) options include `pauseEpisode`, `resumeEpisode`. Pause mechanism already implemented with `pausedByQQRef` to avoid conflicting toggles.
- Speaker metadata in Firestore: `stages.job-audio.persona` (single persona object, last one wins), script segments have `speaker` field but **NO timestamps**. Fixing this is the prerequisite for P1-3 / Phase 2 D2.
- `kitesforu-schemas` v1.46.0 adds `content_purpose: Optional[str]` to `Course`, `CourseListItem`, `Class`, `ClassListItem`, `Writeup`, `WriteupListItem`, `PodcastJob`. `CONTENT_PURPOSE_VALUES` tuple documents accepted values.

## Recommended entry point for next session

Option A (most value): **Run the backfill script in prod** so existing content classifies correctly under the unified library. That's a 30-second task and unlocks the R1 Phase 1 UX fully.

Option B (continue roadmap): **R1 Phase 2 D4 dark-mode polish** — extend `dark:` classes to remaining high-traffic pages (course detail, smart-create chat). Low risk, visible impact.

Option C (unblock bigger deliverables): **R1 Phase 2 D2 speaker timeline** — start with workers emitting `stages.job-audio.segments[]` with timestamps + speaker + persona_id. Spans workers/course-workers/schemas. Preparation for speaker viz.

Option D (start Phase B): **Audit P0-phase1-smart-intake-gap-analysis.md** — highest-priority done-item per user's ordering. Identify exactly 7 issues with `--ultrathink --seq`, fix each, ship one PR per fix or one big PR with clearly separable commits.

User's mandate ordered R1 → R2 → R3 → Phase B, so prefer finishing R1 before starting Phase B. R1 Phase 1 is the only phase fully shipped; R1 Phase 2 and R1 Phase 3 have big deferred deliverables.

## Caveats to remember

- Legacy page files (`app/courses/page.tsx`, `app/classes/page.tsx`, `app/writeups/page.tsx`, `app/activity/page.tsx`, `app/interview-prep/hub/page.tsx`) are unreachable after D3 redirects but **still exist in the codebase**. Delete them in a follow-up after one production cycle confirms no regressions.
- Redirects are 302 (`permanent: false`) — flip to 301 once stable.
- Mobile users can't access the theme toggle yet (ThemeToggle is `hidden md:inline-flex`). Surface it in the Profile tab settings once that tab is built.
- `/create` is now a real page; `/create2` still redirects to `/create-smart`. The `kitesforu-frontend/CLAUDE.md` rule #10 (`"'/create' and '/create2' are redirects"`) is now partially stale — update when next editing that file.
- Pre-existing test failures on kitesforu-api `main` (TestFilterConstants, test_publish_job_includes_language_from_request, TestRefundAbuseDetection::test_prevent_double_refund) — NOT introduced by this session; left as-is. Flag during Phase B audit.
