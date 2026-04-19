# 69E54B51 — Car Mode audio concurrency (two voices at once)

**Status**: FIXED
**Reported**: 2026-04-19 by product owner during live drive-test
**Priority**: P0 — core Q&A flow is unusable
**Surface**: `/car-mode?topic=...` on beta
**Fix PRs**: kitesforu-frontend #456 (pause-on-assistant-speak) — merged 2026-04-19
**Verification**: moved to `closed/` once beta revision 00574+ confirms no overlap between episode audio and Q&A TTS during a full ask→answer cycle.

## Reported symptom (verbatim)

> When I started talking, the player kept going even though I was talking and the lady was replying back, so there were 2 voices at same time causing chaos. The player should stop when the lady (or in Car Mode response) starts. Not when I start talking. It should be able to keep the podcast going but still listen to the mic.

## Correct mental model

- Mic is always listening in Commands mode while episode plays.
- Episode audio KEEPS PLAYING while the user speaks.
- When the assistant RESPONSE begins (TTS answer), episode pauses.
- When the response finishes, episode resumes.
- Invariant: never simultaneously play episode + assistant TTS.

## Root cause

The Q&A flow pauses the episode the moment the user OPENS the overlay (or the voice "ask question" command fires), which is the wrong side of the turn boundary. Real problem sites:

- `hooks/drive/useQuickQuestion.ts:392` — `activateVoice()` calls `pauseIfPlaying()` *before* the mic even opens. Comment says "eliminates competing audio", but user wants competing audio during listening — they only want single-audio during the ASSISTANT turn.
- `hooks/drive/useQuickQuestion.ts:238` — `askQuestion()` pauses again as a "no-op if already paused". Redundant on the voice path and reinforces the wrong invariant.
- `hooks/car-mode-v2/useCarVoiceOrchestrator.ts:202-239` — `speak()` pauses STT but never pauses episode audio. This is the missing hook where the correct pause should fire (utterance.onstart / just before `speechSynthesis.speak`).
- `utterance.onend` / `onerror` already call `resumeAfterSpeech` (line 216) — perfect place for episode resume.

## Proposed change set (≤8)

All inside the car-mode frontend; no API or worker change.

1. `useQuickQuestion.ts:392` — **remove** `pauseIfPlaying()` from `activateVoice()`.
2. `useQuickQuestion.ts:238` — **remove** `pauseIfPlaying()` from `askQuestion()` (or guard to `!isVoiceMode`).
3. `useQuickQuestion.ts:195` — **remove** `pauseIfPlaying()` from `activate()` for consistency (text mode too).
4. `useCarVoiceOrchestrator.ts:74-79` — add optional `onSpeakStart` + `onSpeakEnd` callbacks to options.
5. `useCarVoiceOrchestrator.ts:~218` — call `onSpeakStart?.()` immediately before `speechSynthesis.speak(utterance)`.
6. `useCarVoiceOrchestrator.ts:~220` — inside `resumeAfterSpeech` (runs on `onend` and `onerror`), call `onSpeakEnd?.()`.
7. `app/car-mode/page.tsx:330-334` — wire `onSpeakStart: () => { if (drive.isPlaying) { drive.toggle(); pausedByOrchestratorRef.current = true } }` and `onSpeakEnd: () => { if (pausedByOrchestratorRef.current) { drive.toggle(); pausedByOrchestratorRef.current = false } }`.
8. `useQuickQuestion.ts:168-173` — tighten `resumeIfWePaused()` to no-op when `pausedByQQRef.current === false` (after changes 1-3 this is always the case on voice path, so dismiss/auto-dismiss leave playback alone — orchestrator is the sole pause/resume authority).

## Risks + mitigations

- **Echo / TTS feedback into STT** — orchestrator already stops STT in `speak()` before speaking (`speakingRef.current = true`, abort recognition). `startListening` bails on `speakingRef.current`. Invariant preserved.
- **iOS autoplay block on resume** — `drive.toggle()` from `utterance.onend` is NOT inside a user gesture. Will surface the existing "Tap to resume" overlay from `useAudioPlayer.ts:273-282`. Accept as documented fallback.
- **User dismisses overlay mid-response** — add `voice.stopSpeaking()` call in the Car Mode dismiss path so an in-flight TTS utterance stops cleanly; then `onSpeakEnd` runs and resumes the episode.
- **Double-toggle race** — if user manually paused episode before TTS starts, `onSpeakStart` would flip it wrong. Guard: only call `drive.toggle()` when `drive.isPlaying === true`; mirror in `pausedByOrchestratorRef`.

## Test plan

- **Playwright A (primary repro):** start `/car-mode?topic=test`, wait for episode playing, say "ask a question" then immediately "what is X". Assert `audio.paused === false` for ≥1s after STT dispatch. When `speechSynthesis.speaking === true`, assert `audio.paused === true` within 200ms. When `speechSynthesis.speaking` returns false, assert `audio.paused === false` within 500ms.
- **Playwright B (no-overlap invariant):** poll every 100ms for 30s. Assert `!(speechSynthesis.speaking && !audioEl.paused)` on every sample. Any overlap frame fails the test.
- **Playwright C (dismiss mid-TTS):** trigger Q&A, wait for TTS start, click dismiss. Assert episode resumes within 500ms AND `speechSynthesis.speaking === false` within 200ms.

## Files referenced

- `kitesforu-frontend/hooks/drive/useQuickQuestion.ts`
- `kitesforu-frontend/hooks/car-mode-v2/useCarVoiceOrchestrator.ts`
- `kitesforu-frontend/app/car-mode/page.tsx`
- `kitesforu-frontend/hooks/useDriveMode.ts`
- `kitesforu-frontend/components/AudioPlayer/useAudioPlayer.ts`
