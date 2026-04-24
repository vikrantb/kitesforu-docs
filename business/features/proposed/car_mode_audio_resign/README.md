# Car-Mode Audio Re-Sign — Layer 5b Follow-up

**Status**: PROPOSED
**Priority**: P1 (completes the 2026-04-23 horror-podcast audit; studio + library already self-heal — car-mode is the last uncovered consumer)
**Affected repos**: kitesforu-api (new endpoint + car_mode signing service), kitesforu-workers (confirm `gcs_path` write on car-mode segments), kitesforu-frontend (thread `sessionId`/`segmentIndex` through drive hooks + generalize `resignSegmentAudioUrl`)
**Grounding**: Layer 5b of the 2026-04-23 7-agent horror-podcast audit. Layer 5 (podcast studio + library) shipped in api #286 + workers #315 + frontend #586 + #587; see `business/features/done/audio_signed_urls/README.md`. This proposal extends the same pattern to the car-mode brainstorm path.

---

## 1. One-paragraph thesis

Car-mode brainstorm sessions produce audio segments that live under the `car_mode_sessions` Firestore collection (not `podcast_jobs`). The shipped Layer 5 re-sign endpoint (`GET /v1/podcasts/{job_id}/segments/{segment_index}/audio-url`) cannot resolve a car-mode session because it reads only `podcast_jobs`. Meanwhile the car-mode drive player shares `useAudioPlayer` with studio + library, so its retry mechanism already exists — it just lacks the `jobId`/`segmentIndex` inputs that `useDriveEpisodes` never threads through, and it lacks a re-sign endpoint that knows how to look under `car_mode_sessions`. This proposal (a) confirms or adds `gcs_path` writes in the car-mode segment producer, (b) adds a sibling re-sign endpoint scoped to `car_mode_sessions`, and (c) generalizes the frontend re-sign helper + drive-mode plumbing so car-mode episodes self-heal stale signed URLs identically to studio + library.

## 2. Root cause (evidence, not speculation)

- `kitesforu-frontend/hooks/drive/useDriveEpisodes.ts:29-58` — AudioEpisode objects built for drive-mode course/interview_prep/podcast paths set `{ id, title, subtitle, audioUrl, durationSeconds }` only. No `jobId`, no `segmentIndex`. Layer 5's retry fires only when both are populated (`lib/audio-url-resign.ts` early-returns null otherwise).
- `kitesforu-frontend/hooks/drive/useDriveSession.ts:200-206` — SSE live-episode construction under brainstorm mode also omits `jobId`/`segmentIndex`. The `sessionId` is already in scope on that hook (it owns the car-mode session creation), so plumbing is trivial there.
- `kitesforu-api/src/api/services/car_mode/session.py:16` — `car_mode_sessions` collection. Segments stored under this doc, not `podcast_jobs`. Layer 5's `/v1/podcasts/{job_id}/...` endpoint cannot reach them.
- `kitesforu-api/src/api/services/car_mode/funnel.py:80-95, 139-159` — car-mode playback of EXISTING podcast/course episodes works because those audio URLs derive from `podcast_jobs` records (and Layer 5 already covers that path). Only fresh car-mode brainstorm audio is uncovered.

## 3. Decisions (explicit, not open questions)

### D1. New endpoint: `GET /v1/car-mode/sessions/{session_id}/segments/{segment_index}/audio-url`

Mirrors the Layer 5 podcast endpoint exactly — auth'd with the same user-ownership check car-mode routes already use, 1-hour V4 signed URL, reads `gcs_path` from the `car_mode_sessions.segments[]` array (preferred) or parses the legacy public URL as fallback.

### D2. Workers: ensure `gcs_path` is written on car-mode segments

Verify where car-mode segment audio uploads happen. If the same `segment_uploader.py` path is used, the Layer 5 work already covers it — trace end-to-end (pipeline-integration rule #1). If car-mode uses a separate uploader, add the same `gcs_path + 7-day signed URL` write.

### D3. Shared signing helper — reuse, don't re-extract

`kitesforu-api/src/api/services/podcasts/signed_url.py` already lives at a generic-enough path. The car-mode service imports from there rather than duplicating. Only the Firestore lookup (collection + doc shape) is car-mode-specific.

### D4. Frontend: generalize `resignSegmentAudioUrl`

Current signature: `resignSegmentAudioUrl(jobId, segmentIndex, token)`. New signature accepts an optional `kind: 'podcast' | 'car_mode'` (default `'podcast'`) that routes to the matching endpoint. AudioEpisode gains an optional `kind` alongside the existing `jobId`/`segmentIndex` — unset means `'podcast'`, preserving current behavior.

### D5. Threading through drive hooks

`useDriveSession` already owns `sessionId`; adding it + `segment_index` to each AudioEpisode it builds is one-line each. `useDriveEpisodes`, for existing-course/existing-podcast paths, should thread the `podcast_job_id` + episode number (when available) so those paths ALSO get re-sign coverage — today they only work because Layer 5 api+workers are already live, but the retry never fires due to missing fields.

### D6. Phased rollout, consumer-first (same shape as Layer 5)

1. **PR 1 (api)**: NEW car-mode re-sign endpoint. Net-new surface, nothing calls it yet, zero risk.
2. **PR 2 (workers)**: verify/add `gcs_path` writes on car-mode segments. No schema repo change expected.
3. **PR 3 (frontend)**: generalize `resignSegmentAudioUrl` + AudioEpisode + drive hooks. Wire the retry path end-to-end for both car-mode brainstorm (live SSE) and drive-mode playback of existing content.

Each PR independently deployable.

## 4. What NOT to build

- **Don't collapse car-mode and podcast into one endpoint.** The Firestore shapes are different enough (segments stored differently, auth model differs) that a polymorphic endpoint would leak complexity into both systems. Two sibling endpoints that share the signing helper is cleaner.
- **Don't migrate existing car-mode sessions.** Same reasoning as Layer 5 — legacy public URLs stay alone; new sessions use signed URLs + `gcs_path`; retry handles the crossover naturally.
- **Don't gate behind a new feature flag.** The Layer 5 `feature_audio_resign` flag is still live; reuse it.

## 5. CI/CD implications

- **API PR** — single repo. Adds an auth'd route; dep_sync unchanged. Redeploys `kitesforu-api` on main push.
- **Workers PR** — may not be needed if car-mode reuses `segment_uploader.py` (verify first). If needed, adds `gcs_path` write + retains Layer 5 fallback behavior.
- **Frontend PR** — single repo. Drive hook changes gated behind existing `feature_audio_resign` runtime check. Playwright verification post-deploy (CLAUDE.md rule 11).

## 6. Acceptance criteria

- [ ] `GET /v1/car-mode/sessions/{session_id}/segments/{segment_index}/audio-url` returns a 1-hour V4 signed URL; 404 if the session doesn't belong to the authed user; 404 if the segment doesn't exist.
- [ ] Endpoint re-uses the Layer 5 `signed_url.py` helper; no duplicated signing code.
- [ ] Car-mode segment producer writes `gcs_path` alongside `audio_url` to every new segment.
- [ ] `useDriveSession` threads `sessionId` + `segment_index` + `kind: 'car_mode'` into every live AudioEpisode.
- [ ] `useDriveEpisodes` threads `jobId` + `segmentIndex` + `kind: 'podcast'` into every existing-content AudioEpisode (courses, interview_prep, podcasts).
- [ ] `resignSegmentAudioUrl` accepts optional `kind` and routes to the matching endpoint; unit tests cover both paths + legacy default.
- [ ] Full unit suite green in each repo; ruff / tsc / dep_sync clean.
- [ ] Post-deploy Playwright on beta: create a car-mode brainstorm session, force a 403 on a streamed segment, verify the retry lands fresh audio and playback continues; visit an existing course in drive mode that's >7 days old, verify the same flow.

## 7. Risk / rollback

| Risk | Likelihood | Mitigation | Rollback |
|---|---|---|---|
| Car-mode uses a separate uploader that doesn't match Layer 5 | Medium | PR 2 investigates first; scope may expand | N/A — PR 2 is additive |
| Threading `kind` into AudioEpisode requires schema updates in other consumers | Low | Default is `'podcast'`, legacy fields untouched | Remove optional field |
| Re-sign endpoint becomes hot (high RPS) on long drives | Low | 1-hour TTL means one call per hour per drive | Shorten TTL (operator tunable) |

Rollback strategy: each PR has an env flag or is additive-only. PR 3 gated by `feature_audio_resign` (same flag as Layer 5).

## 8. Ship log

- **2026-04-24 docs this proposal** — scope captured, Layer 5 closed, car-mode thread isolated
- *(next)* PR 1 api — car-mode re-sign endpoint + service extension
- *(next)* PR 2 workers — confirm/add `gcs_path` on car-mode segments
- *(next)* PR 3 frontend — generalize resign helper + thread through drive hooks
- *(next)* beta Playwright verification

## 9. Sources

- Layer 5 done-log at `business/features/done/audio_signed_urls/README.md`
- `kitesforu-frontend/hooks/drive/useDriveEpisodes.ts`, `useDriveSession.ts` — current AudioEpisode shape (no jobId/segmentIndex)
- `kitesforu-frontend/components/AudioPlayer/useAudioPlayer.ts`, `lib/audio-url-resign.ts` — shipped retry mechanism
- `kitesforu-api/src/api/services/car_mode/session.py` — `car_mode_sessions` collection constant
- `kitesforu-api/src/api/services/podcasts/signed_url.py` — shared V4 signing helper to reuse
- Horror-podcast audit memory `project_car_mode_creative_routing_2026_04_23.md`
