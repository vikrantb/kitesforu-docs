# TTS Voice System Architecture

## Overview

KitesForU uses three TTS (Text-to-Speech) providers to deliver natural-sounding podcast audio in 20+ languages. The system automatically selects the best provider based on language and user subscription tier.

## Providers

| Provider | Best For | Pricing |
|----------|----------|---------|
| **OpenAI** | English (optimized) | $15/million chars |
| **Google Cloud TTS** | Non-English (native voices) | $16/million chars |
| **ElevenLabs** | Premium tier (expressive) | $150-300/million chars |

## How Provider Selection Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Request                                  │
│            (language: hi-IN, tier: pro_creator)                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VoiceSelector                                 │
│                                                                  │
│  1. Check tier → pro_creator → ElevenLabs                       │
│  2. Check if ElevenLabs has Hindi voice → YES                   │
│  3. Return: ElevenLabs voice "Rahul" (hi-IN)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      TTSClient                                   │
│                                                                  │
│  Routes to: _generate_elevenlabs()                              │
│  Voice ID: bVMeCyTHy58xNoL34h3p                                 │
│  Output: Audio bytes (MP3)                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Selection Rules

1. **Premium Tier + Voice Available**: Use tier's provider (ElevenLabs)
2. **Otherwise**: Use native provider based on language:
   - English → OpenAI (optimized for English)
   - Non-English → Google (native speaker voices)

## Voice Configuration

All voices are configured in `config/tts_voices.yaml`. No code changes needed to add voices.

### Finding a Voice's Provider

**Q: What provider is "Priya"?**

Look in `tts_voices.yaml`:
```yaml
voices:
  hi-IN:
    google:
      - voice_id: "hi-IN-Neural2-D"
        name: "Priya"
        gender: "FEMALE"
```

**A: Priya is a Google Cloud TTS Neural2 voice for Hindi.**

### Available Voices by Language

| Language | Google Voices | ElevenLabs Voices |
|----------|---------------|-------------------|
| Hindi (hi-IN) | Arjun, Ananya, Vikram, Priya | Rahul, Sneha |
| English (en-US) | Matthew, Sarah, James, Emily | Rachel, Adam |
| Spanish (es-ES) | Carlos, Sofia, Diego, Maria | - |
| French (fr-FR) | Pierre, Elise, Jean, Claire | - |
| German (de-DE) | Hans, Eva, Stefan, Greta | - |
| Japanese (ja-JP) | Takeshi, Yuki, Hiroshi | - |
| ... | See config for full list | |

## Adding a New Voice

1. Open `config/tts_voices.yaml`
2. Find the language section (or create new)
3. Add voice under provider:

```yaml
voices:
  ta-IN:  # Tamil
    google:
      - voice_id: "ta-IN-Neural2-A"
        name: "Kumar"
        gender: "MALE"
      - voice_id: "ta-IN-Neural2-C"
        name: "Lakshmi"
        gender: "FEMALE"
```

4. No code changes needed!

## Tier-Based Routing

| Tier | Default Provider | Model |
|------|------------------|-------|
| Trial | OpenAI | tts-1 |
| Enthusiast | OpenAI | tts-1 |
| Pro Creator | ElevenLabs | eleven_flash_v2_5 |
| Ultimate Creator | ElevenLabs | eleven_multilingual_v2 |

## Emotion Support (Planned)

### Current State
- **ElevenLabs**: Supports emotion via `voice_settings` (stability, style)
- **Google**: Supports SSML prosody tags (not yet implemented)
- **OpenAI**: Limited emotion control

### Future Architecture

Script generation will include emotion metadata:
```json
{
  "speaker": "Host1",
  "text": "Welcome to our show!",
  "emotion": "warm",
  "intensity": 0.7
}
```

Provider adapters will convert generic emotions to provider-specific formats:
- **ElevenLabs**: `{stability: 0.5, style: 0.5}`
- **Google**: `<prosody rate="105%" pitch="+1st">...</prosody>`

## Related Files

| File | Purpose |
|------|---------|
| `config/tts_voices.yaml` | Voice configuration (YAML) |
| `voice_selector.py` | Voice selection logic |
| `tts_client.py` | TTS API calls |
| `llm/reference/tts-architecture.yaml` | Full technical spec |

## FAQ

**Q: Why does Hindi use Google instead of OpenAI?**
A: OpenAI voices are English-native. They pronounce Hindi with an English accent. Google has native Hindi speakers for authentic pronunciation.

**Q: Can premium users still get Google voices?**
A: Yes. If ElevenLabs doesn't have a voice for a language, the system falls back to Google.

**Q: How do I add support for a new language?**
A: Add the language to `language_routing` in the YAML and add voice definitions under `voices`.
