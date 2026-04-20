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

## Audio Expression System

The TTS expression system enables natural, expressive audio by embedding provider-specific audio tags directly in generated scripts. The system is fail-safe—if tags can't be processed, they're stripped without breaking audio generation.

### Provider Capabilities

| Provider | Capability | Tag Format | Example |
|----------|------------|------------|---------|
| **ElevenLabs** | Full expressions | Square brackets | `[whispers]Hello[/whispers]` |
| **Google** | SSML prosody | XML tags | `<prosody rate="slow">Hello</prosody>` |
| **OpenAI** | Basic (stripped) | N/A | Tags removed before TTS |

### How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│                   Script Generation                               │
│                                                                    │
│  1. Detect content type (storytelling, meditation, news, etc.)   │
│  2. Get TTS provider from tier selection                          │
│  3. Generate expression_guidance for that provider                │
│  4. LLM generates script WITH provider-specific tags              │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                      TTS Client                                    │
│                                                                    │
│  ElevenLabs → Pass tags through (native support)                  │
│  Google → Convert to SSML or strip non-SSML tags                  │
│  OpenAI → Strip ALL tags (no expression support)                  │
└──────────────────────────────────────────────────────────────────┘
```

### ElevenLabs Audio Tags

Supported audio tags for eleven_flash_v2_5 and eleven_multilingual_v2:

| Tag | Effect | Use Case |
|-----|--------|----------|
| `[whispers]...[/whispers]` | Soft, breathy | Secrets, intimacy |
| `[laughs]` | Natural laugh | Comedy, joy |
| `[sighs]` | Exhale | Frustration, relief |
| `[gasps]` | Sharp intake | Surprise, shock |
| `[dramatic tone]` | Theatrical delivery | Suspense, reveals |
| `[sad tone]` | Somber voice | Emotional moments |
| `[excited]` | Energetic delivery | Enthusiasm |

### Content Type Profiles

The system maps content types to appropriate expression sets:

| Content Type | Primary Expressions | Intensity Range |
|--------------|---------------------|-----------------|
| **storytelling** | dramatic, whispers, gasps | 0.5-0.9 |
| **comedy** | laughs, excited, dramatic | 0.6-1.0 |
| **meditation** | calm, soothing, gentle | 0.2-0.5 |
| **news** | professional, neutral | 0.4-0.6 |
| **educational** | warm, clear, engaging | 0.4-0.7 |

### Script Output with Expressions

```json
{
  "speaker": "Narrator",
  "text": "[whispers]Listen carefully...[/whispers] The door creaked open. [gasps] Someone was already inside.",
  "emotion": "suspenseful",
  "intensity": 0.8,
  "rate": 0.9
}
```

### Fallback Behavior

When provider fallback occurs (e.g., ElevenLabs → OpenAI due to quota):

1. **Tag Detection**: TTS client detects provider mismatch
2. **Tag Stripping**: `strip_expression_tags()` removes all expression tags
3. **Clean Text**: Pure text sent to fallback provider
4. **No Errors**: Process completes successfully with basic voice

```python
# Fail-safe: strips [tags] and <ssml> before sending to OpenAI
clean_text = strip_expression_tags(text_with_tags)
# Result: "Listen carefully... The door creaked open. Someone was already inside."
```

### Emotion Metadata

In addition to inline tags, dialogue items include emotion metadata:

```json
{
  "speaker": "Host1",
  "text": "Welcome to our show!",
  "emotion": "warm",
  "intensity": 0.7,
  "rate": 1.0
}
```

This metadata is used for:
- TTS voice settings (ElevenLabs stability/style)
- Pacing control (rate affects speaking speed)
- Analytics and quality tracking

## Related Files

| File | Purpose |
|------|---------|
| `config/tts_voices.yaml` | Voice configuration (YAML) |
| `voice_selector.py` | Voice selection logic |
| `tts_client.py` | TTS API calls, tag stripping |
| `tts_expressions.py` | Expression system, provider renderers |
| `llm/reference/tts-architecture.yaml` | Full technical spec |

## FAQ

**Q: Why does Hindi use Google instead of OpenAI?**
A: OpenAI voices are English-native. They pronounce Hindi with an English accent. Google has native Hindi speakers for authentic pronunciation.

**Q: Can premium users still get Google voices?**
A: Yes. If ElevenLabs doesn't have a voice for a language, the system falls back to Google.

**Q: How do I add support for a new language?**
A: Add the language to `language_routing` in the YAML and add voice definitions under `voices`.
