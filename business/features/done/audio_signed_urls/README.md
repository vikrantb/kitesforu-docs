# Audio Signed URLs — Replace Bucket-Public-Read with V4 Signed URLs

**Status**: PROPOSED
**Priority**: P0 (real user-reported playback failure on beta)
**Affected repos**: kitesforu-workers (producer), kitesforu-api (read-side re-sign), kitesforu-frontend (follow-up)
**Owner directive**: 2026-04-23 user-reported "Failed to play audio. The file may be unavailable." banner on an in-flight car-mode session; fix the class of bug completely rather than whack-a-mole.
**Grounding**: Layer 5 of the 7-agent horror-podcast audit (agent 6). Complements layers 1–4 which shipped as PRs #313, #283, #284, #60, #314, #285. This is the last remaining layer of that audit.

---

## 1. One-paragraph thesis

Every episode audio URL we emit today is a raw public `https://storage.googleapis.com/{bucket}/{blob}` link that depends on bucket-wide uniform public read ACLs being correct. The 2026-04-23 user report ("Failed to play audio. The file may be unavailable.") is the recurring tell of that dependency — any permission drift (uniform-bucket-access toggled, a sub-prefix without the expected IAM, a region-specific propagation lag, a CDN cache mismatch) produces 403s that manifest as a vague player error. Meanwhile the same workers repo already has a working V4 signed-URL helper shipping to production (`workers/common/debug_artifacts.py:252`) and the Cloud Run service account already has `roles/iam.serviceAccountTokenCreator` (otherwise debug artifacts would be broken). The infra is already there; we just haven't applied it to the audio URLs users actually listen to. This proposal extracts the existing helper into a shared module, flips `segment_uploader.py` to produce signed URLs, and adds a re-sign endpoint so library playback survives past the per-URL TTL.

## 2. Root cause (evidence, not speculation)

From agent 6 and code inspection:

- `kitesforu-workers/src/workers/stages/audio/segment_uploader.py:125-127` — audio URLs are constructed as `f"https://storage.googleapis.com/{bucket}/{blob_name}"`. No signing. Requires the target blob to be publicly readable at the time of the fetch.
- `kitesforu-workers/src/workers/common/debug_artifacts.py:252-314` — already has a complete, production-shipping V4 signed-URL implementation with Cloud Run SA detection, access-token signing, and a local-dev fallback. Confirms the IAM is granted.
- `kitesforu-api/src/api/routes/podcasts/debug.py:140-162` — also uses V4 signed URLs. Same pattern.
- Bucket `uniformBucketLevelAccess` (assumed true on kitesforu-audio; see ops console) means per-object ACLs silently no-op. All reads depend on a bucket-level IAM grant. Any admin action that narrows that grant (including an auto-remediation by Security Command Center) produces 403s on existing audio.

## 3. Decisions (explicit, not open questions)

### D1. V4 signing, not V2

V4 is Google's current signed URL standard. V2 is deprecated. `debug_artifacts.py` already uses V4. Decision: V4.

### D2. TTL = 1 hour on the read-side signing, 7 days on the initial upload-time signing

Two independent signings. Rationale:

- **Upload-time URL (7 days)**: this is the URL emitted through the SSE stream and rendered directly into the `<audio>` element during the session. A listener can pause an episode mid-drive, come back a few days later, and the cached URL still works. Longer TTL than a typical listen session.
- **Read-side re-sign (1 hour)**: for library playback where the URL was written days/weeks/months ago and the upload-time URL is long expired. 1 hour is enough for a single listen session; short enough that a leaked URL from a log aggregator expires before it's meaningfully exploitable.

Both TTLs are constants in the shared helper — tunable in one place.

### D3. Store the blob path in Firestore, not the signed URL

Today Firestore's `segments[]` records carry `audio_url` (the public URL). If we store a signed URL instead, it expires and the record becomes a dangling pointer. If we store the blob path (`podcasts/{user_id}/{job_id}/segment_{i}.mp3`), the record is permanent; the API re-signs on every read.

Decision: store `audio_url` (still populated with a 7-day signed URL for fast path) AND `gcs_path` (blob path). API read endpoints prefer `gcs_path` for fresh signing; fall back to `audio_url` only when `gcs_path` is missing (legacy records).

### D4. New API endpoint for re-signing

`GET /v1/podcasts/{job_id}/segments/{segment_index}/audio-url` → `{"audio_url": "...", "expires_at": "..."}`

Authenticated with the same user session as the rest of `/v1/podcasts/*`. Reads segment from Firestore, calls the shared signing helper, returns a freshly-signed URL.

### D5. Phased rollout, consumer-first

1. **PR 1 (api)** — NEW re-sign endpoint. Consumer-ready, nothing currently calls it, zero risk to ship and deploy. Extracts the signing helper from `workers/common/debug_artifacts.py` into a shared module so both repos use one implementation.
2. **PR 2 (workers)** — `segment_uploader.py` writes `gcs_path` alongside the existing `audio_url`, and flips `audio_url` itself from public to 7-day-signed. New segments carry both fields; old segments keep working via the existing read path.
3. **PR 3 (frontend)** — audio player calls the re-sign endpoint when the cached `audio_url` returns 403, or proactively for library playback where the stored URL is >7 days old. This is the load-bearing consumer of the whole chain.

Each PR is independently deployable with no regression window.

### D6. Graceful degradation on signing failure

Belt-and-suspenders: if the signing call fails at upload time (IAM revoked, network blip, unexpected SDK exception), `segment_uploader.py` falls back to the existing public URL path with a structured warn log. The pipeline never crashes on a signing failure.

### D7. Shared module home

Extract the V4 signing helper into `kitesforu-workers/src/workers/common/signed_url.py` — the cleanest new home. Both `segment_uploader.py` and `debug_artifacts.py` import from there. API side reuses the same contract by calling GCS directly with the same parameters (the pattern is already duplicated at `podcasts/debug.py:150` — kept there for now to avoid cross-repo imports, but the logic matches).

## 4. What NOT to build

- **Don't invalidate every existing public URL in Firestore.** Leave legacy records alone. The API re-sign endpoint derives a fresh URL from `gcs_path` when present, from `audio_url` regex-parse when not.
- **Don't proxy audio through the API.** Adding a proxy hop would 10× the egress bill and introduce a new latency SLA. Signed URLs go direct to GCS; that's the right architecture.
- **Don't add the re-sign endpoint to the SSE stream.** SSE already emits the fresh 7-day signed URL at segment_ready time; a re-sign while the stream is live is redundant. Re-sign is for library replay, not the active session.
- **Don't migrate existing jobs.** A one-shot backfill script would touch every segment record ever written. Not worth it; PR 1+PR 2 make every NEW segment signed, and the re-sign endpoint handles legacy.

## 5. CI/CD implications

- **Workers PR** — local CI hooks catch ruff + py_compile; full pytest runs on GitHub. `kitesforu-worker-audio` + other relevant workers redeploy on merge to main via parallel Cloud Run pipeline. No schema repo change. No bucket IAM change required (already in place).
- **API PR** — single repo. Re-sign endpoint adds an auth'd route; dep_sync unchanged. `kitesforu-api` redeploys on main push.
- **Frontend PR** — single repo. Audio-hook changes gated behind existing runtime-feature verification (CLAUDE.md rule 11: Playwright post-deploy).

## 6. Acceptance criteria

- [ ] Shared V4 signing helper lives at `workers/common/signed_url.py`; `debug_artifacts.py` imports from there instead of duplicating the logic.
- [ ] `segment_uploader.py` writes `gcs_path` to every new segment record AND produces a 7-day V4 signed URL as `audio_url`.
- [ ] Signing failures degrade to the existing public URL with a structured warn log; no pipeline crash.
- [ ] `GET /v1/podcasts/{job_id}/segments/{segment_index}/audio-url` returns a 1-hour V4 signed URL derived from `gcs_path` (preferred) or from regex-parsed `audio_url` (legacy fallback).
- [ ] Endpoint is authenticated with the same user-ownership check the rest of `/v1/podcasts/*` uses (404 if segment doesn't belong to the user).
- [ ] Frontend audio hook calls the re-sign endpoint when the cached `audio_url` returns 403 or is older than 7 days (timestamp on the segment record).
- [ ] Full unit suite green in each repo; ruff / tsc / dep_sync clean.
- [ ] Post-deploy Playwright run on beta: create a new podcast, wait for completion, verify playback works; visit an existing podcast older than a week, verify the re-sign path kicks in and playback works.

## 7. Risk / rollback

| Risk | Likelihood | Mitigation | Rollback |
|---|---|---|---|
| Cloud Run SA loses signBlob permission | Low (currently granted, verified by debug_artifacts shipping) | Upload-time signing falls back to public URL with warn log | No action needed — fallback is automatic |
| 7-day TTL expires mid-listen | Low (listen session << 7 days) | Frontend catches 403 and calls re-sign endpoint | Shorten TTL via the shared constant |
| New endpoint surface adds attack area | Low | Standard Clerk auth + user-ownership check; same pattern as existing `/v1/podcasts/{id}/debug` | Feature-flag the endpoint if needed |
| Signing adds latency to upload | ~50-100ms per segment | Acceptable vs the 3-10s audio generation time | N/A |

Rollback strategy: every PR is a single env-gated flag away from a no-op. PR 1 (re-sign endpoint) is net-new surface — remove the route handler. PR 2 (segment_uploader signing) has a `SIGN_SEGMENT_URLS` env var that short-circuits to the old public-URL path. PR 3 (frontend consumer) is gated behind `feature_audio_resign`.

## 8. Ship log

- **2026-04-24 docs this proposal** — capture decisions, scope, rollout
- **2026-04-24 api #286** — PR 1: `GET /v1/podcasts/{job_id}/segments/{segment_index}/audio-url` re-sign endpoint + shared `signed_url.py` helper (V4 signing + SSRF-safe `gcs_path` resolver). Extracted from `workers/common/debug_artifacts.py`. Auth'd with same user-ownership check as `/v1/podcasts/*`.
- **2026-04-24 workers #315** — PR 2: `segment_uploader.py` writes V4 7-day signed `audio_url` + new `gcs_path` field to every new segment record. Sibling `sign_segment_url(blob, ttl_days)` helper returns `Optional[str]` and never raises; signing failures degrade to the existing public URL with a structured warn log. No pipeline crash path.
- **2026-04-24 frontend #586** — PR 3a: `useAudioPlayer` gained `attemptResignRetry` + `resignedEpisodeIdsRef` (one-shot per episode) + `lib/audio-url-resign.ts` helper + 24 unit tests. `AudioEpisode` now optionally carries `jobId` + `segmentIndex`. Retry fires on 403 / `MediaError.MEDIA_ERR_NETWORK|SRC_NOT_SUPPORTED` / `src contains storage.googleapis.com` + URL >24h old.
- **2026-04-24 frontend #587** — PR 3b: StudioPlayer opts into the re-sign path by threading `jobId` + `segmentIndex` into each streamed AudioEpisode from the SSE segments.
- **2026-04-24 beta Playwright verification** — studio + library playback silently self-heal stale signed URLs under manual 403 simulation (dev-flag). Rollout gated behind `feature_audio_resign`.

### Status — 2026-04-24

**DONE** — studio + library podcast playback covered end-to-end.

**Deferred as Layer 5b** — **car-mode drive player**: uses the same `useAudioPlayer` hook but (a) `useDriveEpisodes` does not thread `jobId`/`segmentIndex` onto AudioEpisode objects and (b) car-mode brainstorm audio lives in the `car_mode_sessions` Firestore collection, not `podcast_jobs` — so the existing `/v1/podcasts/{job_id}/...` re-sign endpoint cannot resolve it. A separate proposal (`car_mode_audio_resign/README.md`) scopes the 3-repo fix. Car-mode playback of EXISTING course/podcast audio still works via the shipped path, because those segments live under `podcast_jobs`; only fresh car-mode brainstorm sessions remain uncovered.

## 9. Sources

- Agent 6 finding from the 2026-04-23 7-agent horror-podcast audit (see `project_car_mode_creative_routing_2026_04_23.md` memory)
- `kitesforu-workers/src/workers/stages/audio/segment_uploader.py:125-127` — current public-URL construction
- `kitesforu-workers/src/workers/common/debug_artifacts.py:252-314` — existing V4 signing pattern to extract
- `kitesforu-api/src/api/routes/podcasts/debug.py:140-162` — api-side V4 signing pattern
- Google Cloud Storage V4 signing docs — https://cloud.google.com/storage/docs/access-control/signed-urls
