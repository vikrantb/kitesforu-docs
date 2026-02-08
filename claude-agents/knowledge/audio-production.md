# Audio Production & Voice Expertise
<!-- last_verified: 2026-02-07 -->

## TTS Provider Comparison (KitesForU)

### OpenAI TTS
- Models: tts-1 ($15/1M chars), tts-1-hd ($30/1M chars)
- Voices: alloy, echo, fable, onyx, nova, shimmer
- Strengths: Consistent, reliable, good English
- Weaknesses: Limited emotion range, fewer languages, less natural than ElevenLabs
- Best for: Trial/Enthusiast tiers (cost-effective)

### ElevenLabs
- Models: eleven_flash_v2_5 ($150/1M), eleven_turbo_v2_5 ($150/1M), eleven_multilingual_v2 ($300/1M)
- Strengths: Most natural sounding, best emotion range, voice cloning capability, 29 languages
- Weaknesses: 10-20x more expensive than OpenAI, occasional latency spikes
- Best for: Pro/Ultimate tiers (premium quality)

### Google Cloud TTS
- Model: gcp-neural2 ($16/1M chars)
- Strengths: Cheap, reliable, many languages
- Weaknesses: Robotic compared to others, less emotional range
- Best for: Fallback provider

## Voice-Persona Matching
For interview prep specifically:
- Coach persona: Warm, authoritative voice (OpenAI: onyx/nova, ElevenLabs: select warm male/female)
- Interviewer persona: Professional, slightly formal (OpenAI: echo/fable)
- Mixed-gender pairs improve engagement 15-20% vs same-gender
- Match voice energy to content: encouraging for coaching, neutral for mock interviews

## Professional Audio Standards
- Loudness: -16 LUFS for streaming, -19 LUFS for podcasting
- Dynamic range: 6-12 dB for spoken word
- Pacing: 150-170 words/minute for educational, 120-140 for storytelling
- Silence: 0.5-1s between segments, 2-3s for topic transitions

## Script Pacing Markers
- [PAUSE] — 0.5-1 second silence, used between thoughts
- [LONG_PAUSE] — 2-3 seconds, used for topic transitions or "think about this" moments
- [EMPHASIS] — Slight increase in volume/energy on following phrase
- [TRANSITION] — Musical or tonal shift between segments

## Audio Quality Diagnostics
| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Robotic sound | TTS provider/model issue | Upgrade tier or check fallback |
| Unnatural pauses | Script pacing markers wrong | Fix [PAUSE] placement in script prompt |
| Monotone delivery | Script lacks emphasis markers | Add [EMPHASIS] to script generation prompt |
| Abrupt transitions | Missing [TRANSITION] markers | Fix in script generation or outline |
| Audio artifacts | Segment stitching issue | Check audio worker logs |
| Wrong language accent | TTS language parameter wrong | Verify language code passed to TTS |

## KitesForU Audio Pipeline
- Script -> segments (split by speaker/section) -> TTS per segment -> stitch -> master
- Fallback: If primary TTS fails, falls back through provider chain
- Fallback history tracked in `podcast_jobs.tts_calls`
- Debug: /debug/{jobId} shows TTS segment log with voice info
