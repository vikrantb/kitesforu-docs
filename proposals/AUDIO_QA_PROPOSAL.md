# Audio & Content Quality Assurance System Proposal

> **Status**: Draft Proposal for Review
> **Created**: 2026-01-04
> **Updated**: 2026-01-04 (v2 - Added glossary, content QA, honest tradeoffs)

---

## Plain English Glossary

Before diving in, here's what all the technical terms mean:

| Term | What It Means | Analogy |
|------|---------------|---------|
| **WER** (Word Error Rate) | How many words the TTS got wrong compared to the script. If script says "quick brown fox" but audio says "quick brown box" → 1 wrong out of 4 = 25% error rate. **Lower is better.** | Like a spelling test score, but for pronunciation |
| **MOS** (Mean Opinion Score) | A 1-5 star rating for how good audio sounds. 5 = sounds like a real human, 1 = robotic garbage. We use AI to predict what humans would rate it. | Like a Yelp rating for audio quality |
| **Prosody** | The "music" of speech - rhythm, melody, emphasis, pauses. A robot reads everything flat and monotone. Good prosody means natural rise and fall, proper pauses, emphasis on important words. | Like the difference between reading a bedtime story expressively vs. reading a legal document |
| **STT** (Speech-to-Text) | Listen to audio, write down what was said. We use this to check if words were pronounced correctly. | Like a court stenographer for our audio |
| **Pitch / F0** | How high or low a voice sounds, measured in Hz. Women typically 165-255 Hz, men 85-180 Hz. Variation in pitch = more expressive and natural. | Like notes on a piano - high notes vs low notes |
| **Dynamic Range** | The difference between quiet and loud parts of audio. Too flat = boring, too much variation = jarring. | Like the difference between whispers and shouts in a movie |
| **Bitrate / Sample Rate** | Technical specs for audio file quality. Higher = better quality but larger files. 44.1kHz/128kbps is the minimum for good podcasts. | Like resolution for images (720p vs 4K) |

---

## Executive Summary

This proposal outlines a **5-stage quality system** for KitesForU podcasts:

| Stage | What It Checks | Tool | Cost |
|-------|---------------|------|------|
| 1. Format | Is the audio file valid? | ffprobe | FREE |
| 2. Pronunciation | Were words spoken correctly? | Whisper + jiwer | FREE |
| 3. Audio Quality | Does it sound good? | UTMOS | FREE |
| 4. Prosody | Is it natural, not robotic? | librosa | FREE |
| 5. **Content** | Is the SCRIPT itself good? | LLM evaluation | ~$2/month |

**Bottom Line**: The entire system can run for **$0-2/month** using open-source tools, with no sacrifice in quality.

---

## The Honest Truth: Free vs Paid

### Are Free Tools "Good Enough"?

**YES.** Here's why this isn't hand-wavy:

| Tool | Free Version | Paid Version | Quality Difference |
|------|--------------|--------------|-------------------|
| **Whisper STT** | Whisper running on your computer | Whisper API ($0.006/min) | **IDENTICAL** - same model, same accuracy |
| **MOS Scoring** | UTMOS (research model) | No paid alternative exists | N/A - this IS the tool everyone uses |
| **WER Calculation** | jiwer library | N/A | It's just math - no "premium math" |
| **Prosody Analysis** | librosa | N/A | Industry standard, used by Spotify |

### The REAL Tradeoffs (Honest Assessment)

| Concern | Free | Paid | Verdict |
|---------|------|------|---------|
| **Accuracy** | 95-98% | 95-98% | **Same** - Whisper local = Whisper API |
| **Speed** | Slower without GPU | Fast cloud servers | **Free wins if you have GPU**, otherwise add ~5 min processing |
| **Infrastructure** | You manage it | Vendor manages | **Paid is easier** if you don't want to deal with GPUs |
| **Support** | Community/docs only | Vendor SLA | **Paid has support** (but do you need it for QA?) |
| **Scaling** | Need more GPUs | Just pay more | **Paid scales easier** at very high volume |

### My Recommendation

```
Volume < 1000 podcasts/month → FREE (save money, quality is identical)
Volume > 1000 AND no GPU     → Whisper API at $0.006/min (~$60/month for 10K min)
Need vendor support/SLA      → Use APIs, but know you're paying for convenience, not quality
```

### What Could Go Wrong with Free?

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Whisper is slow on CPU | High (if no GPU) | Processing takes 5-10 min per podcast | Use GPU, or process in background overnight |
| UTMOS gives wrong score | Low | Might pass bad audio or fail good audio | Validate against human ratings monthly |
| Library breaks after update | Low | QA pipeline stops working | Pin versions, test before upgrading |

**Honest take**: If you have a GPU (even a basic one), free is genuinely the right choice. The "premium" APIs use the exact same models.

---

## Stage 1: Format Validation

**What it does**: Checks if the audio FILE itself is valid (not corrupted, right format)

**In plain English**: Before we check anything else, make sure the audio file isn't broken.

**Checks**:
| What We Check | What's Good | What's Bad |
|---------------|-------------|------------|
| File readable? | Opens without error | Corrupted, won't open |
| Format | MP3 or WAV | Unknown format |
| Duration | Matches expected ±5% | Way too short or long |
| Sample rate | 44100 or 48000 Hz | Below 22050 Hz (sounds bad) |
| Bitrate | ≥128 kbps | Below 96 kbps (sounds compressed) |

**Tool**: ffprobe (FREE, part of FFmpeg)

**Code**:
```python
import subprocess
import json

def check_audio_format(audio_path: str) -> dict:
    """Check if audio file is valid and get its specs."""
    cmd = [
        'ffprobe', '-v', 'quiet', '-print_format', 'json',
        '-show_format', '-show_streams', audio_path
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)

    if result.returncode != 0:
        return {"valid": False, "error": "File is corrupted or unreadable"}

    data = json.loads(result.stdout)
    return {
        "valid": True,
        "format": data['format']['format_name'],
        "duration_seconds": float(data['format']['duration']),
        "bitrate": int(data['format'].get('bit_rate', 0)),
    }
```

---

## Stage 2: Pronunciation Accuracy

**What it does**: Checks if words were spoken correctly

**In plain English**:
1. Listen to the audio and write down what was said (STT)
2. Compare that to the original script
3. Calculate how many words are wrong

**The Math (WER)**:
```
WER = (Wrong Words + Missing Words + Extra Words) / Total Words in Script

Example:
Script:     "The quick brown fox jumps"
Audio said: "The quick brown box jumps"
                            ^^^^ wrong!
WER = 1 wrong / 5 total = 20% error rate
```

**What's Good vs Bad**:
| WER | Quality | What To Do |
|-----|---------|------------|
| < 5% | Excellent | Ship it |
| 5-10% | Good | Ship it, maybe log |
| 10-15% | Okay | Review, maybe regenerate |
| > 15% | Bad | Regenerate with different settings |

**Tools**:
- Whisper (FREE, runs locally) - listens and transcribes
- jiwer (FREE) - calculates error rate

**Code**:
```python
import whisper
from jiwer import wer

def check_pronunciation(audio_path: str, original_script: str) -> dict:
    """Check if words were pronounced correctly."""

    # Step 1: Listen to audio and transcribe it
    model = whisper.load_model("large-v3")  # Best accuracy
    result = model.transcribe(audio_path)
    what_was_said = result["text"]

    # Step 2: Compare to script
    error_rate = wer(original_script, what_was_said)

    return {
        "what_was_said": what_was_said,
        "error_rate": error_rate,
        "accuracy": 1 - error_rate,  # 0.95 = 95% accurate
        "passed": error_rate < 0.10,  # Pass if < 10% errors
    }
```

---

## Stage 3: Audio Quality (MOS Score)

**What it does**: Predicts how good the audio SOUNDS to human ears

**In plain English**: Instead of asking 100 people "rate this 1-5 stars", we use AI that was trained on millions of human ratings to predict the score.

**The Scale**:
| MOS Score | What It Means | Example |
|-----------|---------------|---------|
| 4.5-5.0 | Excellent - sounds like a real human | Professional podcast |
| 4.0-4.5 | Good - natural with minor imperfections | Good TTS |
| 3.5-4.0 | Okay - clearly synthetic but acceptable | Average TTS |
| 3.0-3.5 | Poor - robotic, distracting artifacts | Old TTS |
| < 3.0 | Bad - unlistenable | Broken audio |

**Tool**: UTMOS (FREE) - Used by actual TTS researchers to evaluate quality

**Code**:
```python
from speechmos import UTMOS

def check_audio_quality(audio_path: str) -> dict:
    """Predict how good the audio sounds (1-5 scale)."""
    predictor = UTMOS()
    score = predictor.predict(audio_path)

    return {
        "mos_score": score,
        "quality": "Excellent" if score >= 4.5 else
                   "Good" if score >= 4.0 else
                   "Okay" if score >= 3.5 else
                   "Poor" if score >= 3.0 else "Bad",
        "passed": score >= 3.5,
    }
```

---

## Stage 4: Prosody (Naturalness)

**What it does**: Checks if the speech sounds natural or robotic

**In plain English**: A robot reads everything in a flat, monotone voice. A human varies their voice - higher for questions, lower for serious topics, pauses for emphasis. This stage measures those variations.

**What We Measure**:
| Metric | What It Means | Good Range | Bad Sign |
|--------|---------------|------------|----------|
| Pitch variation | Does voice go up and down? | 30-80 Hz standard deviation | < 20 Hz = monotone |
| Speaking rate | How fast are words spoken? | 120-180 words/min | Too fast or too slow |
| Dynamic range | Quiet vs loud variation | 6-15 dB | < 4 dB = flat |
| Pause patterns | Natural breathing pauses | Present and varied | None or too regular |

**Tool**: librosa (FREE) - Used by Spotify, music apps, research labs

**Code**:
```python
import librosa
import numpy as np

def check_naturalness(audio_path: str) -> dict:
    """Check if speech sounds natural (not robotic)."""
    audio, sample_rate = librosa.load(audio_path)

    # Measure pitch variation
    pitches, _, _ = librosa.pyin(audio, fmin=50, fmax=500, sr=sample_rate)
    pitch_variation = np.nanstd(pitches)  # How much does pitch vary?

    # Measure energy/loudness variation
    energy = librosa.feature.rms(y=audio)[0]
    dynamic_range = 20 * np.log10(np.max(energy) / np.mean(energy))

    is_monotone = pitch_variation < 20  # Red flag!

    return {
        "pitch_variation_hz": float(pitch_variation),
        "dynamic_range_db": float(dynamic_range),
        "is_monotone": is_monotone,
        "passed": not is_monotone and dynamic_range > 4,
    }
```

---

## Stage 5: Content Quality (NEW - Script Evaluation)

**What it does**: Checks if the SCRIPT ITSELF is good for a podcast

**In plain English**: The previous stages check if the audio is technically good. This stage asks: "Is the content actually good? Is it engaging? Does it cover what the user asked for?"

### Why This Matters

Bad script → good audio = BAD PODCAST

You could have perfect pronunciation and crystal-clear audio, but if the script is:
- Off-topic from what user requested
- Boring and dry like a Wikipedia article
- Too long/short for target duration
- Missing intro or conclusion
- Full of hallucinated facts

...the podcast still fails.

### What We Check

| Dimension | Question | Why It Matters |
|-----------|----------|----------------|
| **Topic Adherence** | Does it cover what the user asked for? | User asked about "AI in healthcare" but script talks about "AI in gaming" = fail |
| **Structure** | Does it have intro, body, conclusion? | Podcast that just... ends... without wrapping up feels incomplete |
| **Engagement** | Would someone want to listen to this? | Dry recitation of facts vs. storytelling with hooks |
| **Speakability** | Will this sound good when read aloud? | 50-word sentences are hard to speak naturally |
| **Accuracy** | Are the facts correct? | Hallucinated claims damage credibility |
| **Style Match** | Does tone match what user wanted? | User wanted "casual and fun" but got "formal academic" |

### How We Implement It (LLM-as-a-Judge)

Since there's no off-the-shelf tool for this (it's our business logic), we use an LLM to evaluate:

```python
import openai  # or anthropic

EVALUATION_PROMPT = """
You are evaluating a podcast script for quality.

USER'S ORIGINAL REQUEST:
{user_request}

GENERATED SCRIPT:
{script}

Rate each dimension 1-10 and explain briefly:

1. TOPIC ADHERENCE: Does it actually cover what was requested?
2. STRUCTURE: Does it have a proper intro, logical flow, and conclusion?
3. ENGAGEMENT: Would this be interesting to listen to? Uses stories/hooks?
4. SPEAKABILITY: Are sentences short enough to speak naturally? No tongue-twisters?
5. ACCURACY: Do claims seem reasonable and grounded (not made up)?
6. STYLE MATCH: Does the tone match what the user wanted?

For each, provide:
- Score (1-10)
- One-sentence explanation
- Any red flags

Then give an OVERALL SCORE (1-10) and PASS/FAIL recommendation.
"""

def evaluate_script_quality(user_request: str, script: str) -> dict:
    """Use LLM to evaluate if the script content is good."""

    response = openai.chat.completions.create(
        model="gpt-4o-mini",  # Cheap but good enough for evaluation
        messages=[{
            "role": "user",
            "content": EVALUATION_PROMPT.format(
                user_request=user_request,
                script=script
            )
        }],
        temperature=0.3,  # More consistent ratings
    )

    # Parse the response (in practice, use structured output)
    evaluation = parse_evaluation(response.choices[0].message.content)

    return {
        "scores": evaluation["dimension_scores"],
        "overall_score": evaluation["overall"],
        "red_flags": evaluation["red_flags"],
        "passed": evaluation["overall"] >= 7,
    }
```

### Cost for Content Evaluation

| Volume | Cost/Script | Monthly Cost |
|--------|-------------|--------------|
| Using GPT-4o-mini | ~$0.002 (0.2 cents) | $2 for 1000 scripts |
| Using Claude Haiku | ~$0.001 (0.1 cents) | $1 for 1000 scripts |

**This is incredibly cheap** for business-logic validation!

### Bonus: FREE Rule-Based Checks

For obvious issues, we can check without LLM:

```python
def quick_script_checks(script: str, target_duration_min: int) -> list[str]:
    """Fast, free checks for obvious script problems."""
    issues = []

    # Word count check (roughly 150 words/minute for podcasts)
    words = len(script.split())
    expected_words = target_duration_min * 150
    if words < expected_words * 0.7:
        issues.append(f"Script too short: {words} words for {target_duration_min} min target")
    if words > expected_words * 1.3:
        issues.append(f"Script too long: {words} words for {target_duration_min} min target")

    # Sentence length check
    sentences = script.split('.')
    long_sentences = [s for s in sentences if len(s.split()) > 30]
    if long_sentences:
        issues.append(f"{len(long_sentences)} sentences are too long to speak naturally")

    # Has intro?
    intro_words = ['welcome', 'hello', 'today', "let's", 'going to']
    if not any(word in script.lower()[:500] for word in intro_words):
        issues.append("Script may be missing a proper introduction")

    # Has outro?
    outro_words = ['thank', 'hope you', 'next time', 'goodbye', 'wrap up', 'conclusion']
    if not any(word in script.lower()[-500:] for word in outro_words):
        issues.append("Script may be missing a proper conclusion")

    # Repetition check (same phrase repeated = LLM got stuck in loop)
    # ... additional checks ...

    return issues
```

---

## Complete 5-Stage Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE AUDIO GENERATION                       │
├─────────────────────────────────────────────────────────────────┤
│  Stage 5: Content Quality                                        │
│  ├── Quick checks (FREE): length, structure, sentences           │
│  └── LLM evaluation (~$0.002): topic, engagement, accuracy       │
│                                                                   │
│  IF FAILS → Regenerate script before spending money on audio     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    AFTER AUDIO GENERATION                        │
├─────────────────────────────────────────────────────────────────┤
│  Stage 1: Format Check (FREE)                                    │
│  └── Is file valid? Right format? Not corrupted?                 │
│                                                                   │
│  Stage 2: Pronunciation (FREE)                                   │
│  └── Transcribe → Compare to script → Calculate WER              │
│                                                                   │
│  Stage 3: Audio Quality (FREE)                                   │
│  └── Predict MOS score (1-5 scale)                               │
│                                                                   │
│  Stage 4: Prosody (FREE)                                         │
│  └── Check for monotone, unnatural speech patterns               │
│                                                                   │
│  IF ANY FAILS → Regenerate audio with different settings         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                        ✅ PASS → Deliver to user
```

---

## Full Cost Breakdown

### Option A: Entirely Free (Except Content Eval)

| Stage | Tool | Cost |
|-------|------|------|
| Format Validation | ffprobe | $0 |
| Pronunciation | Whisper local + jiwer | $0 |
| Audio Quality | UTMOS | $0 |
| Prosody | librosa | $0 |
| Content (basic) | Rule-based checks | $0 |
| Content (full) | GPT-4o-mini eval | ~$2/month |
| **Total** | | **$0-2/month** |

### What You Need to Run Free

- **For Whisper**: GPU with 4-8GB VRAM (or CPU, just slower)
- **For everything else**: Any computer, no special hardware
- **Time**: Without GPU, a 10-min podcast takes ~5-10 min to process

### If You Want APIs (For Convenience, Not Quality)

| Scenario | Monthly Cost |
|----------|--------------|
| 100 podcasts/month, all free | $0-2 |
| 1000 podcasts/month, Whisper API | $60 |
| 1000 podcasts/month, Whisper API + Content eval | $62 |

---

## Implementation Plan

### Phase 1: Quick Wins (Week 1)

1. **Format validation** - catches corrupted files immediately
2. **Rule-based content checks** - catches obviously bad scripts for free
3. **Basic pass/fail reporting**

*Effort*: 2-3 days
*Cost*: $0

### Phase 2: Core Audio QA (Week 2)

1. **Whisper + WER** for pronunciation checking
2. **UTMOS** for audio quality scoring
3. Store results in Firestore

*Effort*: 3-4 days
*Cost*: $0 (or ~$6/month if using API)

### Phase 3: Full System (Week 3-4)

1. **LLM content evaluation**
2. **Prosody analysis**
3. **Automated regeneration** when QA fails
4. **Dashboard** for monitoring trends

*Effort*: 5-7 days
*Cost*: ~$2/month for content eval

---

## Summary: What You Get

| Capability | What It Catches | Cost |
|------------|-----------------|------|
| Format check | Corrupted files, wrong format | FREE |
| Pronunciation check | "box" instead of "fox", garbled words | FREE |
| Quality score | Robotic, glitchy, or poor-sounding audio | FREE |
| Naturalness check | Monotone, unnatural speech patterns | FREE |
| Content check | Off-topic, boring, wrong structure, made-up facts | ~$0.002/script |

**Total: $0-2/month for production-grade quality assurance**

The free tools are NOT inferior - they're the same tools (literally the same Whisper model) that the paid APIs use. You're only paying for convenience and infrastructure, not quality.

---

## Appendix: Sample QA Report

```json
{
  "job_id": "job_abc123",
  "overall_passed": true,

  "content_qa": {
    "evaluated_before_audio": true,
    "topic_score": 9,
    "structure_score": 8,
    "engagement_score": 7,
    "overall_score": 8,
    "passed": true,
    "notes": "Good coverage of topic, engaging storytelling"
  },

  "format_qa": {
    "valid": true,
    "format": "mp3",
    "duration_seconds": 612,
    "bitrate": 192000
  },

  "pronunciation_qa": {
    "word_error_rate": 0.032,
    "accuracy": "96.8%",
    "passed": true
  },

  "quality_qa": {
    "mos_score": 4.23,
    "quality_level": "Good",
    "passed": true
  },

  "prosody_qa": {
    "pitch_variation_hz": 45.2,
    "dynamic_range_db": 8.5,
    "is_monotone": false,
    "passed": true
  }
}
```

---

## Questions for Discussion

1. **GPU availability**: Do we have GPU infrastructure for running Whisper locally? If not, do we want to add it, or just use Whisper API ($0.006/min)?

2. **Content evaluation thresholds**: What should the minimum scores be for each dimension? I've proposed 7/10 overall, but this can be tuned.

3. **Regeneration strategy**: When QA fails, should we:
   - Auto-regenerate with different TTS settings?
   - Auto-regenerate with different LLM for script?
   - Alert human for manual review?
   - Some combination?

4. **Timing**: Should content QA block audio generation? (I recommend yes - why pay for audio on a bad script?)
