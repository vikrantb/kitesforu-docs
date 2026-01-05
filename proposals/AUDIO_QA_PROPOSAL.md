# Audio & Content Quality Assurance System Proposal

> **Status**: Implementation-Ready Specification
> **Created**: 2026-01-04
> **Updated**: 2026-01-04 (v3 - Added voice matching, multi-language, CLI tool, repo structure)
> **Purpose**: This document serves as the complete specification for building the QA system

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

This proposal outlines a **6-stage quality system** for KitesForU podcasts:

| Stage | What It Checks | Tool | Cost |
|-------|---------------|------|------|
| 1. Format | Is the audio file valid? | ffprobe | FREE |
| 2. Pronunciation | Were words spoken correctly? | Whisper + jiwer | FREE |
| 3. Audio Quality | Does it sound good? | UTMOS | FREE |
| 4. Prosody | Is it natural, not robotic? | librosa | FREE |
| 5. **Content** | Is the SCRIPT itself good? | LLM evaluation | ~$1/month |
| 6. **Voice Matching** | Does voice match persona/topic? | Whisper + LLM | ~$0.50/month |

**Bottom Line**: The entire system can run for **$0-2/month** using open-source tools, with no sacrifice in quality.

---

## What This Document Is

This is an **implementation-ready specification**. It contains:

1. **Technical specifications** with code examples
2. **CLI tool design** for executing QA
3. **Repository structure** for standalone deployment
4. **Multi-language architecture** for future expansion
5. **Integration patterns** with KitesForU API

**An AI agent should be able to implement the complete QA system using only this document.**

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
import google.generativeai as genai

# Recommended: Gemini 2.5 Flash-Lite ($0.10/$0.40 per 1M tokens)
# This is 3x cheaper than GPT-4o-mini with comparable quality for evaluation tasks

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

Respond in JSON format:
{
  "dimensions": {"topic": 8, "structure": 7, ...},
  "explanations": {"topic": "...", ...},
  "red_flags": [],
  "overall_score": 7.5,
  "passed": true
}
"""

def evaluate_script_quality(user_request: str, script: str) -> dict:
    """Use LLM to evaluate if the script content is good."""

    genai.configure(api_key=os.environ["GOOGLE_AI_API_KEY"])
    model = genai.GenerativeModel("gemini-2.0-flash")  # or gemini-2.5-flash-lite

    response = model.generate_content(
        EVALUATION_PROMPT.format(
            user_request=user_request,
            script=script
        ),
        generation_config={"temperature": 0.3}
    )

    evaluation = json.loads(response.text)

    return {
        "scores": evaluation["dimensions"],
        "overall_score": evaluation["overall_score"],
        "red_flags": evaluation["red_flags"],
        "passed": evaluation["passed"],
    }
```

### Model Selection: Why Gemini Flash Over GPT-4o-mini

For evaluation tasks (not generation), cheaper models work just as well:

| Model | Input Cost | Output Cost | Per 1000 Evals | Notes |
|-------|-----------|-------------|----------------|-------|
| **Gemini 2.5 Flash-Lite** | $0.10/1M | $0.40/1M | **~$0.50** | CHEAPEST, recommended |
| Gemini 2.0 Flash | $0.15/1M | $0.60/1M | ~$0.75 | Good alternative |
| GPT-4o-mini | $0.15/1M | $0.60/1M | ~$0.75 | Same price as Gemini 2.0 |
| Claude 3.5 Haiku | $0.80/1M | $4.00/1M | ~$4.50 | Too expensive for QA |

**Recommendation**: Use Gemini 2.5 Flash-Lite for all QA evaluations. It's 3x cheaper than GPT-4o-mini with comparable quality for structured evaluation tasks.

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

## Stage 6: Voice/Host Matching (NEW - Persona Validation)

**What it does**: Checks if the voice matches the persona, name, gender, and topic

**In plain English**: We catch these embarrassing issues:
- Audio says "Host 1" or "Host 2" literally instead of using actual names
- Script says "I am Sara" but the voice is clearly male
- Script says "I am Ram" but the voice sounds Chinese/different ethnicity
- Topic is "fitness tips for seniors" but the voice is a young teen

### The Problems We Catch

| Problem Type | Example | How We Detect It |
|-------------|---------|------------------|
| **Placeholder Names** | Audio literally says "Host 1" | Pattern matching in transcript |
| **Gender Mismatch** | "I am Sara" + deep male voice | Compare claimed gender to detected pitch |
| **Name Mismatch** | "I am Ram" + voice doesn't match name origin | Semantic analysis of name vs voice characteristics |
| **Topic Mismatch** | Seniors fitness + teen voice | Compare topic demographics to voice age estimation |
| **Multi-host confusion** | One voice for both "host 1" and "host 2" | Detect if multiple personas use same voice |

### Detection Strategy

#### Problem 1: Placeholder Names in Audio

```python
import re

PLACEHOLDER_PATTERNS = [
    r'\bhost\s*[0-9]+\b',           # "host 1", "host 2"
    r'\bspeaker\s*[0-9]+\b',        # "speaker 1"
    r'\bvoice\s*[0-9]+\b',          # "voice 1"
    r'\bperson\s*[0-9]+\b',         # "person 1"
    r'\b(host|speaker)\s*(one|two|three)\b',  # "host one"
]

def detect_placeholder_names(transcript: str) -> list[str]:
    """Detect if audio literally says 'host 1' etc."""
    issues = []
    transcript_lower = transcript.lower()

    for pattern in PLACEHOLDER_PATTERNS:
        matches = re.findall(pattern, transcript_lower)
        if matches:
            issues.append(f"Audio contains placeholder: '{matches[0]}'")

    return issues
```

#### Problem 2: Gender Mismatch

```python
import librosa
import numpy as np

def detect_voice_gender(audio_path: str) -> dict:
    """Estimate voice gender based on fundamental frequency (F0)."""
    audio, sr = librosa.load(audio_path)

    # Extract pitch
    pitches, magnitudes = librosa.piptrack(y=audio, sr=sr)

    # Get fundamental frequency (F0)
    f0_values = []
    for t in range(pitches.shape[1]):
        index = magnitudes[:, t].argmax()
        pitch = pitches[index, t]
        if pitch > 0:
            f0_values.append(pitch)

    if not f0_values:
        return {"gender": "unknown", "confidence": 0}

    avg_f0 = np.mean(f0_values)

    # Typical ranges: Male 85-180 Hz, Female 165-255 Hz
    if avg_f0 < 140:
        return {"gender": "male", "avg_f0": avg_f0, "confidence": 0.9}
    elif avg_f0 > 190:
        return {"gender": "female", "avg_f0": avg_f0, "confidence": 0.9}
    else:
        return {"gender": "uncertain", "avg_f0": avg_f0, "confidence": 0.5}


def check_gender_consistency(script: str, transcript: str, voice_gender: dict) -> list[str]:
    """Check if claimed gender matches detected voice gender."""
    issues = []

    # Patterns that indicate claimed gender
    female_claims = [
        r"I am (Sara|Sarah|Emma|Lisa|Maria|Anna|Sofia|Rachel|Jessica|Amanda)",
        r"my name is (Sara|Sarah|Emma|Lisa|Maria|Anna|Sofia|Rachel|Jessica|Amanda)",
        r"this is (Sara|Sarah|Emma|Lisa|Maria|Anna|Sofia|Rachel|Jessica|Amanda) speaking",
    ]
    male_claims = [
        r"I am (John|Mike|David|James|Robert|William|Ram|Raj|Michael|Daniel)",
        r"my name is (John|Mike|David|James|Robert|William|Ram|Raj|Michael|Daniel)",
        r"this is (John|Mike|David|James|Robert|William|Ram|Raj|Michael|Daniel) speaking",
    ]

    combined_text = (script + " " + transcript).lower()

    claimed_female = any(re.search(p, combined_text, re.I) for p in female_claims)
    claimed_male = any(re.search(p, combined_text, re.I) for p in male_claims)

    if claimed_female and voice_gender["gender"] == "male" and voice_gender["confidence"] > 0.7:
        issues.append(f"Gender mismatch: Script claims female name but voice is male (F0={voice_gender['avg_f0']:.0f}Hz)")

    if claimed_male and voice_gender["gender"] == "female" and voice_gender["confidence"] > 0.7:
        issues.append(f"Gender mismatch: Script claims male name but voice is female (F0={voice_gender['avg_f0']:.0f}Hz)")

    return issues
```

#### Problem 3: Topic-Voice Mismatch

```python
TOPIC_VOICE_EXPECTATIONS = {
    # topic_keywords: expected_voice_characteristics
    "senior": {"age_range": "mature", "min_pitch_variance": 20},
    "elderly": {"age_range": "mature", "min_pitch_variance": 20},
    "retirement": {"age_range": "mature", "min_pitch_variance": 20},
    "children": {"age_range": "young_adult_or_mature"},  # Not teen
    "kids": {"age_range": "young_adult_or_mature"},
    "parenting": {"age_range": "mature"},
    "teen": {"age_range": "young"},
    "college": {"age_range": "young_adult"},
}

def estimate_voice_age(audio_path: str) -> str:
    """Rough voice age estimation based on pitch characteristics."""
    audio, sr = librosa.load(audio_path)

    # Extract pitch
    f0, voiced_flag, voiced_probs = librosa.pyin(
        audio, fmin=librosa.note_to_hz('C2'),
        fmax=librosa.note_to_hz('C7'), sr=sr
    )

    valid_f0 = f0[~np.isnan(f0)]
    if len(valid_f0) == 0:
        return "unknown"

    avg_f0 = np.mean(valid_f0)
    f0_std = np.std(valid_f0)

    # Very rough heuristics (would need ML model for accuracy)
    # Young voices tend to be higher and more variable
    # Mature voices tend to be lower and more stable

    if avg_f0 > 220:  # Higher pitched
        return "young" if f0_std > 40 else "young_adult"
    elif avg_f0 > 150:
        return "young_adult" if f0_std > 35 else "mature"
    else:
        return "mature"


def check_topic_voice_match(topic: str, voice_age: str) -> list[str]:
    """Check if voice characteristics match topic expectations."""
    issues = []
    topic_lower = topic.lower()

    for keyword, expectations in TOPIC_VOICE_EXPECTATIONS.items():
        if keyword in topic_lower:
            expected_age = expectations.get("age_range", "any")

            if expected_age == "mature" and voice_age == "young":
                issues.append(
                    f"Voice mismatch: Topic '{keyword}' suggests mature voice, "
                    f"but detected young-sounding voice"
                )
            if expected_age == "young" and voice_age == "mature":
                issues.append(
                    f"Voice mismatch: Topic '{keyword}' suggests young voice, "
                    f"but detected mature-sounding voice"
                )

    return issues
```

#### Problem 4: Multi-Host Same Voice Detection

```python
def detect_multiple_speakers(audio_path: str) -> dict:
    """Detect if audio has multiple distinct speakers."""
    # Use pyannote-audio for speaker diarization
    # This is more complex but worth it for multi-host detection

    from pyannote.audio import Pipeline

    pipeline = Pipeline.from_pretrained(
        "pyannote/speaker-diarization-3.1",
        use_auth_token=os.environ.get("HUGGINGFACE_TOKEN")
    )

    diarization = pipeline(audio_path)

    # Count unique speakers
    speakers = set()
    for turn, _, speaker in diarization.itertracks(yield_label=True):
        speakers.add(speaker)

    return {
        "num_speakers_detected": len(speakers),
        "speaker_segments": [
            {"speaker": speaker, "start": turn.start, "end": turn.end}
            for turn, _, speaker in diarization.itertracks(yield_label=True)
        ]
    }


def check_multi_host_consistency(script: str, diarization: dict) -> list[str]:
    """Check if multi-host script has multiple actual voices."""
    issues = []

    # Detect if script expects multiple hosts
    host_patterns = [
        r'\[host\s*[0-9]+\]',
        r'\[(speaker|voice)\s*[0-9]+\]',
        r'host\s*[0-9]+:',
    ]

    script_lower = script.lower()
    expected_hosts = 0
    for pattern in host_patterns:
        matches = re.findall(pattern, script_lower)
        expected_hosts = max(expected_hosts, len(set(matches)))

    detected_speakers = diarization["num_speakers_detected"]

    if expected_hosts > 1 and detected_speakers == 1:
        issues.append(
            f"Multi-host mismatch: Script expects {expected_hosts} hosts, "
            f"but only 1 voice detected in audio"
        )

    return issues
```

### Complete Voice Matching Pipeline

```python
def check_voice_matching(
    audio_path: str,
    script: str,
    topic: str,
    expected_hosts: int = 1
) -> dict:
    """Complete voice/host matching validation."""

    issues = []

    # Step 1: Transcribe audio
    model = whisper.load_model("base")  # Faster than large for this check
    result = model.transcribe(audio_path)
    transcript = result["text"]

    # Step 2: Check for placeholder names
    placeholder_issues = detect_placeholder_names(transcript)
    issues.extend(placeholder_issues)

    # Step 3: Check gender consistency
    voice_gender = detect_voice_gender(audio_path)
    gender_issues = check_gender_consistency(script, transcript, voice_gender)
    issues.extend(gender_issues)

    # Step 4: Check topic-voice match
    voice_age = estimate_voice_age(audio_path)
    topic_issues = check_topic_voice_match(topic, voice_age)
    issues.extend(topic_issues)

    # Step 5: Check multi-host if expected
    if expected_hosts > 1:
        diarization = detect_multiple_speakers(audio_path)
        host_issues = check_multi_host_consistency(script, diarization)
        issues.extend(host_issues)
    else:
        diarization = None

    return {
        "transcript_sample": transcript[:500],
        "voice_gender": voice_gender,
        "voice_age": voice_age,
        "diarization": diarization,
        "issues": issues,
        "passed": len(issues) == 0,
    }
```

### Voice Matching Cost

| Component | Tool | Cost |
|-----------|------|------|
| Transcription | Whisper base (local) | FREE |
| Pitch analysis | librosa | FREE |
| Speaker diarization | pyannote (optional) | FREE (requires HuggingFace token) |
| Semantic validation | Gemini Flash | ~$0.0005/check |

**Total: ~$0.50/month for 1000 podcasts** (only the semantic check costs anything)

---

## Complete 6-Stage Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE AUDIO GENERATION                       │
├─────────────────────────────────────────────────────────────────┤
│  Stage 5: Content Quality                                        │
│  ├── Quick checks (FREE): length, structure, sentences           │
│  └── LLM evaluation (~$0.001): topic, engagement, accuracy       │
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
│  Stage 6: Voice Matching (FREE + ~$0.0005)                       │
│  └── Check placeholder names, gender, topic-voice match          │
│                                                                   │
│  IF ANY FAILS → Regenerate audio with different settings         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                        ✅ PASS → Deliver to user
```

---

## Full Cost Breakdown

### Option A: Entirely Free (Except LLM Evals)

| Stage | Tool | Cost |
|-------|------|------|
| Format Validation | ffprobe | $0 |
| Pronunciation | Whisper local + jiwer | $0 |
| Audio Quality | UTMOS | $0 |
| Prosody | librosa | $0 |
| Voice Matching (basic) | Whisper + librosa | $0 |
| Voice Matching (semantic) | Gemini Flash-Lite | ~$0.50/month |
| Content (basic) | Rule-based checks | $0 |
| Content (full) | Gemini Flash-Lite | ~$0.50/month |
| **Total** | | **$0-1/month** |

### What You Need to Run Free

- **For Whisper**: GPU with 4-8GB VRAM (or CPU, just slower)
- **For pyannote diarization**: GPU recommended, but works on CPU
- **For everything else**: Any computer, no special hardware
- **Time**: Without GPU, a 10-min podcast takes ~5-10 min to process

### If You Want APIs (For Convenience, Not Quality)

| Scenario | Monthly Cost |
|----------|--------------|
| 100 podcasts/month, all free except LLM | $0-1 |
| 1000 podcasts/month, Whisper API | $60 |
| 1000 podcasts/month, Whisper API + all LLM evals | $61 |

---

## Implementation Plan

### Phase 1: Core QA Pipeline

1. **Repository setup** - Create `kitesforu-qa` repo with structure
2. **Format validation** - Stage 1 implementation
3. **Pronunciation check** - Whisper + WER (Stage 2)
4. **CLI framework** - Basic `kqa run` command

**Deliverable**: Can run `kqa run --audio gs://... --request "..."` and get format + pronunciation results

### Phase 2: Audio Quality & Prosody

1. **MOS scoring** - UTMOS integration (Stage 3)
2. **Prosody analysis** - librosa naturalness check (Stage 4)
3. **GCS integration** - Download/upload from Cloud Storage

**Deliverable**: Full audio technical QA working

### Phase 3: Content & Voice Matching

1. **Content evaluation** - LLM-as-judge with Gemini (Stage 5)
2. **Voice matching** - Placeholder detection, gender, topic matching (Stage 6)
3. **Speaker diarization** - pyannote integration for multi-host

**Deliverable**: All 6 stages working

### Phase 4: E2E & Polish

1. **E2E testing** - `kqa e2e` command with API integration
2. **Batch processing** - `kqa batch` for multiple podcasts
3. **Docker support** - Containerized deployment
4. **Multi-language** - Language configuration system

**Deliverable**: Production-ready CLI tool with Docker support

---

## Summary: What You Get

| Capability | What It Catches | Cost |
|------------|-----------------|------|
| Format check | Corrupted files, wrong format | FREE |
| Pronunciation check | "box" instead of "fox", garbled words | FREE |
| Quality score | Robotic, glitchy, or poor-sounding audio | FREE |
| Naturalness check | Monotone, unnatural speech patterns | FREE |
| Content check | Off-topic, boring, wrong structure, made-up facts | ~$0.001/script |
| Voice matching | "Host 1" spoken aloud, wrong gender, wrong voice for topic | ~$0.0005/script |

**Total: $0-1.50/month for production-grade quality assurance**

The free tools are NOT inferior - they're the same tools (literally the same Whisper model) that the paid APIs use. You're only paying for convenience and infrastructure, not quality.

### Key Capabilities Added in v3

| Feature | Description |
|---------|-------------|
| **Voice/Host Matching** | Catches placeholder names, gender mismatches, topic-voice mismatches |
| **Multi-Language Ready** | Architecture supports 100+ languages via Whisper |
| **CLI Tool** | `kqa run`, `kqa e2e`, `kqa batch` commands |
| **Standalone Repo** | `kitesforu-qa` - separate from main codebase |
| **GCS Integration** | Works directly with GCS audio paths |
| **Docker Support** | Containerized for easy deployment |
| **E2E Testing** | Create podcast via API → run QA → report |

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

## Multi-Language Architecture

The QA system is designed to support multiple languages from the start.

### Language-Aware Components

| Component | Language Handling | Notes |
|-----------|------------------|-------|
| **Whisper STT** | 100+ languages supported | Auto-detects language, or specify explicitly |
| **jiwer WER** | Unicode-aware | Works with any language script |
| **librosa** | Language-agnostic | Pitch/prosody analysis works on any audio |
| **UTMOS** | Primarily English-trained | May need language-specific models for MOS |
| **LLM Evaluation** | Any language | Gemini/GPT support 100+ languages |

### Language Configuration

```python
from dataclasses import dataclass
from enum import Enum

class Language(Enum):
    ENGLISH = "en"
    SPANISH = "es"
    FRENCH = "fr"
    GERMAN = "de"
    HINDI = "hi"
    JAPANESE = "ja"
    CHINESE = "zh"
    KOREAN = "ko"
    PORTUGUESE = "pt"
    ARABIC = "ar"
    # Add more as needed

@dataclass
class LanguageConfig:
    """Language-specific QA configuration."""
    language: Language
    whisper_model: str = "large-v3"  # Best for multilingual
    whisper_task: str = "transcribe"  # or "translate" for English output
    name_patterns: dict = None  # Language-specific name patterns
    prosody_thresholds: dict = None  # Language-specific naturalness thresholds

    def __post_init__(self):
        # Default name patterns by language
        if self.name_patterns is None:
            self.name_patterns = LANGUAGE_NAME_PATTERNS.get(
                self.language.value,
                {}
            )

# Language-specific name patterns for gender detection
LANGUAGE_NAME_PATTERNS = {
    "en": {
        "female": ["Sara", "Emma", "Lisa", "Maria", "Rachel"],
        "male": ["John", "Mike", "David", "James", "Robert"],
    },
    "hi": {
        "female": ["Priya", "Kavita", "Sunita", "Neha", "Pooja"],
        "male": ["Ram", "Raj", "Vikram", "Arjun", "Amit"],
    },
    "es": {
        "female": ["Maria", "Carmen", "Ana", "Sofia", "Isabel"],
        "male": ["Jose", "Carlos", "Miguel", "Juan", "Antonio"],
    },
    # Add more languages as needed
}

# Language-specific prosody expectations
LANGUAGE_PROSODY_THRESHOLDS = {
    "en": {"pitch_std_min": 20, "pitch_std_max": 80},
    "ja": {"pitch_std_min": 30, "pitch_std_max": 100},  # Japanese has more pitch variation
    "zh": {"pitch_std_min": 40, "pitch_std_max": 120},  # Tonal language
    # Add more languages as needed
}
```

### Multi-Language QA Pipeline

```python
def run_qa_pipeline(
    audio_path: str,
    script: str,
    user_request: str,
    language: Language = Language.ENGLISH,
    topic: str = "",
) -> dict:
    """Run full QA pipeline with language awareness."""

    config = LanguageConfig(language=language)

    results = {
        "language": language.value,
        "stages": {},
    }

    # Stage 1: Format (language-agnostic)
    results["stages"]["format"] = check_audio_format(audio_path)

    # Stage 2: Pronunciation (language-aware)
    results["stages"]["pronunciation"] = check_pronunciation(
        audio_path=audio_path,
        original_script=script,
        language=language.value,  # Tell Whisper the language
    )

    # Stage 3: Audio Quality (may need language-specific models later)
    results["stages"]["quality"] = check_audio_quality(audio_path)

    # Stage 4: Prosody (use language-specific thresholds)
    results["stages"]["prosody"] = check_naturalness(
        audio_path=audio_path,
        thresholds=LANGUAGE_PROSODY_THRESHOLDS.get(
            language.value,
            LANGUAGE_PROSODY_THRESHOLDS["en"]  # Default to English
        )
    )

    # Stage 5: Content (LLM handles any language)
    results["stages"]["content"] = evaluate_script_quality(
        user_request=user_request,
        script=script,
        language=language.value,
    )

    # Stage 6: Voice Matching (use language-specific name patterns)
    results["stages"]["voice_matching"] = check_voice_matching(
        audio_path=audio_path,
        script=script,
        topic=topic,
        name_patterns=config.name_patterns,
    )

    # Overall pass/fail
    results["passed"] = all(
        stage.get("passed", True)
        for stage in results["stages"].values()
    )

    return results
```

### Language Support Roadmap

| Phase | Languages | Notes |
|-------|-----------|-------|
| **Phase 1 (Now)** | English | Full support, all features |
| **Phase 2** | Spanish, French, German | European languages, high demand |
| **Phase 3** | Hindi, Portuguese | Large user bases |
| **Phase 4** | Japanese, Chinese, Korean | Asian languages, may need special prosody handling |
| **Phase 5** | Arabic, Others | RTL languages, additional complexity |

### What's NOT Language-Specific

These components work identically for all languages:
- Format validation (file integrity)
- Audio quality MOS prediction (sound quality is universal)
- Basic prosody (pitch/energy variation)
- File handling and GCS integration

---

## CLI Tool Design & Repository Structure

The QA system will be a **standalone CLI tool** in its own GitHub repository.

### Repository: `kitesforu-qa`

```
kitesforu-qa/
├── README.md                    # Usage documentation
├── pyproject.toml               # Python package config
├── requirements.txt             # Dependencies
├── setup.py                     # Installation
│
├── src/
│   └── kitesforu_qa/
│       ├── __init__.py
│       ├── cli.py               # CLI entry point
│       ├── config.py            # Configuration management
│       ├── pipeline.py          # Main QA orchestration
│       │
│       ├── stages/              # Individual QA stages
│       │   ├── __init__.py
│       │   ├── format.py        # Stage 1: Format validation
│       │   ├── pronunciation.py # Stage 2: WER check
│       │   ├── quality.py       # Stage 3: MOS scoring
│       │   ├── prosody.py       # Stage 4: Naturalness
│       │   ├── content.py       # Stage 5: Script evaluation
│       │   └── voice_matching.py# Stage 6: Voice/host validation
│       │
│       ├── integrations/        # External service integrations
│       │   ├── __init__.py
│       │   ├── gcs.py           # Google Cloud Storage
│       │   ├── kitesforu_api.py # KitesForU API client
│       │   ├── whisper.py       # Whisper STT wrapper
│       │   └── llm.py           # LLM evaluation (Gemini)
│       │
│       ├── models/              # Data models
│       │   ├── __init__.py
│       │   ├── job.py           # Job data model
│       │   ├── results.py       # QA results model
│       │   └── language.py      # Language configuration
│       │
│       └── utils/               # Utilities
│           ├── __init__.py
│           ├── audio.py         # Audio file utilities
│           └── reporting.py     # Report generation
│
├── tests/                       # Test suite
│   ├── conftest.py
│   ├── test_format.py
│   ├── test_pronunciation.py
│   ├── test_quality.py
│   ├── test_prosody.py
│   ├── test_content.py
│   ├── test_voice_matching.py
│   └── fixtures/                # Test audio files
│       └── sample_podcast.mp3
│
├── scripts/                     # Utility scripts
│   ├── run_qa.sh               # Quick run script
│   └── download_models.sh       # Download ML models
│
└── docker/                      # Docker support
    ├── Dockerfile
    └── docker-compose.yml
```

### CLI Interface

```bash
# Install
pip install kitesforu-qa

# Or from source
git clone https://github.com/vikrantb/kitesforu-qa
cd kitesforu-qa
pip install -e .
```

#### Command: `kqa run` - Run QA on Single Podcast

```bash
# Basic usage with GCS path
kqa run \
    --audio "gs://kitesforu-podcasts/job_123/final.mp3" \
    --request "Create a 10-minute podcast about AI in healthcare" \
    --job-id "job_123"

# With script file
kqa run \
    --audio "gs://kitesforu-podcasts/job_123/final.mp3" \
    --script "/path/to/script.txt" \
    --request "Create a 10-minute podcast about AI in healthcare"

# With language specification
kqa run \
    --audio "gs://kitesforu-podcasts/job_123/final.mp3" \
    --request "Create a podcast about technology in Hindi" \
    --language hi

# Verbose output
kqa run \
    --audio "gs://kitesforu-podcasts/job_123/final.mp3" \
    --request "..." \
    --verbose

# Output to file
kqa run \
    --audio "gs://kitesforu-podcasts/job_123/final.mp3" \
    --request "..." \
    --output report.json
```

#### Command: `kqa e2e` - End-to-End Test

```bash
# Create podcast via API and then run QA
kqa e2e \
    --topic "The future of renewable energy" \
    --duration 10 \
    --style "casual" \
    --wait-timeout 600

# With API credentials
kqa e2e \
    --topic "..." \
    --api-url "https://api.kitesforu.com" \
    --api-key "$KITESFORU_API_KEY"

# Run multiple tests from file
kqa e2e \
    --test-file tests/e2e_scenarios.yaml
```

#### Command: `kqa batch` - Batch Processing

```bash
# Process multiple podcasts
kqa batch \
    --input jobs.csv \
    --output results/ \
    --parallel 4

# Input CSV format:
# job_id,audio_path,request,language
# job_123,gs://...,Create a podcast about...,en
# job_456,gs://...,Make a show about...,es
```

### CLI Implementation

```python
# src/kitesforu_qa/cli.py
import click
import json
from pathlib import Path

from .pipeline import QAPipeline
from .integrations.kitesforu_api import KitesForUClient
from .integrations.gcs import download_from_gcs
from .models.language import Language

@click.group()
@click.version_option()
def cli():
    """KitesForU Audio Quality Assurance CLI"""
    pass


@cli.command()
@click.option('--audio', required=True, help='Audio file path or GCS URI')
@click.option('--request', required=True, help='Original user request')
@click.option('--script', default=None, help='Path to script file (optional)')
@click.option('--job-id', default=None, help='Job ID for tracking')
@click.option('--language', default='en', help='Language code (en, es, hi, etc.)')
@click.option('--topic', default='', help='Podcast topic for voice matching')
@click.option('--verbose', is_flag=True, help='Verbose output')
@click.option('--output', default=None, help='Output file path (JSON)')
def run(audio, request, script, job_id, language, topic, verbose, output):
    """Run QA pipeline on a single podcast."""

    click.echo(f"🔍 Running QA on: {audio}")

    # Download from GCS if needed
    if audio.startswith('gs://'):
        click.echo("  📥 Downloading from GCS...")
        local_audio = download_from_gcs(audio)
    else:
        local_audio = audio

    # Load script if provided
    script_text = None
    if script:
        script_text = Path(script).read_text()

    # Run pipeline
    pipeline = QAPipeline(
        language=Language(language),
        verbose=verbose,
    )

    results = pipeline.run(
        audio_path=local_audio,
        user_request=request,
        script=script_text,
        topic=topic,
        job_id=job_id,
    )

    # Display results
    _display_results(results, verbose)

    # Save to file if requested
    if output:
        Path(output).write_text(json.dumps(results, indent=2))
        click.echo(f"\n📄 Results saved to: {output}")

    # Exit with appropriate code
    if results['passed']:
        click.echo("\n✅ QA PASSED")
        raise SystemExit(0)
    else:
        click.echo("\n❌ QA FAILED")
        raise SystemExit(1)


@cli.command()
@click.option('--topic', required=True, help='Podcast topic')
@click.option('--duration', default=10, help='Duration in minutes')
@click.option('--style', default='conversational', help='Podcast style')
@click.option('--api-url', default='https://api.kitesforu.com', help='API URL')
@click.option('--api-key', envvar='KITESFORU_API_KEY', help='API key')
@click.option('--wait-timeout', default=600, help='Max wait time in seconds')
@click.option('--output', default=None, help='Output file path')
def e2e(topic, duration, style, api_url, api_key, wait_timeout, output):
    """Run end-to-end test: create podcast via API then QA."""

    click.echo(f"🚀 Starting E2E test for: {topic}")

    # Create podcast via API
    client = KitesForUClient(api_url, api_key)

    click.echo("  📝 Creating podcast request...")
    job = client.create_job(
        topic=topic,
        duration_min=duration,
        style=style,
    )
    click.echo(f"  ✓ Job created: {job['job_id']}")

    # Wait for completion
    click.echo(f"  ⏳ Waiting for completion (max {wait_timeout}s)...")
    completed_job = client.wait_for_completion(
        job_id=job['job_id'],
        timeout=wait_timeout,
    )

    if completed_job['status'] != 'completed':
        click.echo(f"  ❌ Job failed: {completed_job.get('error', 'Unknown error')}")
        raise SystemExit(1)

    click.echo(f"  ✓ Job completed!")

    # Get audio and script
    audio_url = completed_job['audio_url']
    script = completed_job.get('script', '')

    # Run QA
    click.echo("  🔍 Running QA pipeline...")
    pipeline = QAPipeline()
    results = pipeline.run(
        audio_path=audio_url,
        user_request=topic,
        script=script,
        job_id=job['job_id'],
    )

    # Display and save results
    _display_results(results, verbose=True)

    if output:
        Path(output).write_text(json.dumps(results, indent=2))

    if results['passed']:
        click.echo("\n✅ E2E TEST PASSED")
        raise SystemExit(0)
    else:
        click.echo("\n❌ E2E TEST FAILED")
        raise SystemExit(1)


def _display_results(results: dict, verbose: bool = False):
    """Display QA results in a nice format."""

    click.echo("\n" + "="*60)
    click.echo("📊 QA RESULTS")
    click.echo("="*60)

    for stage_name, stage_results in results.get('stages', {}).items():
        passed = stage_results.get('passed', True)
        icon = "✅" if passed else "❌"
        click.echo(f"\n{icon} Stage: {stage_name.upper()}")

        if verbose or not passed:
            for key, value in stage_results.items():
                if key != 'passed':
                    click.echo(f"   {key}: {value}")

    click.echo("\n" + "="*60)


if __name__ == '__main__':
    cli()
```

### API Client for End-to-End Tests

```python
# src/kitesforu_qa/integrations/kitesforu_api.py
import time
import requests
from typing import Optional

class KitesForUClient:
    """Client for KitesForU API."""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip('/')
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        }

    def create_job(
        self,
        topic: str,
        duration_min: int = 10,
        style: str = "conversational",
    ) -> dict:
        """Create a new podcast job."""
        response = requests.post(
            f"{self.base_url}/api/v1/jobs",
            headers=self.headers,
            json={
                "topic": topic,
                "duration_min": duration_min,
                "style": style,
            }
        )
        response.raise_for_status()
        return response.json()

    def get_job(self, job_id: str) -> dict:
        """Get job status and details."""
        response = requests.get(
            f"{self.base_url}/api/v1/jobs/{job_id}",
            headers=self.headers,
        )
        response.raise_for_status()
        return response.json()

    def wait_for_completion(
        self,
        job_id: str,
        timeout: int = 600,
        poll_interval: int = 10,
    ) -> dict:
        """Wait for job to complete."""
        start = time.time()

        while time.time() - start < timeout:
            job = self.get_job(job_id)

            if job['status'] in ('completed', 'failed'):
                return job

            time.sleep(poll_interval)

        raise TimeoutError(f"Job {job_id} did not complete within {timeout}s")
```

### Configuration Management

```python
# src/kitesforu_qa/config.py
import os
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class QAConfig:
    """Configuration for QA pipeline."""

    # API Keys
    google_ai_api_key: str = field(
        default_factory=lambda: os.environ.get("GOOGLE_AI_API_KEY", "")
    )
    huggingface_token: str = field(
        default_factory=lambda: os.environ.get("HUGGINGFACE_TOKEN", "")
    )
    kitesforu_api_key: str = field(
        default_factory=lambda: os.environ.get("KITESFORU_API_KEY", "")
    )

    # GCS Configuration
    gcs_project: str = field(
        default_factory=lambda: os.environ.get("GCP_PROJECT", "kitesforu-dev")
    )

    # Model Settings
    whisper_model: str = "large-v3"
    llm_model: str = "gemini-2.0-flash"

    # Thresholds
    wer_threshold: float = 0.10
    mos_threshold: float = 3.5
    content_score_threshold: float = 7.0
    pitch_std_min: float = 20.0

    # Execution
    enable_diarization: bool = True
    enable_content_eval: bool = True
    verbose: bool = False

    @classmethod
    def from_env(cls) -> "QAConfig":
        """Load configuration from environment variables."""
        return cls()

    @classmethod
    def from_file(cls, path: str) -> "QAConfig":
        """Load configuration from YAML file."""
        import yaml
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)
```

### Sample Output Report

```json
{
  "job_id": "job_abc123",
  "timestamp": "2026-01-04T15:30:00Z",
  "language": "en",
  "overall_passed": true,
  "execution_time_seconds": 45.2,

  "stages": {
    "format": {
      "valid": true,
      "format": "mp3",
      "duration_seconds": 612,
      "bitrate": 192000,
      "passed": true
    },
    "pronunciation": {
      "word_error_rate": 0.032,
      "accuracy_percent": 96.8,
      "transcript_sample": "Welcome to today's episode...",
      "passed": true
    },
    "quality": {
      "mos_score": 4.23,
      "quality_level": "Good",
      "passed": true
    },
    "prosody": {
      "pitch_variation_hz": 45.2,
      "dynamic_range_db": 8.5,
      "is_monotone": false,
      "passed": true
    },
    "content": {
      "scores": {
        "topic": 9,
        "structure": 8,
        "engagement": 7,
        "speakability": 8,
        "accuracy": 8,
        "style": 7
      },
      "overall_score": 7.8,
      "red_flags": [],
      "passed": true
    },
    "voice_matching": {
      "placeholder_detected": false,
      "gender_match": true,
      "topic_voice_match": true,
      "multi_host_valid": true,
      "issues": [],
      "passed": true
    }
  },

  "summary": {
    "total_stages": 6,
    "passed_stages": 6,
    "failed_stages": 0,
    "issues": []
  }
}
```

### Docker Support

```dockerfile
# docker/Dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    ffmpeg \
    libsndfile1 \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Download Whisper model (optional - can be done at runtime)
RUN python -c "import whisper; whisper.load_model('base')"

# Copy application
COPY src/ /app/src/
WORKDIR /app

# Set entrypoint
ENTRYPOINT ["python", "-m", "kitesforu_qa.cli"]
```

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  kqa:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    environment:
      - GOOGLE_AI_API_KEY=${GOOGLE_AI_API_KEY}
      - HUGGINGFACE_TOKEN=${HUGGINGFACE_TOKEN}
      - KITESFORU_API_KEY=${KITESFORU_API_KEY}
      - GCP_PROJECT=${GCP_PROJECT:-kitesforu-dev}
    volumes:
      - ~/.config/gcloud:/root/.config/gcloud  # For GCS access
      - ./output:/app/output
```

### Quick Start Script

```bash
#!/bin/bash
# scripts/run_qa.sh

# Load environment
source .env 2>/dev/null || true

# Default values
AUDIO="${1:-}"
REQUEST="${2:-}"

if [ -z "$AUDIO" ] || [ -z "$REQUEST" ]; then
    echo "Usage: ./run_qa.sh <audio_path_or_gcs_uri> <request>"
    echo ""
    echo "Examples:"
    echo "  ./run_qa.sh gs://bucket/audio.mp3 'Create a podcast about AI'"
    echo "  ./run_qa.sh /local/audio.mp3 'Create a podcast about AI'"
    exit 1
fi

python -m kitesforu_qa.cli run \
    --audio "$AUDIO" \
    --request "$REQUEST" \
    --verbose
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

5. **Voice matching sensitivity**: How strict should we be on voice-topic matching? (seniors topic with young voice)

6. **Multi-language priority**: Which languages should we support first after English?
