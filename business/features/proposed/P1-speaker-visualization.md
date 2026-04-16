# P1 — Speaker Visualization in Audio Player

**Status**: PROPOSED
**Priority**: P1
**Affected repos**: kitesforu-frontend
**Key files**: components/AudioPlayer/, stages/audio/voice_intelligence/ (backend reference)

## Problem

The audio player shows episode title + a basic progress bar. But the backend now produces multi-speaker audio with distinct voices (persona system), expression tags, and speaker roles (host, guest, narrator). None of this is visible to the user during playback.

## User story

As a user listening to a two-speaker podcast, I want to see who is speaking, their name, and their role — like a premium podcast player that shows speaker labels.

## Scope

- **Speaker labels**: show "Dr. Sarah Chen (Host)" and "Marcus Reid (Guest)" or similar during playback
- **Speaker transitions**: when the speaker changes in the audio, update the visual indicator
- **Expression indicators**: subtle visual hint when the speaker's tone changes (e.g., "thoughtful", "excited") — optional, could be a V2
- **Avatar or icon**: each persona has a visual identity; show it next to the speaker label

## How speaker data gets to the frontend

The pipeline writes speaker metadata to the job/episode Firestore document:
- `stages.job-audio.persona` — the selected persona
- `stages.job-audio.speakers` — the speaker map with voice IDs and roles
- Script segments have `speaker` fields indicating who speaks each segment

The frontend needs to:
1. Read the speaker map from the episode document
2. Map segment timestamps to speaker labels
3. Update the display during playback based on current playback position

## Acceptance criteria

- [ ] Speaker name + role shown during playback
- [ ] Speaker label updates on speaker changes
- [ ] Persona avatar/icon displayed if available
- [ ] Works for both single-speaker (narration) and multi-speaker (dialogue) content
- [ ] Degrades gracefully if speaker data is missing (just show episode title, no crash)

## Effort estimate

6-8 hours, 2 PRs (data model + UI).
