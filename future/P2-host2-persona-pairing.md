# P2: Select Contrasting Persona Pair for Dialogue

## Problem
Only ONE persona selected for dialogue. Host1 gets persona voice, Host2 gets VoiceSelector default.
No contrast optimization beyond gender alternation.

## Fix
Add select_persona_pair() to selector.py that picks Host2 for maximum contrast with Host1.
Use persona.pairing.pairs_well_with (already in YAML).

## Code Paths
- src/workers/stages/persona/selector.py — add select_persona_pair()
- src/workers/stages/combined/streaming_script_audio_worker.py — _select_persona()
- personas/*.yaml — pairing.pairs_well_with already exists

## Priority: P2 | Impact: High | Effort: 2-3 hours
