# P2: Increase Inworld Temperature for Emotional Content

## Problem
Base temperature 0.9-1.0 makes audio flat for comedy/drama/horror.
Inworld allows up to 2.0. For pre-rendered podcasts, 1.1-1.2 is safe and more expressive.

## Fix
Add genre_temperature_overrides to naturalness_rules.yaml.
Comedy: 1.15, Horror: 1.10, Romance: 1.10, Educational: 1.0, Bedtime: 0.85.

## Code Paths
- config/voice_intelligence/naturalness_rules.yaml
- src/workers/stages/audio/providers/inworld/shared.py — build_voice_params()

## Priority: P2 | Impact: Medium | Effort: 30 min
