# R1 Phase 2 — UX Player & Consumption Experience

**Status**: PROPOSED
**Priority**: P0
**Effort**: 3 weeks
**Affected repos**: kitesforu-frontend
**Depends on**: R1 Phase 1 (navigation + library)
**Absorbs**: P1-speaker-visualization.md (existing proposal)

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
- [ ] Audio continues playing when navigating between pages
- [ ] MiniPlayer visible on all pages when audio is active
- [ ] MiniPlayer progress rail is seekable
- [ ] FullPlayer opens from MiniPlayer (expand button or click info)
- [ ] FullPlayer closes back to MiniPlayer (drag down or collapse button)
- [ ] Close button stops playback and removes MiniPlayer
- [ ] Content area has correct bottom padding (no content hidden behind player)
- [ ] Works on iOS Safari with AirPods
- [ ] MediaSession API integration (lock screen controls) persists across navigation

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
- [ ] Current speaker name + role shown in FullPlayer during multi-speaker content
- [ ] Speaker label updates on speaker changes
- [ ] Voice Cast tab on detail page for content with personas
- [ ] Graceful degradation when no speaker data exists
- [ ] Works for dialogue (2 speakers) and narration (1 speaker)

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
- [ ] Car Mode button visible in FullPlayer and content detail page
- [ ] Clicking Car Mode opens Drive with correct content pre-loaded
- [ ] Q&A button visible in FullPlayer
- [ ] Q&A overlay opens and works (voice + text input)
- [ ] Episode pauses during Q&A (no speech over speech)

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
- [ ] Dark mode toggleable in settings
- [ ] Auto-enables at night (8pm-7am) if no explicit preference
- [ ] Respects system `prefers-color-scheme`
- [ ] All pages render correctly in dark mode
- [ ] Player looks good in dark mode
- [ ] WCAG AA contrast maintained in dark mode

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
- [ ] Sleep timer accessible from FullPlayer
- [ ] Timer options: 15/30/45/60 min + end of episode
- [ ] Gentle volume fade-out when timer expires
- [ ] Download button on detail page
- [ ] Downloaded episodes playable offline
- [ ] Download badge visible in library cards

---

## Testing Plan

### Playwright E2E
- [ ] Play audio on course detail page → navigate to library → audio continues
- [ ] MiniPlayer visible on library page during playback
- [ ] Expand MiniPlayer → verify FullPlayer controls work
- [ ] Click Car Mode in FullPlayer → verify Drive opens with correct content
- [ ] Toggle dark mode → verify all pages render correctly

### Manual Testing
- [ ] iOS Safari: audio persists across navigation
- [ ] iOS Safari: sleep timer does NOT auto-resume (expected)
- [ ] Android Chrome: bottom tabs + MiniPlayer spacing correct
- [ ] AirPods disconnect/reconnect during playback
- [ ] Slow network: library loads with skeletons, player still works
