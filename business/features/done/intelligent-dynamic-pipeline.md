# Intelligent Dynamic Pipeline

**Status**: DONE
**Shipped**: April 4, 2026
**Repos**: kitesforu-workers
**Performance**: 211s → 53s (75% faster), $0.083/episode

## What it does

One Intelligence Call at the start of each job analyzes the topic and dynamically configures the entire pipeline: research depth, script temperature, pause profile, pronunciation hints, content sensitivity, audience level.

## Key capabilities

- **Freshness Hint (regex)** → determines if topic is evergreen or news/current
- **Discovery LLM** → produces an EpisodeProfile with skip_research, script_temperature, pause_profile, pronunciation_hints, content_type, sensitivity, audience_level
- **Evergreen topics**: skip research (saves 3-8s, 30-40% of topics)
- **News/current**: deep research (depth 4-5)
- **Comedy**: high temperature (0.85), creative dialogue
- **Health/finance**: sensitivity=high, temp capped at 0.45, deep research
- **Variable pauses**: conversational=250ms, dramatic=500ms, rapid=150ms
- **LUFS normalization**: -16 LUFS broadcast standard
- **Provider adaptation**: capabilities matrix prevents unsupported features

## User impact

Every piece of content is automatically tuned for its topic. Users don't configure anything — the system decides whether to research, how creative to be, how fast to pace, and which audio features to use.
