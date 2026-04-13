# Voice-First Mastery Loop — Gold Standard UX

**Status**: Architecture proposal
**Owner**: Frontend + Voice
**Depends on**: kitesforu-workers #275 (voice foundation), kitesforu-api #228 (Teach + Practice)
**Date**: 2026-04-12

---

## The differentiator

Most interview-prep products drop the candidate into a Q&A flow and hand them a scorecard at the end. That's commodity. The Gold Standard is **continuity + passive feedback**: the user feels they're in one coherent room the whole way through, the system's latency is visible as delight, and every correction is a whisper — never a lecture.

Five atomic components deliver this:

| # | Component | Purpose | Where it lives |
|---|-----------|---------|----------------|
| 1 | **One Room, Three States** | Single surface morphs Teach → Practice → Interview via atmospheric cue | Root of `/session/[id]` |
| 2 | **Listening Halo** | Visualizes <900ms latency as three states around avatar | Inside `<RoomSurface>` |
| 3 | **STAR Shadow** | Passive scaffold fills as speech hits S/T/A/R | Fixed left rail |
| 4 | **Carry-Forward Chip** | One persistent coaching target surviving the whole session | `fixed top-right` |
| 5 | **One-Fix Replay Card** | Silent sentence-first feedback between turns | Content slot |

All five share one Zustand store + one color token system, so a phase transition recolors every component in a single render pass. No drift, no duplicated state.

---

## The shared substrate

### Zustand store layout

One store, five slices, composed in `hooks/voice/useVoiceStore.ts`:

```ts
export const useVoiceStore = create<VoiceStore>()((...a) => ({
  ...roomAtmosphereSlice(...a),   // phase, transitioning, tokens
  ...voiceSessionSlice(...a),     // haloState, latencyMs, turnPhase
  ...starShadowSlice(...a),       // {s,t,a,r} + lastUpdatedMs
  ...coachingTargetSlice(...a),   // CarryForwardChip
  ...replayCardSlice(...a),       // OneFix | null
}))
```

Components subscribe with shallow selectors — unrelated state changes do not re-render.

### Color tokens (extend `tailwind.config.ts`)

```ts
theme.extend.colors.room = {
  teach:     { bg: '#1a1410', accent: '#f59e0b', halo: '#fbbf24', grain: 0.08 },
  practice:  { bg: '#0a1512', accent: '#10b981', halo: '#34d399', grain: 0.04 },
  interview: { bg: '#0b0f14', accent: '#64748b', halo: '#94a3b8', grain: 0.02 },
}
```

Every component reads `room.accent` / `room.halo` dynamically — a single phase transition recolors the halo, the chip, the STAR markers, and the replay card without touching any of their code.

---

## 1. One Room, Three States

### State machine

```
idle ──▶ teach ──(user: "ready")──▶ practice ──(coach: "real thing?")──▶ interview ──▶ review
          │                            │
          │ interruptible               │ NON-interruptible (1.8s cue must complete)
          │ (user can skip)             │ pointer-events-none while transitioning
```

Transitions fire from LiveKit `RoomEvent.DataReceived` payloads (`{type:'phase_advance'}`) emitted by the voice worker. UI subscribes to the `phase` slice and animates on change.

### Layout

```
<RoomSurface>                  ← root of /session/[id]
  <BackdropLayer />             ← gradient + SVG grain, morphs on phase
  <ToneBed />                   ← Howler ambient loop, cross-faded
  <ListeningHalo />             ← avatar + ring
  <StarShadow />                ← fixed left rail
  <CarryForwardChip />          ← fixed top-right
  <ContentSlot>                 ← question + optional <OneFixReplayCard>
    <TimerRail />               ← opacity-0 in teach, visible in practice/interview
  </ContentSlot>
</RoomSurface>
```

### Framer Motion

```ts
const roomTransition = { duration: 1.8, ease: [0.22, 1, 0.36, 1] }  // expo-out
// Morph: backgroundColor, filter (hue-rotate), opacity, scale (0.98 → 1)
// Do NOT morph: layout (causes LiveKit video track reflow jank)
// ToneBed: Howler.fade(current, 0, 1200) then play next layer
```

**Anti-patterns**: Don't route to `/teach`, `/practice`, `/interview` — unmounting `<LocalParticipant>` costs 400ms+ in LiveKit reconnect. Don't animate `background-image` (GPU expensive); layer opacity instead.

---

## 2. Listening Halo

### Approach

SVG, not Canvas. Three concentric `<circle>` with gaussian blur. Canvas wins only above ~60 particles; we have three rings.

### States

| State | r | stroke | blur | color | animation |
|-------|---|--------|------|-------|-----------|
| Listening | 56 | 4 | 8px | `room.halo` | scale 1→1.04, VAD rms-driven @ 60fps |
| Thinking | 52 | 2 | 16px | `room.halo/60` | rotate 360° over 2.4s linear ∞ |
| Asking | 60 | 6 | 4px | `room.accent` | pulse 1→1.08 synced to TTS level |
| Degraded (>900ms) | 52 | 2 | 16px | desat + red dot @ 12 o'clock | rotate slowed to 4s |

### LiveKit hooks (`hooks/voice/useListeningHalo.ts`)

- **Listening**: `RoomEvent.ActiveSpeakersChanged` when local speaks. RMS from `LocalAudioTrack.getFrequencyData()`, throttled to 30fps via `requestAnimationFrame` (never `setInterval`).
- **Thinking**: worker sends data msg `{type:'turn_end'}` after Silero VAD fires. Start 900ms timer.
- **Asking**: `RoomEvent.TrackSubscribed` on remote agent audio + `track.on('playing')`. Pulse driven by remote `RemoteAudioTrack` analyser.

### Degraded

Never hide the halo — invisibility reads as "broken." Desaturate to `hsl(var(--room-halo) / 0.4)`, add subtle red dot, slow rotation. Log to `latencyTelemetry` for tuning.

**Anti-patterns**: no `setInterval`, no re-creating `AnalyserNode` per state (leaks), don't animate `filter: blur()` via Framer (expensive) — toggle between pre-baked SVG filter variants.

---

## 3. STAR Shadow

### Two-tier classification

Running an LLM per STT partial (150ms cadence) blows the 500ms budget. Use tiered:

| Tier | Runs when | Latency | Purpose |
|------|-----------|---------|---------|
| 1 — client heuristic | Every STT partial | ~5ms | Instant fill on first confident match |
| 2 — server LLM (gpt-4o-mini) | 800ms after silence, debounced | ~400-800ms | Upgrade only — never downgrade Tier 1 |

**Tier 1 keyword rules:**

```
S (Situation): "at", "when I was", "our team", company/role nouns, past-tense context
T (Task):      "needed to", "was asked", "had to", "goal was"
A (Action):    "I built", "I led", "I wrote", "I decided" (first-person active)
R (Result):    "resulted in", "reduced", "increased", "shipped", "%", "$", "x faster"
```

Total word-to-fill latency: `STT partial 150 + heuristic 5 + Zustand set 1 + Framer start 16 ≈ 170ms`. Well under the 500ms target.

### Visual — 4 stacked pill markers, left rail

Not a bar (implies completion pressure). Not radial (decorative). Stacked pills read as scaffold, not scorecard.

```tsx
<div className="fixed left-6 top-1/2 -translate-y-1/2 flex flex-col gap-2">
  {(['s','t','a','r'] as const).map(k => (
    <motion.div
      key={k}
      className="w-1.5 h-10 rounded-full"
      animate={{
        backgroundColor: star[k] === 'filled'
          ? 'rgb(148 163 184)'
          : 'rgb(229 231 235 / 0.4)',
        scaleY: star[k] === 'filled' ? 1 : 0.85,
      }}
      transition={{ duration: 0.4, ease: [0.16, 1, 0.3, 1] }}
    />
  ))}
</div>
```

**State**: `{ s, t, a, r: 'idle' | 'filled'; lastUpdatedMs }` in `starShadowSlice`. Reset on `turnStart`.

**Anti-patterns**: no red (slate-400 only), no labels/tooltips during speech, no "X of 4" counter, no pulsing on idle (= nag), no shrinking on missing elements. Idle = soft gray ghost. Filled = slate. That's it.

---

## 4. Carry-Forward Chip

### Data shape

```ts
type CarryForwardChip = {
  chipId: string
  rule: string                   // "Quantify impact in every answer"
  shortLabel: string             // "Quantify impact" (max 22 chars)
  createdAfterTurn: number       // 1
  evaluationMethod: 'llm_rubric' | 'keyword' | 'regex'
  evaluationPrompt?: string      // for llm_rubric
  keywords?: string[]            // for keyword
  turnResults: { turn: number; met: boolean; evidence: string }[]
  status: 'active' | 'mastered' | 'retired'
  streak: number                 // consecutive turns met
}
```

Persisted at `sessions/{id}/carryForward` in Firestore.

### Creation

After turn 1, coach LLM returns `{carry_forward: {...}}` alongside feedback. Dispatch `setCarryForward(chip)` — chip fades in `opacity 0→1, y -8→0, duration 0.4, delay 0.8` (after coach card lands).

### Visual

```tsx
<motion.div className="
  fixed top-4 right-4 z-40
  px-3 py-1.5 rounded-full
  bg-room-accent/10 border border-room-accent/30
  backdrop-blur-md
  text-xs font-medium text-room-halo
  flex items-center gap-2
">
  <span className={`h-1.5 w-1.5 rounded-full ${
    lastResult === 'met' ? 'bg-emerald-400' :
    lastResult === 'missed' ? 'bg-amber-400' : 'bg-slate-400'
  }`} />
  {chip.shortLabel}
  {chip.streak >= 2 && <span className="text-[10px] tabular-nums">×{chip.streak}</span>}
</motion.div>
```

### Per-turn evaluation (no voice interrupt)

On `RoomEvent.DataReceived {type:'turn_complete', transcript}`, background worker calls `/api/coaching/evaluate-chip`. On result: update `turnResults`, animate dot color (0.6s). On `met`: brief `scale 1→1.06→1` over 400ms. **Never** play audio, never block TTS, never modal.

### Resolution

`streak >= 3` → `mastered`. Chip morphs gold (`bg-amber-400/20`), checkmark replaces dot, auto-dismisses after 2s. Coach issues a new chip next turn. Max one active chip at a time.

**Anti-patterns**: don't re-run eval on transcript deltas — only on `turn_complete`. Don't show rubric scores on chip (cognitive load mid-interview). Don't animate `width` on label change — use `AnimatePresence mode="popLayout"` with `layout` prop. Don't persist failed attempts visibly past 2 turns (demoralizing).

---

## 5. One-Fix Replay Card

### Layout — one sentence, one rewrite

No "feedback" header. No grid. No card chrome that says "report." Glassmorphic whisper.

```tsx
<motion.div
  className="mx-auto max-w-xl px-6 py-5 rounded-2xl
             bg-white/5 backdrop-blur-md border border-white/10"
  initial={{ opacity: 0, y: 8, filter: 'blur(4px)' }}
  animate={{ opacity: 1, y: 0, filter: 'blur(0px)' }}
  exit={{ opacity: 0, y: -4, filter: 'blur(2px)' }}
  transition={{ duration: 0.5, ease: [0.16, 1, 0.3, 1] }}
>
  <p className="text-[15px] leading-relaxed text-neutral-300 font-normal">
    {fix.sentence}
  </p>
  <p className="mt-3 text-[17px] leading-snug text-white font-medium tracking-tight">
    &ldquo;{fix.rewrite}&rdquo;
  </p>
</motion.div>
```

**Hierarchy**: fix sentence = secondary (neutral-300, 15px). Rewrite = primary (white, 17px, medium, quoted). The eye lands on the rewrite — quotation marks signal "say this next time." Total visible content ≈2 lines, reads in ~2.5s.

### Data mapping

```ts
type OneFix = { sentence: string; rewrite: string; dimension: RubricDim }
// Derived from existing TurnFeedback/CoachFeedback:
//   1. Pick LOWEST-scoring rubric dimension
//   2. sentence = coach.fix_suggestions[dim][0]
//   3. rewrite  = coach.rewrites[dim][0]
//   4. DROP: all scores, grid, star booleans, other dimensions, explanations
```

### Timing wiring

- `onSttFinal` → request `/api/coach/one-fix` (reuses existing 5-dim endpoint, server selects min-score dim) → `setReplayCard(fix)`
- Card mounts when `replay.card && !tts.speaking`
- `onTtsStart(nextQuestion)` → `setReplayCard(null)` → AnimatePresence exits
- **Hard ceiling**: auto-dismiss after 4000ms even if TTS is delayed

### Differentiation from existing cards

| Card | Where | Look | Use case |
|------|-------|------|----------|
| **TurnFeedbackCard** (text mode) | Below question | Opaque white, 5-dim grid, ~400px | Assessment mode, text only |
| **CoachFeedbackCard** (text practice) | Below question | Warm amber, icon, multi-line | Practice mode, text only |
| **OneFixReplayCard** (voice) | Inside content slot | Glassmorphic whisper, ~90px | Voice mode, between turns |

Blur-in entry is the signature — reads as "thought surfacing," not "card appearing."

---

## Build order

Build vertically, not horizontally. One component fully wired end-to-end beats five half-finished.

1. **Token substrate + RoomSurface shell** (no animations yet — just the 3 color themes + a hard-cut phase toggle)
2. **Listening Halo** against the existing LiveKit voice worker — the most differentiating piece, validates latency telemetry
3. **STAR Shadow** — Tier 1 heuristic only, ship Tier 2 LLM upgrade in a follow-up
4. **One-Fix Replay Card** — wires into existing `/coach/*` endpoint, small surface area
5. **Carry-Forward Chip** — needs a new `/coaching/evaluate-chip` route + Firestore subcollection
6. **Atmospheric transitions** (Framer morphs + tone bed) — polish layer, ship last after each component works in isolation

---

## Open questions

1. **Tone bed source** — license ambient loops (Epidemic Sound ≈ $15/mo) or commission 3 custom beds?
2. **Halo on mobile** — 96px avatar + 60px halo eats 30% of a 375px viewport. Shrink to 72/44 on `sm:`?
3. **STAR Tier 2 LLM budget** — one gpt-4o-mini call per turn end ≈ $0.0003/turn × 8 turns = $0.0024/session. Acceptable. But does Tier 2 ever overrule Tier 1, or only upgrade?
4. **Carry-Forward persistence across sessions** — does a user's chip history follow them between sessions (motivating streak), or reset each session (fresh start)?
5. **Accessibility** — Halo is visual-only. Do we need a parallel audio chime for `Thinking → Asking` transition for low-vision users?

---

## File layout

```
kitesforu-frontend/
├── app/session/[id]/page.tsx               ← RoomSurface root
├── components/voice/
│   ├── RoomSurface.tsx
│   ├── BackdropLayer.tsx
│   ├── ToneBed.tsx
│   ├── ListeningHalo.tsx
│   ├── StarShadow.tsx
│   ├── CarryForwardChip.tsx
│   └── OneFixReplayCard.tsx
├── hooks/voice/
│   ├── useVoiceStore.ts                    ← composed store
│   ├── useListeningHalo.ts                 ← LiveKit event → halo state
│   └── useStarClassifier.ts                ← Tier 1 + Tier 2 coordination
└── tailwind.config.ts                       ← extend room tokens

kitesforu-api/
└── src/api/routes/voice/
    ├── one_fix.py                          ← POST /api/coach/one-fix
    └── evaluate_chip.py                    ← POST /api/coaching/evaluate-chip

kitesforu-workers/
└── interview_worker/voice/
    └── phase_controller.py                 ← emits {type:'phase_advance'} via LiveKit data
```

---

## Anti-patterns (global)

- **No routing between phases.** One `<RoomSurface>`, one LiveKit connection.
- **No setInterval in animation loops** — `requestAnimationFrame` only.
- **No scores in chips or the halo** — keep mid-interview cognitive load low.
- **No red error states** — this is coaching, not grading.
- **No speech over speech** — replay card is silent, always.
- **No half-shipped components** — each lands end-to-end or not at all.
