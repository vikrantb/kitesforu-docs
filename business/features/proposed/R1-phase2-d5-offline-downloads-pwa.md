# R1 Phase 2 D5 — Offline Downloads (PWA)

**Status**: PROPOSED (awaiting Codex audit before any code per triangulation rule)
**Priority**: P2 (R1-phase2 D5 deferred AC; named in `done/R1-phase2-ux-player-consumption.md` close-out summary)
**Effort**: ~1 week (frontend-only, phased)
**Affected repos**: `kitesforu-frontend` (primary), `kitesforu-infrastructure` (CSP / static asset headers if needed)
**Depends on**: Existing `useAudioPlayer` + `PersistentPlayerHost` (shipped via R1-phase2 D1).
**Origin**: R1-phase2 D5 deferred 3 ACs (download button, offline playback, download badge). Closed phase doc at `business/features/done/R1-phase2-ux-player-consumption.md` names this as the dedicated PWA thread.

---

## 1. One-paragraph thesis

Today `downloadEpisode` (`hooks/useCourseDetail.ts:619-628`) is a one-line anchor-tag click that triggers a browser download to the user's filesystem. It does not register the file with the app, so there's no library badge and no offline playback when the network is unavailable. To close R1-phase2 D5 we add a PWA layer: a Service Worker scoped to the audio domain, a Cache API store keyed by query-stripped audio URL, a Download button that hits the cache (not just the filesystem), and a library badge on `AudioContentCard` when the episode's audio URL is in cache. The audio player already routes through `useAudioPlayer` which can read from cache via `caches.match()` before falling back to network and the Layer 5 re-sign retry.

## 2. Goals / Non-goals

### Goals

1. User can tap a Download button on any course/podcast detail page and get an offline-playable copy of every episode in that series.
2. The library card surfaces a checkmark/badge for each item where audio is fully cached.
3. The audio player plays from cache transparently when offline (or when network is slow).
4. User can clear cache from a settings panel; per-item delete from the library card on long-press / kebab menu.
5. Storage stays bounded — soft-cap at 2 GB per user with explicit confirmation before the cap is exceeded.

### Non-goals

- **Background sync of new episodes** — user must tap Download. Auto-prefetch is a separate later thread (would burn user data without consent).
- **DRM / encrypted offline files** — out of scope; we sign URLs short-lived (V4) and rely on the same model as Layer 5.
- **Cross-device sync of downloads** — each device caches independently. iCloud/Google sync of cache not in scope.
- **Streaming segments cached individually** — initial scope downloads the *final* assembled episode MP3 only, not the segment-by-segment stream. Studio jobs cache only after generation completes.
- **Native iOS / Android app** — PWA only. iOS Safari + Chrome Android coverage.

## 3. Current surface — file:line references

| Concern | Location |
| --- | --- |
| Existing download trigger | `hooks/useCourseDetail.ts:619-628` (anchor-tag .mp3 download) |
| Course detail page wiring | `app/courses/[courseId]/page.tsx:171` (`onDownloadEpisode={h.downloadEpisode}`) |
| Audio player hook | `components/AudioPlayer/useAudioPlayer.ts` |
| Audio URL signing + retry | `lib/audio-url-resign.ts` (Layer 5 self-heal mechanism) |
| Library card | `components/library/AudioContentCard.tsx` |
| Persistent player host | `components/AudioPlayer/PersistentPlayerHost.tsx` |
| Public assets | `public/` (no `manifest.json` or service worker present today) |
| Audio MP3s served from | `public/audio/` (local dev) + GCS signed URLs (prod) |

No PWA tooling installed (no `next-pwa`, no `workbox` packages). This is greenfield.

## 4. Architecture

### 4.1 Service Worker

```
public/sw.js           ← new — scoped to '/'
  - On 'install': skipWaiting()
  - On 'activate': claim all clients
  - On 'fetch': intercept requests matching audio MIME or audio URL pattern
    - cache-first for 'audio-cache-v1', fallback to network
    - never cache range requests (audio playback uses byte-range, complex caching semantics)
    - only cache the *whole* MP3 fetched by an explicit Download request
  - On 'message' (from app):
    - { type: 'cache-episode', url, id } → fetch full URL, store in cache
    - { type: 'evict', id }              → delete from cache
    - { type: 'clear' }                  → delete cache
    - { type: 'get-status', id }         → respond with { cached: bool, size: number }
```

### 4.2 Cache layout

Cache name: `audio-cache-v1` (versioned so a future SW upgrade can purge cleanly).

Each cached entry is keyed by the **query-stripped URL** (so cache hits survive re-signs):

```
key = audioUrl.split('?')[0]   // e.g. https://storage.googleapis.com/.../episode-3.mp3
```

This is safe because GCS object paths are immutable per job. A re-sign produces a different `?Signature=...` but the same path. `cache.match(key, { ignoreSearch: true })` so query-string drift doesn't miss the cache.

### 4.3 IndexedDB metadata

Cache API alone is opaque (no metadata). We mirror cache state in IndexedDB so the library can render badges + storage usage without enumerating Cache contents:

```
db: 'audio-downloads-v1'
store: 'episodes'
  key:    episodeKey (matches cache key, query-stripped)
  value:  {
    episodeKey: string,
    courseId: string,
    episodeId: number | string,
    title: string,
    sizeBytes: number,
    cachedAt: number,
    lastPlayedAt: number | null,
  }
```

This unblocks library badges, "Downloads" panel UI, and LRU-style cache eviction once the soft cap is hit.

### 4.4 React surface

New module: `lib/offline-cache.ts`

```ts
export async function cacheEpisode(ep: AudioEpisode, courseId: string): Promise<void>
export async function evictEpisode(episodeKey: string): Promise<void>
export async function isCached(episodeKey: string): Promise<boolean>
export async function listCached(): Promise<CachedEpisode[]>
export async function totalCacheBytes(): Promise<number>
export async function clearAllCache(): Promise<void>
```

New hook: `hooks/useOfflineDownloads.ts`

```ts
const { downloadEpisode, evictEpisode, downloadStatus, totalBytes, clearAll } =
  useOfflineDownloads()
```

`downloadStatus` is a per-episode dict: `'idle' | 'downloading' | 'cached' | 'failed'`.

### 4.5 Audio player integration

`useAudioPlayer` already supports a one-shot URL re-sign retry (Layer 5). We extend the resolver to check cache first:

```
fetch order:
  1. caches.match(cacheKey, { ignoreSearch: true })
  2. existing audioUrl (signed)
  3. resigned URL (Layer 5 retry)
  4. network failure → show offline error if no cache
```

Cache hit returns a Response object whose `body` is fed to `<audio src=URL.createObjectURL(blob)>`. This works in iOS Safari 16+ and all modern Chrome/Edge/Firefox.

### 4.6 Storage quota guards

- On every `cacheEpisode` call: read `navigator.storage.estimate()`. If projected total > 2 GB, prompt user with "You're using X GB. Continue?".
- LRU eviction: if cap reached AND user confirms continue, evict the oldest `lastPlayedAt` first.
- iOS Safari quota: ~50 MB by default unless user adds the app to home screen, then up to 1 GB. Surface this constraint in the UI ("Add to Home Screen for full offline support" CTA on iOS).

## 5. UI surfaces

### 5.1 Download button

**Where**: course/podcast detail page episode list, FullPlayer secondary controls row.

**State machine** (per episode):
- `idle` → cloud-download icon
- `downloading` → spinner with progress bar (Cache API doesn't expose progress directly; we approximate from `Content-Length` + bytes received via streaming `fetch`)
- `cached` → checkmark badge, tap menu opens "Remove download"
- `failed` → exclamation icon, tap retries

### 5.2 Library badge

`AudioContentCard` reads from `useOfflineDownloads().getCardStatus(itemId)`:
- All episodes cached → solid "Offline ready" pill (emerald)
- Some episodes cached → progress chip "3 of 5 offline"
- None cached → no badge (current behavior)

### 5.3 Settings panel

`/settings/downloads` (new page):
- "Downloaded" section — list of cached items with size, last played, "Remove" button
- "Storage used" — `X GB of 2 GB` progress bar
- "Clear all downloads" button (destructive, confirm modal)

### 5.4 Offline state

Add a thin top banner: "You're offline. Showing downloaded content." Triggered by `navigator.onLine === false`. Library auto-filters to cached items only. Search bar disabled (no API access).

## 6. Phased scope

### Phase 1 — SW + cacheEpisode + status (3 days, low risk)

- `public/sw.js` registered in `app/layout.tsx` via small inline script.
- `lib/offline-cache.ts` core API.
- `useOfflineDownloads` hook.
- Wire into `downloadEpisode` in `useCourseDetail.ts` so the existing button now caches before triggering the OS-level download. Backward-compatible: still gives the user a file they can save, but ALSO populates the cache.
- Library badge MVP: per-item "cached" pill, no progress chip yet.
- Per-item Remove via long-press / kebab menu.

### Phase 2 — Audio player cache-first + offline state (2 days)

- `useAudioPlayer` resolver checks cache first.
- Top banner on offline state.
- Library auto-filter when offline.
- iOS "Add to Home Screen" hint when storage quota is tight.

### Phase 3 — Settings panel + LRU + soft-cap modal (2 days)

- `/settings/downloads` page.
- Storage estimate + LRU eviction.
- Soft-cap modal at 2 GB.
- "Clear all" destructive action.

## 7. Acceptance criteria

- [ ] Service worker registers on first page load; visible in Application → Service Workers panel.
- [ ] Download button on course detail caches the MP3 in `audio-cache-v1`; cache key is query-stripped URL.
- [ ] Reloading the page shows the episode as cached (badge + button state).
- [ ] Disabling network in DevTools → playing the cached episode works.
- [ ] Library card shows "Offline ready" pill for fully-cached items.
- [ ] `navigator.storage.estimate()` is called before each cache; user is prompted at 2 GB.
- [ ] Per-item Remove evicts the cache entry AND removes the IndexedDB record AND updates the badge.
- [ ] Settings → Downloads lists every cached item with size + last played; Clear All wipes both stores.
- [ ] iOS Safari 16+ AND Chrome Android both work for at least 5 episodes cached and played offline.
- [ ] Cache survives a Layer 5 audio re-sign — a stale signed URL doesn't invalidate the cached blob (because we strip the query string).
- [ ] Service worker upgrade (bumping `audio-cache-v1` → `v2`) cleanly evicts old cache without orphaning files.
- [ ] Playwright E2E on `beta.kitesforu.com`: download an episode → offline mode → play → audio plays from cache. (Per CLAUDE.md rule 11.)

## 8. Risks & mitigations

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Service worker breaks Next.js routing or hot-reload | Low | Scope SW to `/`, exclude `/_next/*` and Next.js HMR endpoints in fetch handler. |
| iOS Safari quota too small for multi-episode caching | High | Surface "Add to Home Screen" CTA; document the limit; ship the feature usable but constrained on iOS Safari without PWA install. |
| Cache key collision between two courses re-using the same audio URL | Very low (signed URLs are per-job-segment) | Cache key strips query but keeps full path. |
| User downloads 50 episodes → fills phone storage | Medium | Soft-cap at 2 GB + LRU eviction with explicit user confirm. |
| SW caches a stale 403'd URL | Low | Cache key strips query; first request after cache miss goes through Layer 5 re-sign. We never cache a 403/404 response — only `response.ok && response.status === 200`. |
| Range requests during playback bypass the cache | Medium | SW only handles full GETs without `Range` headers. Player uses `<audio src=blob:>` from cache, which doesn't issue range requests. |

## 9. Out of scope (recap)

- Background prefetch.
- DRM-protected offline content.
- Multi-device sync of downloads.
- Auto-cache of in-progress streaming segments.
- Native app shell.

## 10. Codex audit asks

1. **2 GB cap reasonable?** Or should we tier (free 500 MB / paid 5 GB) to align with credits/pricing? This proposal defaults to a single soft-cap regardless of tier; flagged for explicit decision.
2. **Should cache survive sign-out?** Privacy implication — a logged-out user could conceivably play back another account's cached audio if the device is shared. Default proposal: clear cache on sign-out. Confirm.
3. **Should we ship on iOS Safari without PWA install?** Without home-screen install, quota is so small (~50 MB) that the feature is essentially demo-quality. Default proposal: ship anyway with the home-screen CTA; risk is users get a confused "why so small?" feel.
4. **Per-platform analytics events?** `download_started` / `download_completed` / `download_played_offline` would tell us whether the feature drives engagement. Default proposal: ship analytics in Phase 2 (after we know the feature is stable).
5. **Should the library "offline filter" be a discoverable filter pill, or only auto-applied on `navigator.onLine === false`?** Default proposal: auto-applied only; revisit if users ask.

## 11. Rollback

Each phase is feature-flagged behind `feature_offline_downloads` in `lib/feature-flags.ts` (default OFF). Flag flip restores today's anchor-tag download. Service worker registration is conditional on the flag — if turned off, the SW unregisters itself on next page load via a `self.registration.unregister()` self-heal in the `activate` handler.

## 12. Sources

- `kitesforu-frontend/hooks/useCourseDetail.ts:619-628` — current download
- `kitesforu-frontend/components/AudioPlayer/useAudioPlayer.ts` — resolver to extend
- `kitesforu-frontend/lib/audio-url-resign.ts` — Layer 5 retry (cache-first sits before this)
- `kitesforu-docs/business/features/done/R1-phase2-ux-player-consumption.md` — phase close-out that named this thread
- `kitesforu-docs/business/features/done/audio_signed_urls/README.md` — Layer 5 pattern this proposal layers on top of
- MDN Service Worker + Cache API + Storage Estimation API references
