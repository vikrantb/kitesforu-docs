# KitesForU Audio Quality Assurance (kitesforu-qa)

<!-- METADATA
file: human/services/qa/OVERVIEW.md
category: service-overview
related: [llm/services/qa.yaml]
updated: 2026-01-04
-->

## Summary

**kitesforu-qa** is a 6-stage quality assurance system for validating podcast audio and content. It checks everything from file format validity to voice-topic matching, ensuring generated podcasts meet quality standards before delivery.

## Quick Reference

| Stage | What It Checks | Tool | Cost |
|-------|---------------|------|------|
| 1. Format | File validity, codec, duration | ffprobe | FREE |
| 2. Pronunciation | Word accuracy (WER) | Whisper + jiwer | FREE |
| 3. Audio Quality | Sound quality (MOS 1-5) | UTMOS/librosa | FREE |
| 4. Prosody | Naturalness, not robotic | librosa | FREE |
| 5. Content | Script quality | Gemini LLM | ~$0.001 |
| 6. Voice Matching | Persona/gender match | librosa + pyannote | FREE |

**Total Cost**: $0-1.50/month for production-grade QA

## Installation

```bash
# From PyPI
pip install kitesforu-qa

# From source
git clone https://github.com/vikrantb/kitesforu-qa
cd kitesforu-qa
pip install -e .
```

### System Dependencies

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
apt-get install ffmpeg libsndfile1
```

## CLI Commands

### `kqa run` - Full QA Pipeline

```bash
kqa run \
    --audio "gs://bucket/audio.mp3" \
    --request "Create a 10-minute podcast about AI" \
    --script /path/to/script.txt \
    --language en \
    --verbose
```

### `kqa check-script` - Pre-flight Script Check

```bash
# Quick (free) checks only - no API key needed
kqa check-script --script script.txt --request "..." --duration 10 --quick

# Full LLM evaluation - requires GOOGLE_AI_API_KEY
kqa check-script --script script.txt --request "..." --duration 10
```

### `kqa e2e` - End-to-End Test

```bash
kqa e2e \
    --topic "The future of renewable energy" \
    --duration 10 \
    --api-url "https://api.kitesforu.com"
```

### `kqa batch` - Batch Processing

```bash
kqa batch \
    --input jobs.csv \
    --output results/ \
    --parallel 4
```

## The 6 QA Stages Explained

### Stage 1: Format Validation
Validates basic audio file properties using ffprobe:
- File exists and is readable
- Valid audio codec (mp3, wav, m4a, ogg, flac)
- Duration within expected range (±10% tolerance)
- Sample rate ≥ 16kHz
- Bitrate ≥ 64kbps

### Stage 2: Pronunciation Check
Compares transcribed audio against the original script:
- Transcribes audio using OpenAI Whisper
- Calculates Word Error Rate (WER) using jiwer
- **Pass threshold**: WER ≤ 10%

### Stage 3: Audio Quality (MOS)
Assesses perceptual audio quality:
- Uses UTMOS neural network model (or librosa fallback)
- Returns Mean Opinion Score (1-5 scale)
- **Pass threshold**: MOS ≥ 3.5

| Score | Quality Level |
|-------|---------------|
| 4.5-5.0 | Excellent |
| 3.5-4.5 | Good |
| 2.5-3.5 | Okay |
| 1.5-2.5 | Poor |
| 1.0-1.5 | Bad |

### Stage 4: Prosody Analysis
Detects robotic or unnatural speech:
- Analyzes pitch variation using librosa pyin
- Checks dynamic range
- **Pass threshold**: Pitch std ≥ 20 Hz (English)

Different languages have different thresholds (tonal languages like Chinese have higher natural pitch variation).

### Stage 5: Content Evaluation
Evaluates script quality using LLM-as-judge (Gemini Flash):

**Free Quick Checks** (no API key needed):
- Word count vs target duration
- Intro/outro detection
- Long sentence detection (>35 words)
- Repetition detection (LLM loop artifacts)

**LLM Evaluation** (requires GOOGLE_AI_API_KEY):
- Topic adherence (1-10)
- Structure quality (1-10)
- Engagement factor (1-10)
- Speakability (1-10)
- Accuracy/grounding (1-10)
- Style match (1-10)

**Pass threshold**: Overall score ≥ 7.0

### Stage 6: Voice Matching
Validates voice characteristics against expectations:

| Check | What It Detects |
|-------|-----------------|
| Placeholder names | "Host 1", "Speaker 2" in script |
| Gender detection | F0 pitch analysis (Male: 85-180Hz, Female: 165-255Hz) |
| Gender-name match | Sarah should have female voice |
| Voice age | Young vs mature voice estimation |
| Topic match | Seniors topic shouldn't have young voice |
| Speaker count | Expected number of hosts |

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `GOOGLE_AI_API_KEY` | Gemini API key for content eval | For Stage 5 |
| `HUGGINGFACE_TOKEN` | HuggingFace token for diarization | Optional |
| `KITESFORU_API_KEY` | API key for e2e tests | For e2e |
| `GCP_PROJECT` | GCP project for GCS access | For GCS |

## Sample Output

```json
{
  "job_id": "job_abc123",
  "overall_passed": true,
  "stages": {
    "format": {
      "valid": true,
      "format": "mp3",
      "duration_seconds": 612,
      "passed": true
    },
    "pronunciation": {
      "word_error_rate": 0.032,
      "accuracy_percent": 96.8,
      "passed": true
    },
    "quality": {
      "mos_score": 4.23,
      "quality_level": "Good",
      "passed": true
    },
    "prosody": {
      "pitch_variation_hz": 45.2,
      "is_monotone": false,
      "passed": true
    },
    "content": {
      "overall_score": 7.8,
      "passed": true
    },
    "voice_matching": {
      "placeholder_detected": false,
      "gender_match": true,
      "passed": true
    }
  }
}
```

## Language Support

| Code | Language | Prosody Threshold |
|------|----------|-------------------|
| en | English | 20 Hz |
| es | Spanish | 25 Hz |
| fr | French | 22 Hz |
| de | German | 18 Hz |
| hi | Hindi | 28 Hz |
| ja | Japanese | 30 Hz |
| zh | Chinese | 40 Hz (tonal) |

## Integration with Pipeline

The QA system can be integrated into the KitesForU pipeline:

1. **Post-generation hook**: Run QA after audio is generated
2. **Pre-flight check**: Validate scripts before TTS
3. **Batch validation**: QA all jobs in a CSV
4. **E2E testing**: Full API → QA flow

## Architecture

```
kitesforu-qa/
├── src/kitesforu_qa/
│   ├── cli.py              # CLI entry point
│   ├── config.py           # Configuration
│   ├── pipeline.py         # Orchestration
│   ├── stages/             # QA stages
│   │   ├── format.py       # Stage 1
│   │   ├── pronunciation.py# Stage 2
│   │   ├── quality.py      # Stage 3
│   │   ├── prosody.py      # Stage 4
│   │   ├── content.py      # Stage 5
│   │   └── voice_matching.py # Stage 6
│   ├── integrations/       # External services
│   │   ├── gcs.py
│   │   ├── kitesforu_api.py
│   │   └── llm.py
│   └── models/             # Data models
├── tests/                  # 35 pytest tests
└── docker/                 # Docker support
```

## File References

| Purpose | Path |
|---------|------|
| CLI Entry | `src/kitesforu_qa/cli.py` |
| Pipeline | `src/kitesforu_qa/pipeline.py` |
| QA Stages | `src/kitesforu_qa/stages/` |
| Tests | `tests/` |
| Docker | `docker/Dockerfile` |

## See Also

- [LLM Service Spec](../../llm/services/qa.yaml)
- [Workers Pipeline](../workers/PIPELINE.md)
- [Content Evaluation](../workers/MODEL_ROUTER.md)
