# R2 — Episode Feedback Capture (Free-text + Thumbs, Web + Car Mode)

**Status**: DONE — write-path shipped end-to-end (course detail + Car Mode button + Car Mode voice). Read-path is stubbed as proposed; surfacing on creator dashboards is a future follow-up.
**Priority**: P1 (write-path now, read-path incremental)
**Effort**: ~2 hours actual (down from the ~3-4 day estimate — the proposal was thorough enough that the implementation was mostly typing)
**Origin**: Product-owner note, 2026-04-18 drive-test review.

## Implementation summary (2026-04-20)

**Shipped** (4 PRs, all merged):
- `kitesforu-schemas` PR #69 (v1.49.0) — `EpisodeFeedback` (permissive Firestore read model) + `EpisodeFeedbackRequest` (strict inbound, 200-char note cap, pattern-typed rating / sentiment / source enums).
- `kitesforu-api` PR #243 — `POST /v1/courses/:id/episodes/:n/feedback` (upsert partial patches, 30/min rate limit, Pydantic 200-char cap, server-side control-char strip, course-existence check). `GET .../feedback/me` for cross-device hydration. Stored under `courses/{id}/episodes/{n}/feedback/{user_id}` — one doc per (user, course, episode), trivial to aggregate later.
- `kitesforu-frontend` PR #462 — new `lib/api/episode-feedback.ts` typed client; `components/courses/detail/EpisodeFeedback.tsx` rewritten to own its state, hydrate from `.../feedback/me`, and submit partial patches for bucket / thumbs / 200-char note (600ms debounce). Dropped the useless `useState` + prop-drilling in the page.
- `kitesforu-frontend` PR #463 — Car Mode thumb buttons (left overlay, course-scoped), plus `'like'` / `'dislike'` voice commands with natural-language aliases ("i like this", "thumbs up", "love it", "this is bad", etc.). All Car Mode routes share one `postCarModeFeedbackRef` so `source` attribution is consistent (`car_mode` vs `car_mode_voice`).

**Variations from the proposal**:
- Free-form topic Car Mode sessions have no persistence target (no `course_id`), so the thumb buttons are hidden there. Proposal didn't require coverage but it's worth calling out — a `session_feedback` store could backfill later.
- Note submit debounced to 600ms instead of 500ms (proposal said "debounced 500ms") — the extra 100ms noticeably reduced accidental submits on mobile without feeling laggy.
- Sentiment "clear" affordance is client-only (tapping the same thumb again de-toggles locally). Proposal left this open; adding a server-side clear action was over-engineering for v1.

**Verification**:
- All PRs CI-green.
- End-to-end typed path: Pydantic request → Firestore doc → Pydantic read → TypeScript client → React widget.
- Beta listen-through pending by product owner.

**Deferred (read-path, not blocking closure)**:
- Creator dashboard summarising feedback per course (feeds into the regen hint loop from bug 69E54B54).
- Post-hoc moderation batch job reading the sub-collection and flagging abuse — schema is ready but no worker yet.
- Free-form topic Car Mode feedback target (see Variations above).

---


## Problem

The course-detail page already renders a "How was this episode?" widget with four buckets (Too easy / Just right / Too hard / Not relevant). It looks like feedback, but it is not. It is wired to React state only — nothing is POSTed, nothing is persisted. Car Mode has no feedback surface at all. We are quietly throwing away every signal the learner gives us.

> **Product-owner note (2026-04-18):**
> "you get feedback per episode. This is awesome, but i hope we are saving it somewhere properly for future use. But we should also provide some free text or something restricted to certain small text limit like max 2-3 lines. And also expose something like this like thumbs up or thumbs down in car mode as well"

## Evidence

- Widget renders from `components/courses/detail/EpisodeFeedback.tsx:20-61` with the four `FeedbackRating` buckets declared at `EpisodeFeedback.tsx:5` and the label strings at `EpisodeFeedback.tsx:13-18`.
- It is mounted inside the episode card at `components/courses/detail/EpisodeList.tsx:184-190` (gated on `episode.audio_url` + completed status).
- The only handler wiring is in `app/courses/[courseId]/page.tsx:30-33`:
  ```
  const [episodeFeedback, setEpisodeFeedback] = useState<Record<number, FeedbackRating>>({})
  const handleEpisodeFeedback = useCallback((episodeNumber, rating) => {
    setEpisodeFeedback(prev => ({ ...prev, [episodeNumber]: rating }))
  }, [])
  ```
  No `fetch`, no `apiClient`, no Firestore write. The value dies on unmount.
- API has no episode-feedback route. `ls kitesforu-api/src/api/routes/` shows `courses/`, `car_mode.py`, `podcasts/`, etc. — none contain a feedback handler. A repo-wide grep for `episode_feedback|too_easy|just_right` in `kitesforu-api` returns zero matches.
- Car Mode has no feedback surface. `kitesforu-api/src/api/routes/car_mode.py:50-479` defines 7 handlers (session create/stream, question, delete, etc.) — none touch feedback. `app/car-mode/page.tsx:225-264` voice-command switch handles play/pause/stop/next/previous/skip/etc. — no like/dislike case.
- The shared `CourseEpisode` model at `kitesforu-schemas/src/kitesforu_schemas/models.py:827-868` has no `feedback` field.

**Net: the feedback UI is theatre today. Write-path is missing everywhere, read-path does not exist.**

## Proposal

Add a real, minimal feedback subsystem that is additive (no change to existing buckets) and shared between the course page and Car Mode.

### Scope

1. **Persist the four-bucket rating** that the existing widget already collects.
2. **Add a ≤200-char free-text note** under the bucket row on the episode card (2–3 visual lines, hard limit enforced client + server).
3. **Add thumbs-up / thumbs-down** to the episode card AND to Car Mode (visible button in the Drive top bar + voice commands `like` / `dislike`).
4. **Single schema** shared by all three surfaces. One collection, one endpoint.
5. **Write-only for v1.** Read-path is stubbed but not surfaced in UI — see Roadmap.

### Schema (Firestore + schemas package)

New sub-collection: `courses/{course_id}/episodes/{episode_number}/feedback/{user_id}`.
Document-per-user-per-episode (idempotent upsert; latest wins). Storing under the episode — not a flat top-level collection — keeps security rules trivial and aggregation cheap.

```python
# kitesforu-schemas/src/kitesforu_schemas/models.py (new)
class EpisodeFeedback(BaseModel):
    model_config = ConfigDict(extra='ignore')  # permissive read model

    user_id: str
    course_id: str
    episode_number: int = Field(..., ge=1)
    rating: Optional[Literal['too_easy','just_right','too_hard','not_relevant']] = None
    sentiment: Optional[Literal['up','down']] = None   # thumbs
    note: Optional[str] = Field(None, max_length=200)  # free text, 2-3 lines
    source: Literal['course_detail','car_mode','car_mode_voice'] = 'course_detail'
    created_at: datetime
    updated_at: datetime
```

All three fields (`rating`, `sentiment`, `note`) optional so any surface can submit a partial update. `EpisodeFeedbackRequest` is the strict inbound form (same shape, required user_id derived from Clerk, `note.max_length=200`). Bump `pyproject.toml` version on this change — three downstream repos pick it up on redeploy.

### API

New route file: `kitesforu-api/src/api/routes/courses/feedback.py` registered under the existing `courses` router.

- `POST /v1/courses/{course_id}/episodes/{episode_number}/feedback`
  Body: `{ rating?, sentiment?, note? }`. Upsert into the sub-collection with `user_id` from auth. Returns the persisted doc.
- `GET /v1/courses/{course_id}/episodes/{episode_number}/feedback/me`
  Returns the caller's own feedback doc (used by client to show "Thanks for the feedback" state across devices).
- Rate-limit via existing SlowAPI decorator: 30/min per user. Reject `note` > 200 chars at Pydantic layer. Strip control chars server-side.

Car Mode reuses the same endpoint with `source='car_mode'` or `source='car_mode_voice'`. No new car-mode-specific feedback route.

### Frontend

**Course detail (`components/courses/detail/EpisodeFeedback.tsx`):**
- Keep the four-bucket row unchanged.
- Add a collapsed "Add a note" affordance below it — expands to a `<textarea maxLength={200} rows={2}>` with live char counter (e.g. "42 / 200"). Submit button disabled until content non-empty.
- Add two compact thumb buttons (👍/👎) to the right of the bucket row.
- On any of the three interactions, call a new `lib/api/episode-feedback.ts#submitEpisodeFeedback(courseId, epNum, patch)` helper — debounced 500ms for `note` keystrokes, immediate for rating/sentiment clicks.
- Replace the current local-state-only `handleEpisodeFeedback` in `app/courses/[courseId]/page.tsx:30-33` with the helper call; hydrate initial state from `GET …/feedback/me` on course load.

**Car Mode (`app/car-mode/page.tsx`):**
- Add two top-bar icon buttons (thumb-up / thumb-down) that call the same helper with `source='car_mode'` and the currently-playing `episode_number` (already available via `drive.currentEpisode`).
- Extend `VoiceCommand` type in `hooks/useVoiceCommands.ts:36-53` with `'like' | 'dislike'`. Add aliases in the existing command-alias map: `like` ⇐ {"i like this","thumbs up","this is good","loved it"}; `dislike` ⇐ {"i don't like this","thumbs down","this is bad","not good"}.
- In the voice-command switch at `app/car-mode/page.tsx:225-264`, add cases that POST feedback with `source='car_mode_voice'` and TTS-speak "Noted — thanks." / "Got it, thumbs down." (keeps eyes-free UX honest).

### Privacy, safety, abuse

- **Hard char limit (200)** enforced Pydantic-side AND input `maxLength` — survives cURL.
- **No profanity filter in v1** — decision: defer to server-side post-hoc moderation (batch job reads the sub-collection, flags). Noting free-text abuse here rather than blocking writes keeps the fast path simple and avoids false positives on legit frustration ("this episode sucked — too fast").
- **No PII prompting** — helper text: "Tell us why in a line or two. Don't include personal info."
- **Auth required** (`Depends(get_current_user)`); anonymous feedback rejected.
- **Idempotent upsert on (user, course, episode)** — one doc per learner per episode, resubmits overwrite, no spam amplification.
- **Firestore rules**: user can read/write only their own feedback doc. Creator of the course can read aggregates via a Cloud Function (read-path roadmap item).

### Read-path roadmap (deliberately out of v1)

v1 is write-only. Aggregated reads land in three later drops:

1. **Creator dashboard (R2.1):** per-episode tile showing rating histogram + thumb ratio + last 5 notes. Aggregated via Firestore `count()` + `orderBy(created_at desc).limit(5)`.
2. **Course ranking / recommendations (R2.2):** low-thumb-ratio episodes demoted in "for you" shelves; `too_hard` + `not_relevant` buckets feed the difficulty calibrator.
3. **Regeneration hint loop (ties into Bug 69E54B54's episode edit flow):** when an episode crosses thresholds (≥3 thumbs-down OR ≥2 `too_hard` with notes), surface a creator prompt "Regenerate with these learner notes as edit hints?" — notes piped as `user_feedback` context into the episode regeneration worker.

Explicitly **not** built in v1 — but the sub-collection schema is shaped to make all three trivial additions.

## Acceptance criteria

1. Clicking any of the four buckets on a completed episode persists a Firestore doc under `courses/{id}/episodes/{n}/feedback/{uid}` with `rating` set.
2. Typing a 200-char note and submitting persists the doc with `note` set; 201st char is blocked in UI and rejected by API with 422.
3. Clicking thumb-up OR thumb-down on the episode card persists `sentiment='up'|'down'` on the same doc (upsert, not a second doc).
4. Reloading the course page shows the previously-chosen bucket, thumb state, and note restored from `GET …/feedback/me`.
5. Thumb-up / thumb-down buttons are visible in Car Mode top bar while an episode plays; tapping either persists with `source='car_mode'` and the currently-playing `episode_number`.
6. Saying "I like this" / "this is bad" (or any configured alias) while in voice-command mode persists with `source='car_mode_voice'` and TTS-confirms. Works hands-free.
7. Unauthenticated POSTs return 401. Other users cannot GET/overwrite your feedback doc (verified by Firestore rules test).
8. Rate limit: 31st write/min from same user returns 429.
9. Resubmitting feedback updates the existing doc (`updated_at` bumps, `created_at` stable) — verified by Playwright flow that rates, notes, thumbs, then re-rates.
10. Playwright run on `beta.kitesforu.com` completes: sign in, play an episode, rate + note + thumb, reload, see persisted state, open Car Mode, voice-say "I like this", confirm Firestore doc via debug page.

## Risks + mitigations

- **Spammy free-text** → 200-char cap + auth + rate limit + post-hoc moderation batch. No user-facing moderator UI in v1.
- **Voice-command false positives** ("I like this episode format in general" → unintended thumb-up) → only match when utterance is short (≤6 words) and matches an alias phrase near verbatim; log the raw utterance on the doc for audit.
- **Schema churn in `kitesforu-schemas`** → permissive `ConfigDict(extra='ignore')`; bump version; all three consuming repos redeployed (documented in R2 deploy checklist).
- **UI crowding on small episode cards** → note field is collapsed-by-default; thumbs are 28px icons, same line as buckets.
- **Feedback widget silently regresses again** → add a Jest test that asserts `EpisodeFeedback.onSubmit` triggers a network call; add a Playwright flow to CI so "theatre mode" cannot return.

## Out of scope (v1)

- Creator dashboard UI (R2.1).
- Course ranking / recommendation use of feedback (R2.2).
- Auto-regeneration based on feedback thresholds (ties to Bug 69E54B54, separate PR).
- Per-episode aggregated counters on the learner-facing card.
- Feedback on courses as a whole, on individual segments, or on Q&A answers.
- Profanity filter at write time (moderation is post-hoc, manual review).
- Localization of voice-command aliases beyond English (the existing `VoiceCommand` aliases are English-only today).
