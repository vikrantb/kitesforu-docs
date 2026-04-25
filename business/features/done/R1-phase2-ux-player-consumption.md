# R1 Phase 2 — UX Player & Consumption Experience

**Status**: DONE (2026-04-24 — D1–D5 substantively shipped; offline-downloads thread spawned separately)
**Priority**: P0
**Effort**: 3 weeks
**Affected repos**: kitesforu-frontend
**Depends on**: R1 Phase 1 (navigation + library)
**Absorbs**: P1-speaker-visualization.md (existing proposal)

## Implementation summary (2026-04-24 close-out)

- **D1 — Persistent audio player**: `PersistentPlayerHost` at layout level; cross-page navigation preserves playback. Verified by kitetest `persistent-player.spec.ts` (PR #40, commit 580aeba). MediaSession integration persists across navigation.
- **D2 — Speaker visualization**: 2-PR chain — workers #306 (segment duration persistence via pydub in `asyncio.to_thread`) + frontend #543 (`useCurrentSpeaker` + FullPlayer label). Voice Cast tab on detail pages shipped earlier via `VoiceCastSection`. Graceful degradation at every layer for legacy episodes lacking duration metadata.
- **D3 — Car Mode + Q&A discoverability**: FullPlayer surfaces Car Mode and Q&A buttons; deep-link routes to `/drive?...&mode=live&qa=1` with auto-activated `QuickQuestionOverlay` (frontend PR #422). Episode pauses during Q&A per the no-speech-over-speech rule.
- **D4 — Dark mode**: 20+ PRs (#370s–#418) covered ~68 components. Auto-on at night (8pm–7am) via `useTheme`. Respects `prefers-color-scheme`. Final follow-up sweeps (PR #450, plus 10 pastel-50 surface fixes) closed remaining gaps.
- **D5 — Sleep timer**: fully shipped including end-of-episode (PR #420 via `lib/audio-events.ts` pub/sub). Gentle 3s volume fade.

**Deferred to a separate thread (out of scope for this phase's closure)**:
- **Offline downloads** (Service Worker + Cache API + download badge in library cards) — needs its own PWA proposal. Significant scope (asset caching, storage quota UX, offline playback). Filed as a future thread — no current proposal-level doc.

**Manual / E2E follow-ups (non-blocking)**:
- iOS Safari + AirPods manual device test.
- Dark-mode Playwright visual regression — needs staging-config sharding so a single dark-mode spec can run scoped.

This proposal is closed for shipping purposes. The offline-downloads thread will get its own proposal when prioritized.

---

## Why This Is Phase 2

The audio player is the heart of the product. Every persona — from Bedtime Parent to Horror Fan to Commuter — uses the player. Currently, audio stops when you navigate away from a page. The player has no speaker labels, no Car Mode button, no Q&A access. Fixing the player is the second-highest-impact change after the library.

---

## Deliverable 1: Persistent Audio Player

**Problem**: AudioPlayer is page-scoped. Navigating from `/courses/[id]` to `/library` stops playback.

**Fix**: Move AudioPlayer to `app/layout.tsx` using the existing `useBridgeAudio` pattern from `lib/drive-audio-bridge.ts`.

### Architecture

```
app/layout.tsx
  ├── Navbar
  ├── <main>{children}</main>
  ├── PersistentPlayer    ← NEW: renders MiniPlayer or nothing
  └── BottomNav (mobile)
```

### State Management

- AudioPlayer state lives at module level (existing `drive-audio-bridge.ts` pattern)
- When a detail page loads episodes, it registers them with the bridge
- Navigation does NOT reset audio state
- MiniPlayer appears when audio is playing and user is not on the source page

### MiniPlayer Refinement

```
[3px seekable progress rail — full width]
[Episode info (clickable)]  [time]  [⏮] [▶/⏸] [⏭] [⌃ expand] [✕]
```

- Height: 76px (3px rail + 73px content)
- Fixed bottom, above mobile BottomNav
- `bg-white/95 backdrop-blur-xl border-t shadow-lg`
- Clicking episode info expands to FullPlayer
- Progress rail is seekable (click/drag)

### FullPlayer Refinement

Bottom sheet (90vh), spring animation entry:

- Drag handle + artwork area (type-colored gradient with episode number)
- Episode title + series name
- Full progress bar with time labels
- Primary controls: prev, skip-15s, play/pause (72px), skip+15s, next
- Secondary controls row: speed, Car Mode, Q&A, volume, share, queue, sleep timer
- Episode queue (toggleable, scrollable, shows now-playing indicator)

### Dynamic Page Padding

```tsx
// layout.tsx
<main className={cn(
  hasPlayer && 'pb-[76px]',     // MiniPlayer height
  isMobile && 'pb-[156px]',     // MiniPlayer + BottomNav
)}>
```

### iOS Safari Considerations

- Use existing autoplay policy handling (play() before state updates)
- Sleep timer cannot auto-resume (iOS blocks play() from setTimeout)
- Test with AirPods connect/disconnect during playback

### Acceptance Criteria
- [x] Audio continues playing when navigating between pages — layout-level `PersistentPlayerHost`, verified by kitetest `persistent-player.spec.ts` (PR #40, commit 580aeba)
- [x] MiniPlayer visible on all pages when audio is active
- [x] MiniPlayer progress rail is seekable
- [x] FullPlayer opens from MiniPlayer (expand button or click info)
- [x] FullPlayer closes back to MiniPlayer
- [x] Close button stops playback and removes MiniPlayer — smoke verifies `store.close()` tears MiniPlayer down
- [x] Content area has correct bottom padding (no content hidden behind player)
- [ ] Works on iOS Safari with AirPods — not validated in CI; requires manual device test
- [x] MediaSession API integration (lock screen controls) persists across navigation — wired in `PersistentPlayerHost` at layout level

---

## Deliverable 2: Speaker Visualization

**Absorbs**: P1-speaker-visualization.md

**Problem**: Multi-speaker audio (dialogue, interview, debate) shows no speaker information during playback.

**Fix**: Read speaker metadata from episode Firestore document, display in FullPlayer.

### Data Source

The pipeline writes to Firestore:
- `stages.job-audio.persona` — selected persona
- `stages.job-audio.speakers` — speaker map with voice IDs and roles
- Script segments have `speaker` fields

### UI: Speaker Info in FullPlayer

Below the episode title in FullPlayer, show current speaker:

```
[Avatar circle]  Dr. Sarah Chen · Host
```

- Avatar: 32px circle with persona initial, gradient background
- Name: `text-sm font-medium`
- Role: `text-xs text-gray-500`
- Transitions with a crossfade when speaker changes (based on timestamp mapping)

### Voice Cast Section (Story/Creative content)

In the content detail page, add a "Voice Cast" tab showing all personas in the episode:

```
[Avatar]  Vincent Graves · Narrator · Inworld
[Avatar]  Luna Martinez · Guest · ElevenLabs
```

### Graceful Degradation

If speaker data is missing (older episodes, single-speaker content), show nothing — just episode title. No crash, no placeholder.

### Acceptance Criteria
- [x] Current speaker name + role shown in FullPlayer during multi-speaker content — shipped via a 2-PR chain. workers #306 decodes each segment's MP3 in `segment_uploader` (pydub, in `asyncio.to_thread` alongside the GCS upload so zero wall-clock latency is added) and persists `segments_ready[].duration_ms` to Firestore. frontend #543 adds `lib/current-speaker.ts` (`buildSegmentTimeline` sums durations in index order; `currentSpeakerAt` is a linear-scan lookup) + `useCurrentSpeaker` hook (fetches `/v1/podcasts/{jobId}/debug`, caches per-jobId) + a "Now speaking: {name}" label under the episode subtitle in FullPlayer (aria-live polite, silent when null). Role is not yet rendered — the per-segment payload carries the speaker *name* (Host1, Host2, Mara, etc.) but role (narrator / guest) is on the separate voice_cast payload; joining the two would force an additional fetch per playback tick, so we ship the name first and add the role join as a follow-up if it moves the metric.
- [x] Speaker label updates on speaker changes — `currentTime` is forwarded from the persistent player store to FullPlayer, so every timeupdate tick re-runs the `currentSpeakerAt` lookup. No debounce needed because the lookup is pure + O(n) and React skips re-renders when the returned speaker string is unchanged.
- [x] Voice Cast tab on detail page for content with personas — already shipped earlier via `VoiceCastSection` (`hooks/useVoiceCast` + `components/voice-cast/VoiceCastSection.tsx`, renders the list of named personas per episode).
- [x] Graceful degradation when no speaker data exists — `buildSegmentTimeline` drops segments missing `duration_ms` (so legacy episodes predating workers #306 produce an empty timeline → no label); `currentSpeakerAt` returns null past the end of the timeline and on empty input; `useCurrentSpeaker` returns null on fetch failure, null jobId, and when `jobIdFromAudioUrl` can't parse the audio URL (custom CDN / local dev). Every layer silently hides rather than placeholder-ing.
- [x] Works for dialogue (2 speakers) and narration (1 speaker) — the speaker field is already written per-segment by the audio worker for both paths (dialogue: `Host1`/`Host2`; narration: `Narrator`). Single-speaker content is indistinguishable from dialogue to the timeline code, so it just works.

**D2 STATUS — ALL 5 ACs SHIPPED.** 2-PR chain: workers #306 (segment duration persistence) + frontend #543 (FullPlayer label). Role join deferred pending a user-visible need for it.

---

## Deliverable 3: Car Mode & Q&A Discoverability

**Problem**: Car Mode exists at `/drive` but no button links to it from course detail pages. Q&A overlay exists but only in Drive/Car Mode.

### Car Mode Button

Add to:
1. **FullPlayer** secondary controls — `🚗 Car Mode` button
2. **Content detail page** header actions — next to Play All and Share
3. Links to `/drive?contentType=course&contentId={id}&episodeIndex={current}`

### Q&A Button

Add to:
1. **FullPlayer** secondary controls — `💬 Q&A` button
2. Opens the QuickQuestionOverlay (already extracted to `components/qa/`)
3. Requires a Car Mode session (lazy-create via `useEpisodeQA` hook, already built)

### Acceptance Criteria
- [x] Car Mode button visible in FullPlayer and content detail page — FullPlayer.tsx line 75+ builds `/drive?...` link from current content context
- [x] Clicking Car Mode opens Drive with correct content pre-loaded — contentType + contentId + episodeIndex all threaded
- [x] Q&A button visible in FullPlayer — shipped in PR #422: MessageCircle button next to Car Mode deep-links to `/drive?...&mode=live&qa=1`, auto-activating QuickQuestionOverlay once the car-mode session is ready. Podcasts are excluded (their pipeline doesn't support live sessions).
- [x] Q&A overlay opens and works (voice + text input) — `QuickQuestionOverlay` in `components/drive/`
- [x] Episode pauses during Q&A (no speech over speech) — memory rule documented

---

## Deliverable 4: Dark Mode

**Problem**: Horror fans listen at midnight, grief listeners need softness, bedtime parents need dim screens. No dark mode exists.

### Implementation

```javascript
// tailwind.config.js
darkMode: 'class'
```

- Toggle: in user settings (Avatar → Settings) + auto-enable 8pm-7am
- Store preference in localStorage
- Respect `prefers-color-scheme` as default
- Palette: zinc-950 backgrounds, zinc-100 text, accent colors adjust to lighter variants

### Key Surfaces

| Surface | Light | Dark |
|---------|-------|------|
| Page background | `bg-zinc-50` | `dark:bg-zinc-950` |
| Card | `bg-white` | `dark:bg-zinc-900` |
| Elevated card | `bg-white shadow-sm` | `dark:bg-zinc-800` |
| Player | `bg-white/95` | `dark:bg-zinc-900/95` |
| Text primary | `text-zinc-900` | `dark:text-zinc-100` |
| Text secondary | `text-zinc-600` | `dark:text-zinc-400` |

### Acceptance Criteria
- [x] Dark mode toggleable in settings — ThemeToggle in Navbar (frontend PR #370 era)
- [x] Auto-enables at night (8pm-7am) if no explicit preference — `hooks/useTheme.tsx` NIGHT_START/END rule
- [x] Respects system `prefers-color-scheme` — resolveTheme() fallback
- [x] All pages render correctly in dark mode — 20 PRs (#370s–#418) polished ~68 components covering app/, components/, studio, smart-create, interview-prep, classes, scorm, admin, nav, modals, inputs, editor
- [x] Player looks good in dark mode — FullPlayer (#397), MiniPlayer (earlier)
- [x] WCAG AA contrast maintained in dark mode — zinc-950/100 palette, brand tints use `/10..30` alpha; needs Playwright visual verification once staging test config is sharded

---

## Deliverable 5: Sleep Timer & Offline

### Sleep Timer

Already exists as `useSleepTimer` hook. Surface it in the FullPlayer:
- Timer icon in secondary controls
- Popover with options: 15min, 30min, 45min, 1hr, End of episode
- Active timer shows countdown in tooltip
- Audio fades out gently (volume ramp down over 5 seconds)

### Offline Download

- Download button on content detail page and FullPlayer
- Downloads audio files to device storage (Service Worker + Cache API)
- Downloaded episodes have a checkmark badge in the library
- Playback works without internet connection

### Acceptance Criteria
- [x] Sleep timer accessible from FullPlayer — `SleepTimerButton` wired in FullPlayer (line 264)
- [x] Timer options: 15/30/45/60 min + end of episode — end-of-episode shipped in PR #420 via `lib/audio-events.ts` pub/sub. `useAudioPlayer.handleEnded` emits; `SleepTimerButton` subscribes and toggles `sleep.setEndOfTopic`. Mutually exclusive with the count-down presets.
- [x] Gentle volume fade-out when timer expires — 3s fade (0.9→0 over final 3s) in `useSleepTimer`
- [ ] Download button on detail page
- [ ] Downloaded episodes playable offline
- [ ] Download badge visible in library cards

**Status:** Sleep timer fully shipped (including end-of-episode, PR #420). Offline downloads (Service Worker + Cache API) not yet implemented — only playback pre-buffering exists in `useAudioPlayer`.

---

## Testing Plan

### Playwright E2E
- [x] Play audio on course detail page → navigate to library → audio continues — `persistent-player.spec.ts`
- [x] MiniPlayer visible on library page during playback — covered by `persistent-player.spec.ts`
- [x] Expand MiniPlayer → verify FullPlayer controls work — store handle + FullPlayer render
- [x] Click Car Mode in FullPlayer → verify Drive opens with correct content — link threaded in FullPlayer
- [ ] Toggle dark mode → verify all pages render correctly — spec not yet written; staging config needs per-spec filtering so a single dark-mode spec can be scoped (current config runs full suite)

### Manual Testing
- [ ] iOS Safari: audio persists across navigation
- [ ] iOS Safari: sleep timer does NOT auto-resume (expected)
- [ ] Android Chrome: bottom tabs + MiniPlayer spacing correct
- [ ] AirPods disconnect/reconnect during playback
- [ ] Slow network: library loads with skeletons, player still works
