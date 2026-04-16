# Voice Intelligence System

**Status**: DONE
**Shipped**: April 4-5, 2026
**Repos**: kitesforu-workers
**PRs**: #195-204 (50+ files, ~4,500 lines, 68 tests)

## What it does

Genre-aware content experience across the entire audio pipeline. The system detects the content genre (comedy, horror, educational, drama, etc.) and applies genre-specific rules to every stage: script generation, voice selection, expression, pacing, and post-processing.

## Key capabilities

- **18 genre profiles** with narration rules + genre craft rules
- **Expression adapters** for 5 TTS providers (Inworld, ElevenLabs, Google, OpenAI, Azure)
- **Fallback translation** — if a provider doesn't support an expression, it translates to the closest equivalent
- **Quality logging** — every expression decision is logged for debugging
- **ElevenLabs style baseline**: 0.10 (was 0.35 — reduced to prevent over-expression)
- **Fallback chain**: Inworld → ElevenLabs → Google → OpenAI
- **Streaming generator**: genre-aware template selection

## User impact

Users creating audio content now get genre-appropriate voice, pacing, and expression automatically. A horror podcast sounds different from a comedy set — no manual configuration needed.
