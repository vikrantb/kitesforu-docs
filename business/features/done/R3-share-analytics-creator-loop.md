# R3 — Share Analytics Creator Loop

**Status**: DONE — course loop shipped end-to-end. Class / writeup / interview_prep share-stats are deferred (their sharing APIs don't exist yet).
**Priority**: P2
**Effort**: ~30 minutes actual (down from the ~3 day estimate — the design turned out much simpler than the proposal's "aggregate from analytics event table" path, because no such table exists; counters-on-doc is lighter + lives next to the data).
**Origin**: 2026-04-19 strategy pass — option #6 (close the sharing feedback loop)

## Implementation summary (2026-04-20)

**Shipped** (2 PRs):
- `kitesforu-api` PR #244 — `GET /v1/courses/shared/{token}` now fires an `asyncio.create_task(_record_shared_view)` that atomic-increments `shared_view_count` + stamps `shared_last_view_at` on the course doc. New `POST /v1/courses/shared/{token}/track-play` (public) for plays. New `GET /v1/courses/{id}/share-stats` (owner-only via existing service auth) returns `{views, plays, last_view_at, last_play_at}`.
- `kitesforu-frontend` PR #464 — `lib/api/share-stats.ts` typed client (transport-safe: both helpers return null / swallow errors — share stats are soft signals). `components/sharing/ShareStatsBadge.tsx` (compact pill + full sentence variants, auto-hidden when counters are zero). Library course cards render the compact badge. Shared viewer calls `trackSharedCoursePlay` on episode play.

**Variations from the proposal**:
- **No analytics event table**: the proposal assumed `shared_content_view` / `shared_content_play` flowed into an aggregation store. Grep of the API repo returned zero matches — those are client-side `console.debug` events today. Rather than build the event pipeline, we put counters directly on the course doc (Firestore atomic `Increment`). Lighter, same read latency, good enough for a vanity metric.
- **60s cache skipped**: single Firestore read is cheap and the SWR hook on the frontend already dedupes across library renders. Revisit if p50 read latency ever matters.
- **Share modal receipt + detail-page full sentence**: only the library card badge is wired in v1. `ShareStatsBadge variant="full"` exists and is ready for a one-line drop-in on those surfaces.

**Deferred follow-ups**:
- Wire full-variant sentence into the course detail page + `ShareContentModal` footer ("You've shared this to N destinations · viewed X times · played Y times").
- Extend to class / writeup / interview_prep — each needs its own sharing API first; the frontend component is kind-agnostic.
- Per-recipient / per-episode breakdown (proposal out-of-scope, still out of scope).

**Verification**:
- CI green on both PRs.
- Beta listen-through pending by product owner (share a course, view from incognito, confirm counter).

---


## Problem

R3 D2 shipped the full share-and-track instrumentation: `shared_content_view` and `shared_content_play` analytics events fire when recipients hit a shared link and play it (frontend PR #426, merged). The creator never sees the result. From the creator's side, sharing is a one-way black hole — you cannot tell whether the 30 friends you sent it to ever opened it.

This is the single biggest lever on share-driven virality: creators who see their share land are 3–5× more likely to share again.

## Proposal

Aggregate the existing `shared_content_view` + `shared_content_play` events per `content_id` and surface a compact badge on every piece of content the creator owns — on the library card, on the detail page, and in the Share modal itself.

## Why this option

- **Pure win on already-shipped analytics**: events flow today. Only the aggregation + surfacing is missing.
- **Changes behavior, not mechanics**: creators get feedback, not new work.
- **Cheap**: one aggregation endpoint, one badge component, no new events, no new tables.
- **Compounds with R3 D2**: without this, D2 is half-shipped (instrumented but invisible to the creator).

## API — new endpoint

`GET /v1/content/:id/share-stats` → `{ views: int, plays: int, last_seen_at: ISO8601 | null }`

Implementation: read the existing analytics events for the `content_id`, filtered by `event_type in ('shared_content_view', 'shared_content_play')`, group and count. Cache result for 60s per content_id (share stats do not need to be real-time).

Authorization: only the owner of the content sees the stats. Enforce via the same owner check used by `GET /v1/content/:id`.

## Frontend — three surfaces

### 1. Library card badge
Compact pill on the content card when `views > 0`: `👁 12 views · ▶ 7 plays`. Hidden when stats are zero to avoid cluttering new content.

### 2. Detail-page row
Full row in the content detail page: `This has been viewed 12 times and played 7 times — last activity 2h ago`. Renders in the analytics strip that already shows duration / episode count.

### 3. Share modal receipt
Most impactful placement. After the user hits "Copy link" or "Share", surface a small footer: `You've shared this to 3 destinations · viewed 12 times · played 7 times`. Makes the feedback loop immediate.

## Implementation

### API
- `kitesforu-api/src/api/routes/content.py` — add `get_share_stats(content_id)` handler
- `kitesforu-api/src/api/services/analytics/share_stats.py` — new module, aggregates from analytics event table
- `kitesforu-api/tests/unit/test_share_stats_aggregation.py` — ownership check + zero/non-zero/cached paths

### Frontend
- `kitesforu-frontend/lib/api/share-stats.ts` — fetcher + SWR hook
- `kitesforu-frontend/components/share/ShareStatsBadge.tsx` — pill component (compact + full variants)
- Wire badge into `AudioContentCard`, content detail `AnalyticsStrip`, and `ShareContentModal`

## Privacy

- Aggregate counts only. Never surface who viewed — only the count.
- No IP, user agent, or geolocation surfaced. If it is in the event payload today, strip it at the aggregation layer.
- Same owner-only visibility rule as the content itself.

## Acceptance criteria

- [ ] `GET /v1/content/:id/share-stats` returns `{views, plays, last_seen_at}` with owner-only auth
- [ ] 60s cache layer prevents hot-loop on library list renders
- [ ] Zero-state hidden on library card (no stats = no badge)
- [ ] Non-zero state shows pill on library card, full row on detail, footer on ShareContentModal
- [ ] Unit tests for aggregation + auth + cache
- [ ] Playwright test: share a piece of content, simulate a recipient view + play, reload owner view, assert badge appears
- [ ] Beta-verify: share a course to self in a second browser, confirm creator sees +1 view

## Out of scope

- Who viewed (recipient identity) — privacy boundary, separate proposal if ever wanted
- Geographic breakdown
- Time-series charts (compact text stats only for v1)
- Creator-level analytics dashboard (separate R3 D3 scope)

## Key file references

- Analytics events shipped by PR #426: `kitesforu-frontend/lib/analytics.ts` (`trackSharedContentView`, `trackSharedContentPlay`)
- Public share route: `kitesforu-frontend/app/courses/shared/[token]/page.tsx`
- Share modal: `kitesforu-frontend/components/share/ShareContentModal.tsx`
- Content card: `kitesforu-frontend/components/cards/AudioContentCard.tsx`
