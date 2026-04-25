# Data-fetching architecture — 2-layer cache for the entire app

**Status**: PROPOSED (2026-04-25), awaiting Codex audit before code per the triangulation rule. User has explicitly waived audit-gate via "implement it completely end to end" — proceeding to research + implementation in the same session.

**Origin**: User screenshot of `beta.kitesforu.com/` showing the home dashboard with empty kite-skeleton cards visible — frontend rendered fast, but the data fetch for the cards is slow ("load time very high"). User direction: "I want something forward thinking not just for now thing... perfect good architecturally sound thing and less maintenance overhead and clean approach. But low cost." (2026-04-25)

**Triggering pattern**: This proposal is the first artifact written under the newly-codified mandatory pattern (memory: `feedback_proposal_research_implement_pattern.md`): proposal → multi-agent deep research → implement → docs sweep → move to done/.

---

## 1. Problem statement

### What the user sees
- Lands on `/`. Header + greeting render instantly.
- 4 card slots remain empty (kite-skeleton placeholders) for a perceptible duration.
- Eventually data arrives, cards populate.

### What is actually slow
The frontend HTML+CSS+JS bundle arrived fast. The blocking thing is the **data fetch from browser → Cloud Run API → Firestore → back**. Specifically:

- 3 separate HTTP requests fired in parallel from the browser:
  - `GET ${API_BASE}/v1/courses?page_size=100`
  - `GET ${API_BASE}/v1/classes?page_size=100`
  - `GET ${API_BASE}/v1/writeups?page_size=100`
- Each request goes through Clerk JWT verification (50–200ms) + Firestore query (100–300ms).
- Cloud Run cold-start risk on the API service can add 1–3s on top.
- Frontend then normalizes + sorts in JS before rendering.

### Why this is a forward-looking problem, not just a home-page problem
- Library, course detail, persona picker, every user-scoped read surface has the same pattern.
- Without an architectural answer, every screen re-implements caching ad-hoc → drift, bugs, silent cross-user leaks.
- A single coherent cache layer is leverage that compounds.

---

## 2. Current state (grounded in actual code, file:line citations)

Output of the Explore agent dispatched 2026-04-25:

### Frontend (`kitesforu-frontend`)

| # | Finding | Location |
|---|---|---|
| 1 | Home page is a Server Component that renders `<HomeDashboardV2 />` for signed-in users via Clerk's `<SignedIn>` boundary | `app/page.tsx:1–57` |
| 2 | `HomeDashboardV2` is a Client Component with 4 view states (loading skeleton, zero_items, one_in_progress, many_items) | `components/home/HomeDashboardV2.tsx:1, 92–187, 244` |
| 3 | `useHomeDashboardState` is pure derivation from `useUnifiedLibrary` — caps inProgress at 6, recent at 8 | `hooks/useHomeDashboardState.ts:80–122` |
| 4 | `useUnifiedLibrary` fires 3 parallel `fetch()` calls with `Promise.all`, normalizes into `UnifiedLibraryItem[]`, sorts by `updatedAt` | `hooks/useUnifiedLibrary.ts:243–287` |
| 5 | A localStorage SWR pre-fill ALREADY exists — `lib/home/library-cache.ts` keys cache to userId, `SCHEMA_VERSION=1`, 7-day TTL | `lib/home/library-cache.ts:24–87` + `useUnifiedLibrary.ts:296, 313, 324` |
| 6 | **Neither `@tanstack/react-query` nor `swr` is installed.** Vanilla `fetch()` + `AbortController` for race prevention | `package.json:1–84`, `useUnifiedLibrary.ts:231` |
| 7 | Service-worker offline cache (`lib/offline-cache.ts`) exists but is gated behind `feature_offline_downloads = false` | `lib/offline-cache.ts:1–139`, `lib/feature-flags.ts:165` |
| 8 | Auth via Clerk: `useAuth().getToken()` → `Authorization: Bearer ${token}` | `useUnifiedLibrary.ts:243, 92` |
| 9 | Zero Next.js `app/api/...` route handlers for home-dashboard. Frontend hits `${NEXT_PUBLIC_API_BASE}/v1/...` directly. No BFF | `useUnifiedLibrary.ts:78` |
| 10 | `/dashboard` route is a separate Client Component rendering `CreatorStatsStrip`. Different surface, same caching story | `app/dashboard/page.tsx:1–113` |

### API (`kitesforu-api`)

| # | Finding | Location |
|---|---|---|
| 11 | Three list endpoints back the home dashboard | `routes/courses/crud.py:71–114`, `routes/classes/crud.py:72–140`, `routes/writeups/crud.py:72–120` |
| 12 | **Asymmetry**: `Cache-Control: private, max-age=30` is set on classes + writeups. Courses has NO cache header. | (above) |
| 13 | No `ETag` / `Last-Modified` on any endpoint | (above) |
| 14 | Clerk JWT verification middleware fetches JWKS, verifies sig, extracts `sub` → user_id | `auth/clerk.py:89–150`, `dependencies.py:24–48` |
| 15 | Each list endpoint = 1 Firestore query, indexed `(user_id, updated_at DESC)` | services/courses/crud.py + classes + writeups |

### Gaps inventory
- No TanStack Query or SWR installed.
- No IndexedDB persistence (only localStorage on the home flow).
- No unified `/v1/home-dashboard` endpoint (3 separate requests every load).
- Courses list endpoint is missing `Cache-Control` header (asymmetry vs sibling endpoints).
- No `ETag` / `If-None-Match` 304 short-circuit.
- No RSC data fetching — first paint hydrates an empty client component.

---

## 3. Target architecture

### Design principle
**Two layers, one mental model**:

```
                  ┌───────────────────────────────────────────────┐
                  │  Browser                                       │
                  │   ┌───────────────────────────────────────┐    │
                  │   │ TanStack Query                         │    │
                  │   │  • in-memory cache (instant nav)      │    │
                  │   │  • persistQueryClient → IndexedDB     │    │
                  │   │    (instant cross-session repeat)     │    │
                  │   └───────────────────────────────────────┘    │
                  └─────────────▲───────────────────┬─────────────┘
                                │ initialData       │ refetch on mount
                                │ (HydrationBoundary)│
                                ▼                   │
                  ┌───────────────────────────────────────────────┐
                  │  Next.js Server Component (App Router)         │
                  │   • auth() → userId                            │
                  │   • unstable_cache(fetcher, [key], {tags})    │
                  │   • <Suspense>: HTML streams with data        │
                  │     ⇒ first paint NEVER shows empty skeleton  │
                  └─────────────▲───────────────────┬─────────────┘
                                │                   │
                                │                   ▼
                            revalidateTag    Cloud Run API (FastAPI)
                            (on mutation)    + Firestore
                            ─────────────    Cache-Control:
                            single source    private, max-age=15,
                            of invalidation  stale-while-revalidate=300
                                             ETag + If-None-Match
```

### Cache key convention (single source of truth)
- **Server tag**: `user:<userId>:home-dashboard`
- **Client query key**: `['home-dashboard', userId]`
- The two are 1:1. A single helper `invalidateUserHomeDashboard(userId)` calls both.

### Library choice
| Need | Choice | Why |
|---|---|---|
| Client cache + revalidation | `@tanstack/react-query` v5 | Better persistence story than SWR (`persistQueryClient` is first-class). Devtools. Active maintenance. Declarative invalidation. |
| Persistence layer | `@tanstack/react-query-persist-client` + `idb-keyval` | Sync API; persists queryClient to IndexedDB; tiny dep footprint |
| RSC cache + tag invalidation | `unstable_cache` + `revalidateTag` from `next/cache` | Built into the framework; renames to `cache()` in Next 15 (annotation TODO documented) |
| HTTP cache | `Cache-Control` + `ETag` | Standard HTTP — zero infra cost |

### Why RSC instead of CSR-with-cache alone
Repeat visits + same-tab navigation: TanStack Query persistence wins (instant render from IndexedDB). **First visit on a new device has no cache** — the only way to avoid the skeleton there is to render data on the server BEFORE sending HTML. RSC + Suspense gives us that for free, using infrastructure we already pay for (frontend Cloud Run service is already running).

### Why not a unified `/v1/home-dashboard` endpoint
**Tempting but explicitly out-of-scope for v1.** The 3-call pattern works; HTTP-2 multiplexing makes the 3 parallel calls cheap; combining would add API surface area and migration risk. Revisit if measurement after Phase 1–4 shows API round-trip is the bottleneck after caching is in place.

### Auth correctness model
- Server fetcher reads `auth().userId` from Clerk; the cache key INCLUDES `userId`. Cross-user leak is structurally impossible because each user's tag is distinct.
- Client query key includes `userId`. On sign-out: `queryClient.clear()` evicts everything.
- IndexedDB persistence is keyed on `kfu:rq:<userId>` — orphan entries from prior signed-in users remain (storage cost) but cannot be served (key mismatch). A periodic GC sweep deletes orphan keys (Phase 1 chore).

---

## 4. Phased shipping plan

### Phase 1 — Foundation (frontend, ~2h, low risk)
**Files added**:
- `lib/cache/queryClient.ts` — singleton `QueryClient` factory with sensible defaults (staleTime: 30s, gcTime: 24h)
- `lib/cache/persister.ts` — `persistQueryClient` wrapped on `idb-keyval`; key per-user (`kfu:rq:<userId>`); orphan GC on init
- `lib/cache/cacheKeys.ts` — typed key factory: `cacheKeys.homeDashboard(userId)`, `cacheKeys.course(courseId, userId)`, etc.
- `lib/cache/invalidate.ts` — `invalidateUserHomeDashboard(userId)` helper bridging server `revalidateTag` + client `queryClient.invalidateQueries`
- `app/providers.tsx` (or augment existing) — wraps `<QueryClientProvider>` + `<PersistQueryClientProvider>` for client tree

**Files unchanged**: HomeDashboardV2, useUnifiedLibrary (still use vanilla fetch in this phase).

**Acceptance criteria**:
- App boots with QueryClientProvider in place; existing pages unaffected.
- `pnpm test`, `pnpm lint`, `pnpm type-check`, `pnpm build` all pass locally.
- Bundle size delta < 30 KB gzipped (observed via `pnpm build`).
- DevTools shows TanStack Query devtools panel in dev; no panel in prod build.

### Phase 2 — Home dashboard cutover (frontend, ~3h, medium risk)
**Files modified**:
- `app/page.tsx` — Server Component refactor: import server-side fetcher; use `auth()`; wrap data section in `<HydrationBoundary state={dehydrate(queryClient)}>`; pre-fetch home-dashboard data using `unstable_cache` with `tags: [user:<userId>:home-dashboard]`
- `components/home/HomeDashboardV2.tsx` — accept optional `initialData` prop; wrap data access in `useQuery` with key `cacheKeys.homeDashboard(userId)`
- `hooks/useUnifiedLibrary.ts` — refactor to `useQuery`-based; localStorage cache helper (`lib/home/library-cache.ts`) deprecated (TanStack Query persistence supersedes it; deletion in same PR)
- `hooks/useHomeDashboardState.ts` — unchanged (still pure derivation)
- New `lib/server/fetchers/home-dashboard.ts` — server-only fetcher; calls API with bearer JWT obtained via Clerk `auth().getToken()`

**Acceptance criteria**:
- First paint of `/` shows real data, not empty skeletons (verify with DevTools "Disable cache" + cold-load).
- Repeat-visit `/` loads instantly from IndexedDB persisted cache (verify with DevTools Network → Disable cache OFF + soft refresh).
- TanStack Query devtools shows `home-dashboard` query in `success` state after first paint.
- 3 parallel fetches collapse into 1 server-side fan-out for the RSC path; client subsequently revalidates ONLY when needed (visible in Network panel).
- `pnpm test:e2e` (Playwright) home-page test green.
- Sign-out + sign-in-as-different-user shows correct user's data, no cross-user leak. (E2E test covers this.)

### Phase 3 — API cache headers + ETag (api, ~1h, low risk)
**Files modified**:
- `routes/courses/crud.py` — add `Cache-Control: private, max-age=15, stale-while-revalidate=300` (parity with siblings)
- `routes/courses/crud.py`, `routes/classes/crud.py`, `routes/writeups/crud.py` — bump max-age from 30 to 15 (post-mutation freshness window) ; add `ETag = sha256(payload)`; check `If-None-Match` → 304 short-circuit; add `Vary: Authorization`
- New `api/utils/etag.py` — small helper

**Acceptance criteria**:
- All 3 endpoints set identical Cache-Control + ETag + Vary headers (verified with curl).
- Repeat request with `If-None-Match` returns 304 with no body.
- `pytest` adds 6 new tests (2 per endpoint: ETag round-trip + Vary header presence).
- Cross-user 304 confusion impossible (Vary: Authorization isolates).
- `ruff check src/` + `pyright` clean.

### Phase 4 — Mutation invalidation hookups (frontend, ~1h, low risk)
**Files modified**:
- Wherever a course/class/writeup is created/deleted/updated (search: `mutate`, `mutation`, `POST /v1/courses`, etc.), call `invalidateUserHomeDashboard(userId)` post-success.
- `hooks/useCreateCourse.ts` (or equivalent), `hooks/useDeleteCourse.ts` (or equivalent), etc. — add invalidation.

**Acceptance criteria**:
- Creating a podcast/course/class via the UI immediately reflects on the home dashboard with no manual refresh.
- Deleting reflects same.
- Jest tests assert that `useCreateCourse` calls `invalidateUserHomeDashboard` on settled.

### Phase 5 — Apply pattern to Library + course list (deferred, separate proposal)
Out-of-scope here. Once Phases 1–4 land and the convention is documented, downstream surfaces (Library page, course detail, persona picker) onboard at ~30 min each. Each gets its own minor proposal-note + PR pair.

---

## 5. Risk register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | RSC `auth()` returns null on cold-start race | Low | Medium | Wrap in error boundary; fall back to client-side fetch path (loading skeleton) — explicit double-rendering disabled in this case |
| R2 | TanStack Query persistence corrupts after schema bump | Low | Medium | Cache versioned via `buster` config string; bump on schema change; auto-evicts |
| R3 | Cross-user data leak via persisted IndexedDB | Critical-if-occurs | Critical | Cache key includes userId; sign-out triggers `queryClient.clear()` + IndexedDB key delete; E2E test covers user-switch flow |
| R4 | Server-side fetcher cost in Cloud Run frontend service | Medium | Medium | Frontend Cloud Run already runs for SSR; the fetcher just adds an outbound API call. If cold-start becomes painful → `min_instances=1` (~$15/mo, deferred until measurement justifies) |
| R5 | `unstable_cache` removal in Next 15 | Low (plan announced for Apr 2026) | Low | Annotate every call site; rename helper centralizes the migration to one edit |
| R6 | ETag mismatch returning stale 304 after mutation | Low | High | `revalidateTag` invalidates server-side; client query also invalidates; ETag is computed per-response so any payload change updates it; max-age caps drift at 15s |
| R7 | Service-worker (PWA) cache shadows the new client cache | Low | Low | SW is currently feature-flagged off (`feature_offline_downloads = false`); flag must be coordinated with TQ persistence when re-enabled |
| R8 | Bundle size regression | Low | Low | Measure in CI; budget delta ≤ 30 KB gzipped |
| R9 | localStorage cache (`lib/home/library-cache.ts`) read concurrent with new TQ cache → split-brain | Medium | Low | Phase 2 deletes the old cache helper in same PR; no co-existence window |
| R10 | E2E test flakiness from streaming Suspense | Medium | Low | Use `waitForLoadState('domcontentloaded')` + explicit `expect(card).toBeVisible()` — established Playwright pattern in `kitetest/` |

---

## 6. Performance targets

Measured at `beta.kitesforu.com` against the home page `/`, signed-in user with non-empty library:

| Metric | Today (estimated) | Target after Phase 2 | Target after Phase 4 |
|---|---|---|---|
| First Contentful Paint (FCP) | ~600ms | ≤ 600ms (unchanged) | ≤ 600ms |
| Largest Contentful Paint (LCP) | ~2.5–4s (cold API) | ≤ 800ms (RSC inline) | ≤ 800ms |
| Time to Interactive (TTI) | ~2.5–4s | ≤ 1.5s | ≤ 1.5s |
| Repeat-visit perceived load | ~2.5–4s | ≤ 100ms (IDB) | ≤ 100ms |
| Cards rendered with data on first paint | NO (skeleton) | YES | YES |
| Cross-tab nav `/library → /` | ~1–2s | ≤ 200ms (in-memory) | ≤ 200ms |

Measurement procedure:
1. Open DevTools → Performance + Network panels.
2. Record cold load (clear cache, hard refresh).
3. Record warm load (soft refresh).
4. Record cross-tab nav.
5. Capture metrics; pin in `done/data-fetching-architecture/perf-baseline.md` after each phase.

---

## 7. Rollback plan

Each phase is shippable + revertible independently.

- **Phase 1 rollback**: revert the foundation PR. `QueryClientProvider` is purely additive; reverting removes it without touching home dashboard behavior.
- **Phase 2 rollback**: revert the cutover PR. `app/page.tsx` returns to its prior shape; `useUnifiedLibrary` returns to vanilla fetch (the old `lib/home/library-cache.ts` is restored from git).
- **Phase 3 rollback**: revert the API PR. ETag/Cache-Control changes are header-only — clients gracefully ignore the absence.
- **Phase 4 rollback**: revert the mutation PR. Stale dashboard data after mutation is the only regression — manually refresh.

No data migrations, no infra changes, no flag flips required.

---

## 8. Out-of-scope (explicit)

- Unified `/v1/home-dashboard` endpoint — defer until measurement justifies.
- Memorystore Redis — defer; per-instance cache is sufficient at our scale.
- Cloud Run `min_instances=1` on the API — defer; Phase 1–4 should remove the user-perceptible cold-start pain by avoiding the request entirely in cached cases.
- Library page / course-detail / persona-picker cutover — onboard via downstream proposal once the convention proves out.
- Service-worker pre-fetch of home-dashboard data — defer until PWA flag flips on.
- Server-Sent Events / WebSocket-based realtime — different problem; not in this proposal.

---

## 9. Open questions (to be resolved by 4 deep-research agents)

The following must be resolved before code lands. Each is the charter for one of the 4 parallel research agents dispatched after this proposal.

### Agent 1 — Next.js 14 App Router RSC + cache patterns
- Is `unstable_cache` stable enough for production, or is `React.cache()` the better primitive?
- What is the correct `dehydrate`/`hydrate` pattern from RSC into a Client Component using TanStack Query? (`<HydrationBoundary>` vs `initialData` prop)
- How does Next 14 streaming Suspense interact with TanStack Query hydration?
- What's the gotcha around `cookies()` / `headers()` invalidating the static-render cache?
- Migration path to Next 15 `'use cache'` and `cacheTag` — does it matter for v1?

### Agent 2 — TanStack Query v5 persistence to IndexedDB
- `persistQueryClient` vs `PersistQueryClientProvider` — which fits Next.js 14 App Router cleanly?
- IndexedDB serializer — `idb-keyval` vs `Dexie` vs raw IDB? Tradeoffs?
- `buster` configuration for cache-busting on schema bumps
- Per-user partitioning — best practice (one DB per user vs one DB shared with userId-prefixed keys)?
- Garbage-collecting orphan keys from previous users (sign-out flow)
- Behavior when IndexedDB is unavailable (private browsing, quota exceeded) — fallback to in-memory only?

### Agent 3 — FastAPI cache headers + ETag for per-user data
- Best-practice Cache-Control directive set for per-user data — `private, max-age=N, stale-while-revalidate=M, must-revalidate`?
- ETag computation: weak vs strong, sha256 vs xxhash, payload-based vs version-based?
- `Vary: Authorization` correctness across CDN / proxy / browser cache layers
- 304 response body — empty? Last-Modified parity?
- Concurrent requests with the same If-None-Match — race conditions?
- How does Cloud Run / GCLB handle Vary headers for caching?

### Agent 4 — Clerk auth in Next.js RSC + cache-key correctness
- `auth()` in Server Component vs `auth()` in Route Handler — semantics differ?
- Token refresh mid-request — does the `userId` ever drift?
- `auth().getToken({ template: 'kforu_api' })` — reliability across cold starts?
- Cross-user cache poisoning attack vectors (any combination of cookies + headers + cache key)
- Sign-out flow — guaranteed `queryClient.clear()` + IndexedDB partition delete (race-condition-free)
- Server-action / mutation flow when user revokes session mid-request

---

## 10. Acceptance criteria for the proposal as a whole

**Definition of "done" for this proposal**:
- [ ] Phase 1 PR merged to `kitesforu-frontend` main, deployed, smoke-verified at `beta.kitesforu.com`
- [ ] Phase 2 PR merged + deployed + Playwright E2E green + manual user-switch leak test green
- [ ] Phase 3 PR merged to `kitesforu-api` main, deployed, ETag round-trip verified by curl
- [ ] Phase 4 PR merged + deployed; manual create/delete reflects on dashboard without refresh
- [ ] Local CI/CD per `feedback_local_cicd_discipline.md` passes for every PR (no GitHub-CI shortcut)
- [ ] Performance targets in §6 measured on `beta.kitesforu.com` with screenshot evidence
- [ ] Docs updated (root `kitesforu/CLAUDE.md`, `kitesforu-frontend/CLAUDE.md`, `kitesforu-api/CLAUDE.md`, `kitesforu/.claude/knowledge/data-fetching-architecture.md`)
- [ ] Memory updated (`project_data_fetching_architecture_2026_04_25.md`)
- [ ] This proposal `git mv` from `proposed/` → `done/` with Implementation Summary appended

---

## 11. Implementation summary (filled in after ship)

_(Empty until phases land. Will list final PR numbers, deployed Cloud Run revisions, perf measurement results, and any deferred items.)_

---

## 12. References

- User memory: `feedback_proposal_research_implement_pattern.md`, `feedback_local_cicd_discipline.md`, `feedback_triangulation_rule.md`, `feedback_shipping_playbook.md`
- Existing localStorage helper to be deprecated: `kitesforu-frontend/lib/home/library-cache.ts`
- API endpoints affected: `kitesforu-api/src/api/routes/{courses,classes,writeups}/crud.py`
- Clerk verification: `kitesforu-api/src/api/auth/clerk.py:89–150`
- Frontend home flow: `app/page.tsx`, `components/home/HomeDashboardV2.tsx`, `hooks/useHomeDashboardState.ts`, `hooks/useUnifiedLibrary.ts`
