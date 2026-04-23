# R1 Phase 1 — Manual Actions Checklist

**Session:** 2026-04-17 autonomous shipping run
**Status:** R1 Phase 1 code merged end-to-end; some operator steps pending
**Author:** Claude (autonomous)

This runbook lists the manual steps a human operator needs to take to **complete** R1 Phase 1 (and in-flight P2/P3 slices) and **move the codebase forward safely**. Do these in order.

---

## 1. Run the `content_purpose` backfill in prod (blocker)

**Why it matters:** Executors now write `content_purpose` on all NEW content, and list endpoints expose it. But **every existing** Course / Class / Writeup / PodcastJob in Firestore still has no `content_purpose`. Until you run this, the `/library` filter pills will undercount — legacy content only appears under "All" and under "Interview Prep" via the `style === 'Interview'` fallback.

### Steps

```bash
cd kitesforu-api

# 1a. Authenticate for Firestore prod access (one-time)
gcloud auth application-default login

# 1b. Dry run first — prints what WOULD change, writes nothing
python scripts/backfill_content_purpose.py --dry-run

# 1c. Review the log. Expect ~N lines per collection with
#     ``<collection>/<doc_id> -> content_purpose=<value>`` lines.
#     If the numbers look sane, run for real:
python scripts/backfill_content_purpose.py --apply
```

### Classification rules the script applies

| Collection | Rule |
|---|---|
| `courses` | `content_purpose = "interview_prep"` when `course_type == "interview_prep"`, else `"course"` |
| `classes` | `"class"` |
| `writeups` | `"writeup"` |
| `podcast_jobs` | `"podcast"`, except docs with a `parent_course_id` (course-episode jobs stay untagged — the parent course's classification is what the library shows) |

### Safety

- Script is idempotent — re-running only writes docs missing the field or with a blank value.
- Uses `doc.reference.update({...})` so no other fields are touched.
- **Does not** backfill any other field.

---

## 2. Verify the unified library on beta (visual)

CI already curl-verified the redirects — we need a real human look at the new `/library` page once you're signed in.

### Desktop checklist

1. Sign in at `https://beta.kitesforu.com`.
2. Navigate to `/library`. Verify:
   - [ ] Content you've created appears
   - [ ] Filter pills show counts (All, Interview Prep, Audio Series, Classes, Writing)
   - [ ] Clicking "Interview Prep" filters to interview-prep courses only — **after backfill runs**
   - [ ] Search + Sort work
3. In the browser address bar, type `/courses`. Verify it 302-redirects to `/library?type=course` and the **Audio Series** pill is pre-selected.
4. Repeat with `/classes`, `/writeups`, `/activity`, `/interview-prep/hub`.

### Mobile checklist

1. Open `https://beta.kitesforu.com` on iOS Safari (or Chrome Android).
2. Verify:
   - [ ] Bottom navigation bar is fixed at the bottom with 5 tabs
   - [ ] **+Create** button is the visually elevated center tab
   - [ ] Tapping Library navigates to `/library`
   - [ ] Safe-area padding on a notched device is honored (bar doesn't sit under the home indicator)
   - [ ] The old hamburger menu is gone

### Dark-mode checklist

1. Click the sun/moon icon in the desktop Navbar (top-right, next to the avatar).
2. Verify:
   - [ ] Navbar, BottomNav (mobile), `/library`, and cards all render cleanly in dark
   - [ ] Refresh the page — theme persists, no white flash
3. **Known deferred:** course / class / writeup / studio / smart-create pages are NOT fully dark-styled yet. Seeing light surfaces on those pages is expected.

### Car Mode button

1. Open any course detail (`/courses/{id}`) and start an episode.
2. Expand the FullPlayer (click the mini-player info area).
3. Verify:
   - [ ] The car icon button is visible in the secondary controls
   - [ ] Clicking it opens `/drive?contentType=course&contentId={id}&episodeIndex=N` with N = current episode index
4. Repeat on `/classes/{id}` — Car Mode button should now appear (it didn't before this session).

---

## 3. Legacy cleanup (after one stable prod cycle)

The following files are unreachable after the D3 redirects landed but still in the repo. Delete them in a single chore PR **after ~1 week in prod with no rollback**:

```
kitesforu-frontend/app/courses/page.tsx
kitesforu-frontend/app/classes/page.tsx
kitesforu-frontend/app/writeups/page.tsx
kitesforu-frontend/app/activity/page.tsx
kitesforu-frontend/app/interview-prep/hub/page.tsx
```

Keep `app/courses/[courseId]/page.tsx` (still used). Keep `app/courses/shared/[token]/page.tsx` (public share route).

---

## 4. Flip redirects 302 → 301

In `kitesforu-frontend/next.config.js`, the five legacy-route redirects are currently `permanent: false` (302). Once the library UX is proven stable (see step 3 cadence), flip each to `permanent: true` (301) so search engines + bookmarks update.

---

## 5. Update stale rule in kitesforu-frontend CLAUDE.md

Rule #10 currently says:

> `/create` and `/create2` are REDIRECTS — all creation flows go through `/create-smart`.

After this session, `/create` is a real landing page (templates + hero). `/create2` still redirects. Update to:

> `/create2` redirects to `/create-smart`. `/create` is the template-picker landing (R1 Phase 3 D1). The AI chat-based creation flow still lives at `/create-smart`.

---

## 6. Deferred work — re-read before next session

| Item | Why deferred |
|---|---|
| **R1 P2 D1** Persistent audio player | Large refactor; iOS Safari autoplay risk; needs dedicated session |
| **R1 P2 D2** Speaker visualization | **BLOCKED** — workers/course-workers don't emit per-segment timestamps to Firestore yet. Multi-repo change (workers + course-workers + schemas) required first. |
| **R1 P2 D3 rest** Q&A button in FullPlayer | Needs a non-Car-Mode-session episode Q&A endpoint + `useEpisodeQA` hook (the Car Mode endpoint currently reads from `car_mode_sessions` doc). |
| **R1 P2 D4 rest** Per-page dark polish | Low-risk add-`dark:`-classes follow-ups on course/class/writeup/studio/smart-create pages. |
| **R1 P2 D5** Sleep timer + offline | `useSleepTimer` exists; surface it in FullPlayer. Offline needs PWA Service Worker + Cache API. |
| **R1 P3 D1 rest** Smart intent + voice in `/create` hero | Smart suggestions need an intent-classifier endpoint that isn't exposed yet. |
| **R1 P3 D2** Personalized home dashboard | Not started. Needs activity feed hook + persona-adaptive quick actions. |
| **R1 P3 D3** Smart-Create clarification question | Not started. Needs `clarify_content_purpose` SSE event in `services/smart_create/intake.py` + `useSmartCreateChat` handler. |
| **R2 Phase 1** Content quality & voice | Not started. 172-line spec in `business/features/proposed/`. |
| **R2 Phase 2** Interview prep polish | Not started. 162-line spec. |
| **R3** Platform growth & distribution | Not started. 214-line spec. |
| **Phase B** Audit `done/` features × 7 issues each | Not started. 18 done/ items. Per user ordering: P0 → P1 → P2/P3 → systems. |

---

## 7. Reference — what shipped + where to see it

| PR | Repo | Revision | Scope |
|---|---|---|---|
| #66 | kitesforu-schemas | v1.46.0 (Artifact Registry) | `content_purpose` field on Course/Class/Writeup/PodcastJob + list items |
| #237 | kitesforu-api | kitesforu-api-00481 | Executor writes + list endpoints expose + backfill script |
| #334 | kitesforu-frontend | kitesforu-frontend-00453 | R1 P1 D1 — `isInterviewPrepCourse` prefers `content_purpose` |
| #335 | kitesforu-frontend | kitesforu-frontend-00454 | R1 P1 D2 — unified `/library` page |
| #336 | kitesforu-frontend | kitesforu-frontend-00455 | R1 P1 D3 — 5 redirects + simplified Navbar + mobile BottomNav |
| #337 | kitesforu-frontend | kitesforu-frontend-00456 | R1 P2 D4 slice — dark mode infra + Navbar/BottomNav/Library |
| #338 | kitesforu-frontend | kitesforu-frontend-00457 | R1 P2 D3 slice — Car Mode from class + studio + `episodeIndex` |
| #339 | kitesforu-frontend | deploying at handoff | R1 P3 D1 slice — `/create` landing page with templates + hero |

All eight PRs squash-merged and branches deleted.

---

## 8. Known caveats

- Pre-existing test failures on `kitesforu-api main` (`TestFilterConstants`, `TestPublishJobToQueue::test_publish_job_includes_language_from_request`, `TestRefundAbuseDetection::test_prevent_double_refund`). Not introduced by this session. Flag during Phase B.
- Mobile users can't toggle theme yet — `<ThemeToggle>` is `hidden md:inline-flex`. Surface it in the Profile tab settings once that tab exists.
- The existing `legacy list-page` source files live alongside the new redirects. Harmless, but a cleanup item (step 3).
- Redirects are 302 until proven in prod (step 4).
