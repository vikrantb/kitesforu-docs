# Voice Persona System

**Status**: DONE
**Shipped**: April 5, 2026
**Repos**: kitesforu-workers
**Key files**: personas/ directory, stages/persona/selector.py, stages/persona/resolver.py

## What it does

20 named character personas (e.g., "Vincent Graves", "Dr. Sarah Chen") that can be selected as hosts for audio content. Each persona has a voice, personality, delivery style, and provider-specific voice mappings.

## Key capabilities

- **YAML-driven**: personas live in `personas/` directory — zero code changes to add a new one
- **6-stage selector**: hard filters → content scoring → audience → cost → pairing → diversity
- **Per-provider voice mappings**: Inworld ($0.15/ep), ElevenLabs ($4.50/ep), Google ($0.24/ep), OpenAI ($0.18/ep)
- **Script context injection**: persona shapes how the LLM writes (expression tags, delivery style)
- **5 personas enriched with speech patterns**: example lines, anti-examples, signature phrases
- **Cost-aware selection**: free/trial users get Inworld voices; premium users get ElevenLabs

## User impact

Audio content has consistent, recognizable voices. Users don't choose voices manually — the system picks the best match for the genre and content, then consistently uses that persona across all episodes.
