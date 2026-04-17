# P2 — Voice Architecture Consolidation

**Status**: PROPOSED
**Priority**: P2 (blocked on LiveKit integration decision)
**Affected repos**: kitesforu-frontend
**Maps to**: UI Excellence Sweep S6, S7, S8, S9 (Wave 3)

## Problem

Two divergent voice stacks exist in the frontend:
1. **Car Mode** uses `useCarVoiceOrchestrator` — real SpeechRecognition, SpeechSynthesis, barge-in, turn-taking
2. **Voice-First Simulation** uses `MockVoiceController` — setTimeout-driven theater with no real voice I/O

The mock preview can't be upgraded to real voice — it must be rewritten against the car mode orchestrator (or a shared abstraction).

## Scope

- Extract a shared `useVoiceOrchestrator` from Car Mode
- Rewrite Voice-First Simulation to use the shared orchestrator
- Add barge-in handling (`asking_interrupted` state)
- Wire halo state to real audio events (not manual `setHaloState` calls)
- Fix the speech-over-speech guard on `OneFixReplayCard` (check audio bridge state, not just haloState)

## Acceptance criteria

- [ ] One voice orchestrator serves both Car Mode and Voice Simulation
- [ ] ListeningHalo reflects real audio state
- [ ] OneFixReplayCard gates on actual TTS speaking state
- [ ] Barge-in works (user can interrupt the interviewer)
- [ ] Car Mode regression test passes

## Dependencies

- Decision on LiveKit vs browser-native SpeechRecognition for the shared orchestrator
- Car Mode must stay functional throughout the migration

## Effort estimate

16-24 hours, 3-4 PRs (the biggest structural lift in the sweep).
