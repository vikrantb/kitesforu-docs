# P2: Add Inworld Voices to VoiceSelector

## Problem
config/tts_voices.yaml has no Inworld voices. Host2 gets OpenAI/Google voice IDs
that Inworld doesn't recognize.

## Fix
Add Inworld voices to tts_voices.yaml with gender labels. Update get_tier_provider()
so free tier routes through Inworld for both hosts.

## Code Paths
- config/tts_voices.yaml — add inworld voice catalog
- src/workers/stages/audio/voice_selector.py — get_tier_provider()

## Priority: P2 | Impact: Medium | Effort: 1-2 hours
