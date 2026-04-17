# ElevenLabs TTS Voice Expression, Styling, and Optimization

Deep research report covering voice_settings parameters, inline expression tags, voice catalog, content-type styling profiles, model differences, pronunciation control, and fallback considerations.

**Research Date**: 2026-04-02
**Models Covered**: `eleven_multilingual_v2`, `eleven_v3`, `eleven_flash_v2_5`
**Codebase Reference**: `kitesforu-workers/src/workers/stages/audio/providers/elevenlabs/`

---

## Table of Contents

1. [Voice Settings Deep Dive](#1-voice-settings-deep-dive)
2. [Inline Expression Tags (v3 Only)](#2-inline-expression-tags-v3-only)
3. [Voice Catalog Analysis](#3-voice-catalog-analysis)
4. [Content-Type Styling Profiles](#4-content-type-styling-profiles)
5. [Model Differences](#5-model-differences)
6. [Pronunciation and Language](#6-pronunciation-and-language)
7. [Fallback Considerations](#7-fallback-considerations)
8. [Our Current Implementation Assessment](#8-our-current-implementation-assessment)

---

## 1. Voice Settings Deep Dive

### 1.1 stability (0.0 - 1.0)

**What it actually does**: Controls the randomness between each audio generation. Determines how closely the generated voice sticks to the average tone of the reference audio versus allowing deviation for expressiveness.

**Official documentation** (ElevenLabs, verbatim):
> "Determines how stable the voice is and the randomness between each generation. Lower values introduce broader emotional range for the voice. Higher values can result in a monotonous voice with limited emotion."

**Behavioral ranges**:

| Range | Behavior | Best For | Risks |
|---|---|---|---|
| 0.0 - 0.25 | Very expressive, wide emotional variance | Character voices, dramatic fiction | May produce artifacts, speak too fast, "unhinged" delivery |
| 0.25 - 0.40 | Expressive with reasonable consistency | Comedy, storytelling, entertainment | Occasional unexpected tonal shifts |
| 0.40 - 0.60 | **Natural conversation** — balanced | Podcasts, dialogue, interviews | Sweet spot for most use cases |
| 0.60 - 0.75 | Consistent, professional | News, corporate, meditation | Less emotional range |
| 0.75 - 1.0 | Very predictable, near-monotone | Robotic characters, legal narration | Sounds artificial at extremes |

**V3-specific stability modes** (UI terminology):
- **Creative** (low stability): Maximum emotional expression and tag responsiveness. Prone to hallucinations.
- **Natural** (mid stability): Closest to original voice recording. Balanced and neutral.
- **Robust** (high stability): Highly stable but less responsive to audio tags. Consistent, similar to v2 behavior.

**Critical v3 insight**: The stability slider is the MOST IMPORTANT setting in v3. For maximum expressiveness with audio tags, use Creative or Natural. Robust reduces responsiveness to directional prompts (tags).

**Latency impact**: No documented latency impact from changing stability.

### 1.2 similarity_boost (0.0 - 1.0)

**What it actually does**: Controls how closely the AI attempts to replicate the original voice. In the UI this is labeled "Clarity + Similarity Enhancement."

**Official documentation**:
> "Determines how closely the AI should adhere to the original voice when attempting to replicate it."

**Behavioral ranges**:

| Range | Behavior | Best For | Risks |
|---|---|---|---|
| 0.0 - 0.3 | Loose interpretation of original voice | Creative experimentation | Voice may drift from original significantly |
| 0.3 - 0.6 | Moderate resemblance | General use where exact match not needed | Some voice drift is normal |
| 0.6 - 0.8 | **Strong resemblance** — recommended range | Production content, podcasts | Sweet spot for premade voices |
| 0.8 - 1.0 | Maximum fidelity to original | Voice clones where exact match matters | May reproduce artifacts/noise from original recording |

**Interaction with stability**:
- High similarity + low stability = Expressive delivery that still sounds like the target voice. This is the recommended combination for dramatic content.
- High similarity + high stability = Consistent, recognizable voice with limited expression. Good for corporate/news.
- Low similarity + low stability = Unpredictable voice that may not sound like the target. Generally avoid.
- High similarity does NOT inherently make audio more robotic. That is stability's domain.

**Quality note**: If the original voice recording has background noise or artifacts, setting similarity_boost too high will cause the model to reproduce those artifacts.

**Latency impact**: No documented latency impact.

### 1.3 style (0.0 - 1.0)

**What it actually does**: Attempts to amplify the style characteristics of the original speaker. This is separate from emotional expression (stability) -- it tries to exaggerate the speaker's natural speaking patterns.

**Official documentation** (updated Feb 2026):
> "Determines the style exaggeration of the voice. Attempts to amplify the style of the original speaker. Consumes additional computational resources and might increase latency if set to anything other than 0."

**ElevenLabs' own recommendation**: Keep this at 0 at all times.

**Behavioral ranges**:

| Range | Behavior | Latency Impact |
|---|---|---|
| 0.0 | No style exaggeration (neutral) | **No extra latency** |
| 0.0 - 0.3 | Subtle style enhancement | Slight latency increase |
| 0.3 - 0.5 | Moderate style amplification | Noticeable latency increase |
| 0.5 - 0.7 | Strong style exaggeration | Significant latency increase |
| 0.7 - 1.0 | Maximum exaggeration | Maximum latency, may cause instability |

**How it differs from stability**:
- **Stability** controls randomness/variance (emotional range)
- **Style** amplifies the inherent speaking patterns of the voice (cadence, emphasis, quirks)
- Low stability + high style = Maximally expressive AND stylistically exaggerated. Can produce interesting but unpredictable results.
- A neutral voice with style=0.8 will try to exaggerate whatever subtle style patterns exist, which may sound strange.

**Critical note**: Style makes the model slightly less stable (ElevenLabs' words). It consumes additional compute, so non-zero values add latency. For batch podcast generation this is acceptable; for real-time use, keep at 0.

### 1.4 use_speaker_boost (bool)

**What it actually does**: Applies a post-processing enhancement that boosts similarity to the original speaker. It is essentially a more aggressive form of similarity_boost that uses additional computation.

**Official documentation**:
> "This setting boosts the similarity to the original speaker. Using this setting requires a slightly higher computational load, which in turn increases latency."

**When to use**:
- **On (True)**: When voice consistency and recognizability matter. Production podcasts, audiobooks, anything where the voice identity must be maintained.
- **Off (False)**: When latency is critical (real-time agents, live streaming). The differences are "generally rather subtle" per ElevenLabs.

**Latency impact**: Yes, adds latency. The difference is small but real.

### 1.5 speed (0.25 - 4.0)

This parameter was not in the original question but is worth noting. Added to the API in 2025.

**What it does**: Adjusts playback speed of the generated audio. Default is 1.0.

| Value | Effect |
|---|---|
| < 1.0 | Slower speech |
| 1.0 | Normal speed |
| > 1.0 | Faster speech |

**Note**: For v3, speed is better controlled through audio tags like `[speaking slowly]` or `[rushed]` rather than this parameter.

---

## 2. Inline Expression Tags (v3 Only)

### 2.1 Model Compatibility

| Model | Tags Supported | Behavior When Tags Sent |
|---|---|---|
| `eleven_v3` | YES | Interprets and performs tags |
| `eleven_english_v3` | YES | Interprets and performs tags |
| `eleven_multilingual_v2` | NO | Tags are spoken aloud as literal text |
| `eleven_flash_v2_5` | NO | Tags are spoken aloud as literal text |
| `eleven_turbo_v2_5` | NO | Tags are spoken aloud as literal text |

**Our code correctly handles this** -- `elevenlabs_model_supports_tags()` in `tts_expressions.py` checks model, and `_prepare_text()` strips tags for non-v3 models.

### 2.2 Complete Tag Reference

Tags are natural-language words in square brackets. v3 does NOT have a fixed vocabulary -- it interprets tags contextually. However, certain categories are well-tested and reliable.

#### Emotion Tags (control emotional tone)
```
[sad]           [angry]          [happily]        [excited]
[nervous]       [scared]         [confident]      [sorrowful]
[annoyed]       [disgusted]      [surprised]      [proud]
[nostalgic]     [anxious]        [hopeful]        [frustrated]
[bored]         [amused]         [sarcastic]      [suspicious]
[jealous]       [grateful]       [empathetic]     [determined]
[resigned]      [euphoric]       [melancholic]    [wistful]
```

#### Delivery Style Tags (control how speech is delivered)
```
[whispers]      [whispering]     [shouts]         [shouting]
[speaking softly]                [loudly]
[calmly]        [gently]         [firmly]
[urgently]      [rushed]         [speaking slowly]
[drawn out]     [hesitant]       [confident]
[singing]       [sings]          [monotone]
[x accent]      (e.g. [British accent], [Southern accent])
```

#### Human Reaction Tags (non-verbal sounds)
```
[laughs]        [laughing]       [chuckles]       [giggles]
[sighs]         [gasps]          [coughs]         [clears throat]
[gulps]         [sniffs]         [yawns]          [groans]
[breathes deeply]                [sobbing]
[stuttering]    [hiccups]        [sneezes]
```

#### Pause and Pacing Tags
```
[pause]         [long pause]     [dramatic pause]
[beat]          [silence]
```

#### Scene/Atmosphere Tags (experimental, less consistent)
```
[dramatic tone]                  [building intensity]
[suspicious tone]                [mysterious]
[intense]       [eerie]          [foreboding]
[gunshot]       [explosion]      [clapping]       (sound effects -- experimental)
```

### 2.3 Tag Combination

Tags can be stacked for nuanced delivery:

```
[angry][laughing] You really thought I'd fall for that?!
[sad][whispers] I never wanted it to end this way.
[nervous][stuttering] I... I don't think we should be here.
[excited][rushed] Oh my god you have to see this right now!
```

Combining multiple tags produces complex emotional delivery. ElevenLabs encourages experimentation.

### 2.4 Tag Best Practices

1. **Match tags to voice character**: A meditative voice will not shout convincingly. A hype voice will not whisper well. Choose tags that align with the voice's natural range.

2. **Use longer prompts**: Tags work better in prompts over ~250 characters. Short prompts with tags may produce inconsistent results.

3. **Punctuation matters in v3**:
   - Ellipses (...) add pauses and weight
   - CAPITALIZATION increases emphasis
   - Standard punctuation provides natural rhythm
   - Example: `"It was a VERY long day [sigh] ... nobody listens anymore."`

4. **Placement**: Tags BEFORE text affect the following speech. Tags AFTER text act as reactions.
   - `[whispers] Don't make a sound` -- whispered delivery
   - `That's incredible! [laughs]` -- laughs after the statement

5. **Frequency**: Use tags intentionally, not on every line. Over-tagging produces unnatural results.

### 2.5 Our Current Tag Pattern

From `shared.py`, our EXPRESSION_TAG_PATTERN regex captures:
```
whisper(s), shout(s), yell(s), laugh(s), chuckle(s), sigh(s), gasp(s),
dramatic, pause, dramatic pause, long pause,
building, building intensity, rising,
softly, gently, quietly, loudly,
excited(ly), tense, tensely, calm(ly),
sad(ly), angry, angrily,
scared, terrified, horrified,
mysterious(ly), suspense(ful),
warm(ly), cold(ly),
playful(ly), serious(ly),
emotional(ly), intense(ly)
```

**Gap identified**: Our regex does not capture several well-supported v3 tags:
- `[singing]`, `[sings]`, `[monotone]`
- `[nervous]`, `[confident]`, `[sarcastic]`, `[annoyed]`
- `[coughs]`, `[clears throat]`, `[gulps]`, `[sniffs]`
- `[rushed]`, `[hesitant]`, `[drawn out]`
- `[sobbing]`, `[stuttering]`
- Accent tags like `[British accent]`
- Sound effect tags like `[gunshot]`, `[clapping]`

This gap matters for `count_expression_tags()` which boosts style settings based on tag count. Unrecognized tags still get passed through to v3 (which is correct), but they do not contribute to the voice_settings boost calculation.

---

## 3. Voice Catalog Analysis

### 3.1 Key Premade Voices and Their Characteristics

#### Best for Podcasts (2-Host Dialogue)

| Voice | ID | Gender | Age | Accent | Character | Best For |
|---|---|---|---|---|---|---|
| **Rachel** | 21m00Tcm4TlvDq8ikWAM | Female | Young | American | Calm, warm | Narration, host |
| **Adam** | pNInz6obpgDQGcFmaJgB | Male | Deep | American | Deep narrator | Host, authority |
| **Drew** | 29vD33N1CtxCmqQRPOHJ | Male | Mid-age | American | Well-rounded | News, general host |
| **Sarah** | EXAVITQu4vr4xnSDxMaL | Female | Young | American | Soft, clear | News, gentle host |
| **Daniel** | onwK4e9ZLuTAKqWW03F9 | Male | Mid-age | British | Deep, authoritative | News presenter |
| **Alice** | Xb7hH8MSUJpSbSDYk0k2 | Female | Mid-age | British | Confident | News anchor |
| **George** | JBFqnCBsd6RMkjVDRZzb | Male | Mid-age | British | Warm | Default narrator |
| **Antoni** | ErXwobaYiN019PkySvjV | Male | Young | American | Well-rounded | General narrator |
| **Bella** | EXAVITQu4vr4xnSDxMaL | Female | Young | American | Expressive | Conversational |
| **Brian** | nPczCjzI2devNBz1zQrb | Male | Mid-age | American | Deep, funny | Narration, radio |

#### Character/Special Purpose Voices

| Voice | ID | Gender | Character | Best For |
|---|---|---|---|---|
| **Clyde** | 2EiwWnXFnvU5JabPnv8n | Male | War veteran, intense | Video games, drama |
| **Domi** | AZnzlk1XvdvUeBnXmlld | Female | Strong, assertive | Impactful narration |
| **Fin** | D38z5RcWu1voky8WS1ja | Male | Irish sailor | Character work |
| **Glinda** | z9fAnlkpzviPz146aGWa | Female | Theatrical witch | Entertainment |
| **Thomas** | GBv7mTt0atIp3Br8iCZE | Male | Ultra calm | Meditation |
| **Emily** | LcfcDJNUP1GQjkzn1xUU | Female | Calm, soothing | Meditation, relaxation |
| **Nicole** | piTKgcLEGmPE4e6mEKli | Female | Whispery | ASMR |

### 3.2 Voice Pairing Recommendations for Podcasts

**Principle**: Pair voices with contrasting timbres but complementary energy levels. A deep male + bright female creates natural dialogue contrast.

#### Recommended Pairings

| Content Type | Host 1 | Host 2 | Why |
|---|---|---|---|
| **General Podcast** | Adam (deep male) | Rachel (warm female) | Classic contrast, both versatile |
| **News/Current Events** | Drew (professional male) | Sarah (soft female) | Both have news-appropriate delivery |
| **Educational** | George (warm British male) | Alice (confident British female) | Authoritative pair |
| **Comedy** | Brian (deep, funny male) | Domi (strong, assertive female) | Both have character |
| **Storytelling** | Daniel (British authoritative) | Rachel (calm American) | Dramatic range |
| **Interview** | Adam (deep host) | Antoni (young guest) | Host/guest dynamic |

#### Our Current Configuration

From `tts_voices.yaml`, we currently use:
- **ElevenLabs en-US**: Rachel (female) + Adam (male) -- good pairing
- **ElevenLabs _default**: Rachel, Adam, Bella, Antoni -- solid selection

**Assessment**: Our voice selection is solid. Rachel + Adam is one of the strongest premade pairings for podcast dialogue.

### 3.3 V3-Specific Voice Recommendations

ElevenLabs notes that Professional Voice Clones (PVCs) are NOT fully optimized for v3. For v3's audio tag features, use:
1. Premade voices (best tag responsiveness)
2. Instant Voice Clones (IVC) with diverse emotional samples
3. Avoid PVCs with v3 for now

For IVCs targeting v3, include a broader emotional range in training samples than you would for v2 models.

---

## 4. Content-Type Styling Profiles

### 4.1 Your Proposed Settings vs Documentation-Backed Recommendations

The most common baseline recommended by ElevenLabs is: **stability ~0.50, similarity ~0.75-0.80, style 0, speaker_boost on**.

| Content Type | Your Proposal | Recommended | Rationale |
|---|---|---|---|
| **News** | stab=0.65, style=0.3, sim=0.8 | stab=0.65, style=0.0, sim=0.80 | Style adds latency with minimal benefit for news. News needs consistency, not style exaggeration. |
| **Education** | stab=0.5, style=0.4, sim=0.75 | stab=0.50, style=0.15, sim=0.75 | Moderate expression is good. Style 0.4 is higher than needed -- 0.15 adds subtle character without instability. |
| **Comedy** | stab=0.35, style=0.6, sim=0.7 | stab=0.35, style=0.30, sim=0.70 | Low stability is correct for comedic range. Style 0.6 risks artifacts. 0.30 is sufficient for personality. |
| **Storytelling** | stab=0.4, style=0.5, sim=0.75 | stab=0.40, style=0.25, sim=0.75 | Storytelling benefits from expression (low stab), but style 0.5 adds too much exaggeration. |
| **Meditation** | stab=0.8, style=0.1, sim=0.9 | stab=0.80, style=0.0, sim=0.85 | Meditation needs maximum consistency. Style should be 0 for calm. Similarity 0.85 avoids artifact risk from 0.9. |

### 4.2 Complete Recommended Profiles

These profiles are for `eleven_multilingual_v2` and `eleven_flash_v2_5` (no tag support, voice_settings do the heavy lifting). For `eleven_v3`, tags handle most expression, so settings should be closer to baseline.

#### For eleven_multilingual_v2 / eleven_flash_v2_5

```python
CONTENT_VOICE_PROFILES = {
    "news": {
        "stability": 0.65,
        "similarity_boost": 0.80,
        "style": 0.0,
        "use_speaker_boost": True,
    },
    "educational": {
        "stability": 0.50,
        "similarity_boost": 0.75,
        "style": 0.15,
        "use_speaker_boost": True,
    },
    "interview": {
        "stability": 0.50,
        "similarity_boost": 0.75,
        "style": 0.20,
        "use_speaker_boost": True,
    },
    "comedy": {
        "stability": 0.35,
        "similarity_boost": 0.70,
        "style": 0.30,
        "use_speaker_boost": True,
    },
    "storytelling": {
        "stability": 0.40,
        "similarity_boost": 0.75,
        "style": 0.25,
        "use_speaker_boost": True,
    },
    "documentary": {
        "stability": 0.55,
        "similarity_boost": 0.80,
        "style": 0.15,
        "use_speaker_boost": True,
    },
    "meditation": {
        "stability": 0.80,
        "similarity_boost": 0.85,
        "style": 0.0,
        "use_speaker_boost": True,
    },
    "debate": {
        "stability": 0.45,
        "similarity_boost": 0.75,
        "style": 0.20,
        "use_speaker_boost": True,
    },
    "default": {
        "stability": 0.45,
        "similarity_boost": 0.75,
        "style": 0.20,
        "use_speaker_boost": True,
    },
}
```

#### For eleven_v3 (tags handle expression, keep settings neutral)

```python
V3_CONTENT_VOICE_PROFILES = {
    "news": {
        "stability": 0.60,
        "similarity_boost": 0.80,
        "style": 0.0,
        "use_speaker_boost": True,
    },
    "educational": {
        "stability": 0.50,
        "similarity_boost": 0.75,
        "style": 0.10,
        "use_speaker_boost": True,
    },
    "comedy": {
        "stability": 0.35,
        "similarity_boost": 0.70,
        "style": 0.15,
        "use_speaker_boost": True,
    },
    "storytelling": {
        "stability": 0.35,
        "similarity_boost": 0.75,
        "style": 0.15,
        "use_speaker_boost": True,
    },
    "meditation": {
        "stability": 0.75,
        "similarity_boost": 0.85,
        "style": 0.0,
        "use_speaker_boost": True,
    },
    "default": {
        "stability": 0.45,
        "similarity_boost": 0.75,
        "style": 0.10,
        "use_speaker_boost": True,
    },
}
```

**Key principle for v3**: Audio tags are the PRIMARY expression mechanism. voice_settings should be neutral-to-moderate. Over-tuning voice_settings AND using heavy tag usage can produce unpredictable results.

### 4.3 Tag Usage Patterns per Content Type

For `eleven_v3` only:

**News**:
```
Sparse tags. Occasional [pause] between segments.
[confidently] Breaking news tonight...
[concerned] Officials say the situation remains critical.
```

**Educational**:
```
Moderate tags. Curiosity and clarity focus.
[curiously] So what happens when you combine these two elements?
[excited] And that is the key insight!
[pause] Let me explain why this matters.
```

**Comedy**:
```
Rich tags. Timing-focused.
[deadpan] So I told him. No.
[laughs] And he just stood there!
[whispering] But here's the thing nobody talks about...
[shouts] IT WAS A TUESDAY!
```

**Storytelling**:
```
Rich tags. Dramatic range.
[whispers] The door creaked open... [dramatic pause]
[gasps] There was no one there.
[building intensity] And then, the ground began to shake...
[shouts] RUN!
```

**Meditation**:
```
Very sparse tags. Calm pacing only.
[calmly] Take a deep breath in... [long pause] ...and slowly release.
[gently] Let your thoughts drift away like clouds.
[speaking slowly] You are safe. You are at peace.
```

---

## 5. Model Differences

### 5.1 Comparison Matrix

| Feature | eleven_v3 | eleven_multilingual_v2 | eleven_flash_v2_5 |
|---|---|---|---|
| **Quality** | Highest | High | Good |
| **Expressiveness** | Best (tags + natural) | Rich (voice_settings only) | Moderate |
| **Latency** | Higher (~1-2s inference) | Medium | Lowest (~75ms) |
| **Languages** | 70+ | 29 | 32 |
| **Max chars/request** | 5,000 | 10,000 | 40,000 |
| **Cost** | 1 credit/char | 1 credit/char | 0.5 credit/char |
| **Audio tags** | YES | NO | NO |
| **Multi-speaker dialogue** | YES (via API) | Limited | NO |
| **Pronunciation dict** | Alias tags only | Alias tags only | Alias tags only |
| **Phoneme tags (IPA/CMU)** | NO | NO | NO (only eleven_flash_v2) |
| **PVC optimization** | Not fully optimized | Fully optimized | Fully optimized |
| **Real-time suitability** | NO | Moderate | YES |
| **Best for** | Dramatic content, audiobooks, podcasts | Production content, multilingual | Real-time agents, cost optimization |

### 5.2 When to Use Which

**Use eleven_v3 when**:
- Audio quality and expressiveness are the top priority
- Content is dramatic: storytelling, audiobooks, character work
- Batch generation (latency is not a concern)
- Multi-speaker dialogue is needed
- You want audio tag control over delivery
- English content (v3 is strongest in English, decent in other languages)

**Use eleven_multilingual_v2 when**:
- Non-English content where language fidelity matters
- Consistent, predictable output is more important than dramatic range
- Working with Professional Voice Clones
- Need longer text per request (10,000 char limit)
- Want proven, stable production model

**Use eleven_flash_v2_5 when**:
- Real-time applications (voice agents, live streaming)
- Cost is a primary concern (50% cheaper)
- Latency must be under 100ms
- Quality is "good enough" (not premium)
- Large batch jobs where cost adds up

### 5.3 Quality Architecture Differences

From ElevenLabs' latency documentation:

> "Flash models are smaller and use more aggressive approximations. They sacrifice some quality headroom to substantially reduce inference time. Eleven v3 uses a larger model with a higher-fidelity voice codec, which takes longer to run but produces richer, more emotionally nuanced audio."

> "This is a genuine tradeoff, not a technical limitation that will eventually go away."

This means: there will never be a v3-quality model at Flash speeds. The quality comes from the additional computation.

### 5.4 eleven_turbo_v2_5 Status

**Deprecated**. Functionally equivalent to `eleven_flash_v2_5` with higher latency. ElevenLabs recommends Flash over Turbo in all cases.

Our codebase still lists it in `SUPPORTED_MODELS`. This should be removed.

---

## 6. Pronunciation and Language

### 6.1 Pronunciation Dictionaries

ElevenLabs supports pronunciation customization through dictionaries using `.pls` (PLS/XML) files. Two rule types:

**Phoneme Rules** (IPA or CMU):
```xml
<lexeme>
  <grapheme>tomato</grapheme>
  <phoneme>/te'meItoU/</phoneme>
</lexeme>
```

**CRITICAL LIMITATION**: Phoneme tags ONLY work with `eleven_flash_v2` and `eleven_monolingual_v1`. They silently skip on ALL other models including v3, multilingual_v2, and flash_v2_5. This is a significant limitation.

**Alias Rules** (word substitution -- works on ALL models):
```xml
<lexeme>
  <grapheme>UN</grapheme>
  <alias>United Nations</alias>
</lexeme>
```

Alias rules are the universal approach. They work by replacing a word with an alternative spelling/phrase that produces the desired pronunciation.

### 6.2 Handling Proper Nouns

Since phoneme-based pronunciation only works on two legacy models, the practical approach for our pipeline is:

1. **Alias dictionaries**: Create aliases that spell out proper nouns phonetically
   - `"Vikrant" -> "Vik-rahnt"` (guides pronunciation through respelling)
   - `"Kubernetes" -> "Koo-ber-net-ees"`
   - `"gRPC" -> "gee R P C"`

2. **API integration**: Dictionaries are applied per-request by attaching `pronunciation_dictionary_locators` to the TTS call.

3. **Recent API additions** (March 2026): New `case_sensitive` and `word_boundaries` fields for more precise rule matching. New `set-rules` endpoint to replace all rules in one call.

### 6.3 Code-Switching (English with Foreign Words)

**For eleven_multilingual_v2**: The model handles code-switching reasonably well within its 29 supported languages. When encountering a foreign word in an English sentence, it attempts contextual pronunciation. Sending `language_code` in the request helps anchor the primary language. Our code already does this:

```python
if model == "eleven_multilingual_v2" and not language.startswith("en"):
    payload["language_code"] = language.split("-")[0]
```

**For eleven_v3**: Supports 70+ languages. Better at code-switching than v2 due to broader training data. No explicit language_code needed for most cases -- v3 auto-detects.

**For eleven_flash_v2_5**: Supports 32 languages. Decent code-switching but less sophisticated than v3 or multilingual_v2.

### 6.4 Language Auto-Detection

- **v3**: Yes, auto-detects language from text context. Best auto-detection of all models.
- **multilingual_v2**: Has auto-detection but benefits from explicit `language_code` for accuracy.
- **flash_v2_5**: Auto-detects but less reliable for mixed-language content.

**Recommendation**: Always send explicit `language_code` for multilingual_v2 and flash_v2_5. For v3, it is optional but harmless.

### 6.5 Text Normalization

Models differ in how they handle numbers, symbols, and abbreviations:
- **multilingual_v2**: Best text normalization. Correctly reads "$1,000,000" as "one million dollars."
- **flash_v2_5**: Smaller model, less sophisticated normalization. May read "$1,000,000" as "one thousand thousand dollars."
- **v3**: Good normalization, comparable to multilingual_v2.

**Important**: The `optimize_streaming_latency` parameter at level 4 disables text normalization entirely, causing mispronunciation of numbers and dates. Never use level 4 for podcast content.

---

## 7. Fallback Considerations

### 7.1 Our Current Fallback Chain

From memory: `Inworld -> ElevenLabs -> OpenAI -> Google` (circuit breaker aware)

### 7.2 Expression Features Lost When Falling Back FROM ElevenLabs

| Feature | ElevenLabs | OpenAI tts-1 | Google Cloud TTS |
|---|---|---|---|
| Audio tags (v3) | YES -- inline tags | NO -- stripped, ignored | NO -- stripped, ignored |
| voice_settings | YES -- full control | NO -- speed parameter only | Partial -- SSML prosody |
| Emotion via stability | YES | NO | NO |
| Style exaggeration | YES | NO | NO |
| Speaker boost | YES | NO | NO |
| Pronunciation dict | Alias only (universal) | NO | SSML phoneme tags |
| Pause control | Tags: [pause] | Ellipses only | SSML: `<break time="500ms"/>` |
| Whisper/shout | Tags: [whispers], [shouts] | NO | SSML prosody approximation |
| Non-verbal (laugh, sigh) | Tags: [laughs], [sighs] | NO | NO |

### 7.3 Mapping voice_settings to Other Providers

**voice_settings do NOT exist in other providers.** There is no direct mapping. The adaptation strategy is:

**ElevenLabs -> OpenAI**:
- `stability` -> Partially mapped via speed parameter: low stability (expressive) maps to slightly higher speed for energy
- `similarity_boost` -> No equivalent (OpenAI voices are fixed)
- `style` -> No equivalent
- `use_speaker_boost` -> No equivalent
- Emotion metadata -> `instructions` parameter for gpt-4o-mini-tts, or speed mapping for tts-1

**ElevenLabs -> Google Cloud TTS**:
- `stability` -> SSML `<prosody>` rate/pitch approximation
- Low stability (expressive) -> `<prosody rate="medium" pitch="+1st">`
- High stability (calm) -> `<prosody rate="slow" pitch="-1st">`
- Pauses -> `<break time="Xms"/>` (direct SSML support)

Our codebase already has this adaptation logic in `capabilities.py` and `tts_expressions.py`.

### 7.4 Tag Stripping Quality Impact

When falling from v3 (with tags) to a non-tag provider:

**What is lost**:
- Non-verbal sounds ([laughs], [sighs]) -- completely lost, no equivalent
- Delivery changes ([whispers], [shouts]) -- partially recoverable via SSML for Google, lost for OpenAI
- Emotional tones ([sad], [excited]) -- partially recoverable via metadata/instructions
- Pauses ([pause], [dramatic pause]) -- recoverable via SSML or silence insertion

**Quality degradation**: Significant for dramatic/storytelling content, moderate for educational, minimal for news (which uses sparse tags anyway).

**Our code handles this correctly**: `strip_expression_tags()` in `tts_expressions.py` removes all `[bracket]` tags and SSML tags, cleaning up whitespace. `adapt_script_for_provider()` in `capabilities.py` converts emotions to the target provider's format.

---

## 8. Our Current Implementation Assessment

### 8.1 What We Are Doing Right

1. **Tag-aware model routing**: `elevenlabs_model_supports_tags()` correctly gates tag usage to v3/english_v3 only
2. **Tag stripping for non-v3**: `_prepare_text()` strips tags when sending to multilingual_v2 or flash
3. **Emotion-driven voice_settings**: `render_voice_settings()` maps emotion metadata to stability/style deltas
4. **Content-type expression profiles**: `CONTENT_EXPRESSION_PROFILES` in `tts_expressions.py` provides per-content-type tag guidance
5. **Provider capability matrix**: `capabilities.py` ensures no unsupported features are sent to any provider
6. **Multi-transport support**: Both REST and WebSocket providers with consistent voice_settings logic

### 8.2 Improvement Opportunities

1. **Style parameter is too high in baseline**:
   - Current: `style: 0.35` in NATURAL_BASELINE
   - ElevenLabs recommends: `style: 0` (or close to it)
   - Our style adds latency and instability for minimal benefit
   - **Recommendation**: Lower to 0.15 for baseline, use content-type profiles for higher values

2. **Missing content-type voice_settings profiles**:
   - We have emotion-based deltas but not content-type-based baseline profiles
   - A meditation podcast should start at stability=0.80, not 0.45
   - **Recommendation**: Implement the CONTENT_VOICE_PROFILES from Section 4.2

3. **Tag regex is incomplete**:
   - Missing several well-supported v3 tags (see Section 2.5)
   - `count_expression_tags()` underestimates tag density
   - This affects the tag_boost calculation in `render_voice_settings()`
   - **Recommendation**: Broaden the regex or switch to a simpler `\[...\]` pattern and count all bracket expressions

4. **No content-type-aware voice_settings for v3**:
   - v3 needs different baseline settings than v2 because tags handle expression
   - Using the same NATURAL_BASELINE for both means v3 gets double-expression (tags + settings)
   - **Recommendation**: Implement V3_CONTENT_VOICE_PROFILES with lower style values

5. **Deprecated model in supported list**:
   - `eleven_turbo_v2_5` is deprecated, should be removed from SUPPORTED_MODELS
   - **Recommendation**: Remove and map to `eleven_flash_v2_5`

6. **No pronunciation dictionary integration**:
   - We have no mechanism to handle proper nouns consistently
   - For educational/technical podcasts, this causes mispronunciation
   - **Recommendation**: Create alias-based pronunciation dictionaries for common technical terms

7. **optimize_streaming_latency not set for REST**:
   - Our REST provider sends `"optimize_streaming_latency": 2` in the payload
   - This parameter is a QUERY parameter on streaming endpoints only, not a body parameter on regular REST
   - The regular REST endpoint ignores it, so this is harmless but misleading
   - **Recommendation**: Remove it from the REST payload to avoid confusion

---

## Sources

1. ElevenLabs Voice Settings Documentation: https://elevenlabs.io/docs/api-reference/voices/settings/update
2. ElevenLabs Models Reference: https://elevenlabs.io/docs/overview/models
3. ElevenLabs Best Practices (v3): https://elevenlabs.io/docs/overview/capabilities/text-to-speech/best-practices
4. ElevenLabs Audio Tags Blog: https://elevenlabs.io/blog/v3-audiotags
5. ElevenLabs Audio Tags Delivery Control: https://elevenlabs.io/blog/eleven-v3-audio-tags-precision-delivery-control-for-ai-speech
6. ElevenLabs Latency Documentation: https://elevenlabs.io/docs/eleven-api/concepts/latency
7. ElevenLabs Model Selection Guide: https://elevenlabs.io/docs/eleven-api/choosing-the-right-model
8. ElevenLabs Pronunciation Dictionaries: https://elevenlabs.io/docs/eleven-api/guides/how-to/text-to-speech/pronunciation-dictionaries
9. ElevenLabs Changelog (Feb-Mar 2026): https://elevenlabs.io/docs/changelog
10. ElevenLabs v3 Complete Guide: https://audio-generation-plugin.com/elevenlabs-v3/
11. ElevenLabs Premade Voices: https://elevenlabs-sdk.mintlify.app/voices/premade-voices
12. Inworld v3 Review (competitive analysis): https://inworld.ai/resources/elevenlabs-v3-review
13. ElevenLabs Python SDK Models: https://mintlify.com/elevenlabs/elevenlabs-python/models
14. ElevenLabs Voice Settings SDK: https://mintlify.com/elevenlabs/elevenlabs-python/api-reference/voices/settings
15. Our internal pipeline optimization research: `kitesforu-docs/human/research/pipeline-optimization/elevenlabs-api-optimization-research.md`
