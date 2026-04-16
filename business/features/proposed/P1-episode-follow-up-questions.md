# P1 — Episode Follow-Up Questions

**Status**: PROPOSED
**Priority**: P1
**Affected repos**: kitesforu-frontend, kitesforu-api
**Key files**: components/drive/QuickQuestionOverlay.tsx, app/car-mode/ (existing implementation)

## Problem

Car Mode already has a "Quick Question" feature — users can ask questions about the content mid-listen and get voice responses. But this feature is only available in Car Mode, not on the regular course detail page or the standalone audio player.

## User story

As a user listening to an interview prep episode at my desk, I want to pause and ask "can you explain that STAR example again?" without leaving the player — just like I can in Car Mode.

## Scope

- Extract the Quick Question overlay from Car Mode into a reusable component
- Wire it into the course detail audio player
- Support both voice input (tap mic, ask) and text input (type a question)
- The response should be:
  - **Text**: shown inline below the player
  - **Voice** (optional V2): spoken via TTS, with episode audio ducked
- Context: the question should include the current episode's topic + the approximate position in the transcript

## Implementation path

1. Extract `QuickQuestionOverlay` from `components/drive/` to a shared location
2. Add a "Ask a question" button to the audio player footer
3. When clicked, show the overlay with voice + text input
4. POST to a new API endpoint or reuse the Car Mode Q&A endpoint
5. Display the answer

## Acceptance criteria

- [ ] "Ask a question" button visible on the audio player during episode playback
- [ ] Voice input works (reuse VoiceInput component)
- [ ] Text input works as fallback
- [ ] Response displayed inline below the player
- [ ] Episode audio pauses while the user is asking (no speech over speech)
- [ ] Works on mobile (touch target, responsive overlay)

## Dependencies

- Car Mode's Q&A API endpoint (check if it's episode-aware or needs context injection)
- QuickQuestionOverlay component (already exists, needs extraction)

## Effort estimate

8-12 hours, 2-3 PRs (component extraction + player integration + API adapter).
