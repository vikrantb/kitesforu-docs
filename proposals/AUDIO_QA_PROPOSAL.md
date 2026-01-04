# Audio Quality Assurance System Proposal

> **Status**: Draft Proposal for Review
> **Created**: 2026-01-04
> **Author**: AI-assisted research

## Executive Summary

This proposal outlines a comprehensive Audio Quality Assurance (QA) system for KitesForU podcast generation. The system validates generated audio across four dimensions:

1. **Format Validation** - Technical audio specifications
2. **Pronunciation Accuracy** - Words spoken correctly
3. **Audio Quality Scoring** - Perceptual quality measurement
4. **Content Verification** - Script-to-audio alignment

**Recommended Approach**: A hybrid system using primarily **free/open-source tools** with optional paid API fallbacks for enhanced accuracy.

---

## Problem Statement

Generated podcasts need quality validation before delivery to ensure:
- Audio file is technically valid (correct format, bitrate, sample rate)
- Words are pronounced correctly (no mispronunciations, garbled speech)
- Audio quality meets minimum standards (clear, natural-sounding)
- Generated audio matches the intended script content

---

## Proposed QA Pipeline

```
Audio File → Format Check → Transcription → Quality Scoring → Report
     ↓             ↓              ↓                ↓
  ffprobe     Whisper STT    UTMOS/DNSMOS    Aggregated
  validation   + WER calc    neural MOS       Results
```

### Stage 1: Format Validation

**Purpose**: Verify audio file meets technical specifications

**Tools** (FREE):
- `ffprobe` (part of FFmpeg) - Extract audio metadata
- `pydub` Python library - Audio file validation

**Checks**:
| Check | Expected Value | Action if Failed |
|-------|---------------|------------------|
| Format | MP3/WAV | Reject |
| Sample Rate | 44100 or 48000 Hz | Warning |
| Bitrate | ≥128 kbps | Warning |
| Channels | 1 (mono) or 2 (stereo) | Info |
| Duration | Matches expected ±5% | Warning |
| Corruption | No errors | Reject |

**Implementation**:
```python
import subprocess
import json

def validate_audio_format(audio_path: str) -> dict:
    """Validate audio file format using ffprobe."""
    cmd = [
        'ffprobe', '-v', 'quiet', '-print_format', 'json',
        '-show_format', '-show_streams', audio_path
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)

    if result.returncode != 0:
        return {"valid": False, "error": "File corrupt or unreadable"}

    data = json.loads(result.stdout)
    audio_stream = next(
        (s for s in data.get('streams', []) if s['codec_type'] == 'audio'),
        None
    )

    return {
        "valid": True,
        "format": data['format']['format_name'],
        "duration": float(data['format']['duration']),
        "bitrate": int(data['format'].get('bit_rate', 0)),
        "sample_rate": int(audio_stream['sample_rate']) if audio_stream else None,
        "channels": audio_stream['channels'] if audio_stream else None,
    }
```

**Cost**: FREE (ffprobe is open source)

---

### Stage 2: Speech-to-Text & Pronunciation Accuracy

**Purpose**: Transcribe audio and compare against original script to detect:
- Mispronunciations
- Missing words
- Added words
- Overall accuracy

**Primary Tool** (FREE):
- **OpenAI Whisper (local)** - Run locally, no API costs
  - Model: `whisper-large-v3` (best accuracy)
  - Languages: 99+ supported
  - Accuracy: 95-98% on clear audio

**Metric**: Word Error Rate (WER)
```
WER = (Substitutions + Insertions + Deletions) / Total Reference Words
```

**Tools**:
- `openai-whisper` - Local STT inference
- `jiwer` - WER calculation library (FREE)

**Implementation**:
```python
import whisper
from jiwer import wer, cer

def transcribe_and_compare(audio_path: str, expected_script: str) -> dict:
    """Transcribe audio and calculate accuracy against script."""

    # Load Whisper model (runs locally - FREE)
    model = whisper.load_model("large-v3")

    # Transcribe
    result = model.transcribe(audio_path)
    transcribed_text = result["text"]

    # Calculate Word Error Rate
    word_error_rate = wer(expected_script, transcribed_text)
    char_error_rate = cer(expected_script, transcribed_text)

    return {
        "transcription": transcribed_text,
        "expected_script": expected_script,
        "word_error_rate": word_error_rate,
        "char_error_rate": char_error_rate,
        "accuracy_score": 1 - word_error_rate,  # 0-1 scale
        "passed": word_error_rate < 0.10,  # 90% accuracy threshold
    }
```

**WER Thresholds**:
| WER | Quality Level | Action |
|-----|---------------|--------|
| < 5% | Excellent | Pass |
| 5-10% | Good | Pass with note |
| 10-15% | Acceptable | Review recommended |
| > 15% | Poor | Regenerate audio |

**Cost Comparison** (if API needed):

| Provider | Price/Minute | Free Tier | Notes |
|----------|-------------|-----------|-------|
| **Whisper Local** | **FREE** | Unlimited | Requires GPU for speed |
| OpenAI Whisper API | $0.006/min | None | Best accuracy |
| AssemblyAI | $0.0025/min | $50 credit | Good accuracy |
| Deepgram | $0.0043/min | $200 credit | Fast |
| Azure Speech | $0.0167/min | 5 hrs/month | Includes pronunciation assessment |

**Recommendation**: Use Whisper locally (FREE). For 1000 10-minute podcasts/month:
- Local Whisper: $0
- Whisper API: $60/month
- Azure: $167/month

---

### Stage 3: Audio Quality Scoring (MOS Prediction)

**Purpose**: Automatically predict perceptual audio quality without human listening tests

**Metrics**:
- **MOS (Mean Opinion Score)**: 1-5 scale of perceived quality
- **DNSMOS**: Microsoft's neural MOS predictor (designed for noisy speech)
- **UTMOS**: Strong learner for TTS evaluation

**Primary Tools** (FREE):

1. **UTMOS** - Universal TTS Mean Opinion Score
   - GitHub: `tarepan/SpeechMOS`
   - Designed for TTS evaluation
   - Predicts MOS without reference audio

2. **DNSMOS** - Deep Noise Suppression MOS
   - Microsoft's neural quality predictor
   - P.808 (overall) and P.835 (sig, bak, ovrl) scores

3. **speechmos** PyPI Package
   - Easy-to-use wrapper for multiple MOS predictors
   - Includes UTMOS and DNSMOS

**Implementation**:
```python
# Option 1: Using speechmos library
from speechmos import UTMOS

def calculate_mos(audio_path: str) -> dict:
    """Calculate predicted MOS score using neural network."""
    predictor = UTMOS()
    score = predictor.predict(audio_path)

    return {
        "mos_score": score,
        "quality_level": get_quality_level(score),
        "passed": score >= 3.5,
    }

def get_quality_level(mos: float) -> str:
    if mos >= 4.5: return "Excellent"
    if mos >= 4.0: return "Good"
    if mos >= 3.5: return "Acceptable"
    if mos >= 3.0: return "Poor"
    return "Bad"
```

**MOS Thresholds**:
| MOS Score | Quality | Action |
|-----------|---------|--------|
| 4.5-5.0 | Excellent | Pass |
| 4.0-4.5 | Good | Pass |
| 3.5-4.0 | Acceptable | Pass with note |
| 3.0-3.5 | Poor | Review/regenerate |
| < 3.0 | Bad | Regenerate |

**Cost**: FREE (all models are open source)

---

### Stage 4: Prosody & Naturalness Assessment

**Purpose**: Evaluate speech rhythm, intonation, and naturalness

**Metrics** (all computable locally):

1. **Pitch (F0) Analysis**
   - F0 mean, std, range
   - Pitch contour smoothness
   - Compare to human speech norms

2. **Speaking Rate**
   - Words per minute
   - Pause distribution
   - Rhythm consistency

3. **Energy/Loudness**
   - RMS energy profile
   - Dynamic range
   - Loudness normalization (LUFS)

**Tools** (FREE):
- `librosa` - Audio feature extraction
- `praat-parselmouth` - Prosody analysis
- `pyloudnorm` - Loudness measurement

**Implementation**:
```python
import librosa
import numpy as np

def analyze_prosody(audio_path: str) -> dict:
    """Analyze prosodic features of audio."""
    y, sr = librosa.load(audio_path)

    # Pitch analysis
    f0, voiced_flag, voiced_probs = librosa.pyin(
        y, fmin=50, fmax=500, sr=sr
    )
    f0_valid = f0[~np.isnan(f0)]

    # Speaking rate (rough estimate via onset detection)
    onset_env = librosa.onset.onset_strength(y=y, sr=sr)
    tempo = librosa.beat.tempo(onset_envelope=onset_env, sr=sr)[0]

    # Energy analysis
    rms = librosa.feature.rms(y=y)[0]

    return {
        "pitch_mean": float(np.mean(f0_valid)) if len(f0_valid) > 0 else 0,
        "pitch_std": float(np.std(f0_valid)) if len(f0_valid) > 0 else 0,
        "pitch_range": float(np.ptp(f0_valid)) if len(f0_valid) > 0 else 0,
        "tempo_bpm": float(tempo),
        "energy_mean": float(np.mean(rms)),
        "energy_std": float(np.std(rms)),
        "dynamic_range_db": float(20 * np.log10(np.max(rms) / np.mean(rms) + 1e-10)),
    }
```

**Prosody Quality Indicators**:
| Metric | Good Range | Notes |
|--------|------------|-------|
| Pitch Mean (F0) | 85-255 Hz | Varies by voice |
| Pitch Std | 30-80 Hz | Too low = monotone |
| Speaking Rate | 120-180 WPM | Podcast optimal |
| Dynamic Range | 6-15 dB | Natural variation |

**Cost**: FREE

---

## Complete QA Pipeline Implementation

```python
from dataclasses import dataclass
from typing import Optional
import json

@dataclass
class QAResult:
    job_id: str
    audio_path: str

    # Format validation
    format_valid: bool
    format_details: dict

    # Pronunciation accuracy
    word_error_rate: float
    transcription: str
    pronunciation_passed: bool

    # Quality score
    mos_score: float
    quality_level: str
    quality_passed: bool

    # Prosody
    prosody_metrics: dict

    # Overall
    overall_passed: bool
    issues: list[str]
    recommendations: list[str]

class AudioQAPipeline:
    """Complete audio quality assurance pipeline."""

    def __init__(self):
        self.whisper_model = None  # Lazy load
        self.mos_predictor = None  # Lazy load

    def run_qa(
        self,
        job_id: str,
        audio_path: str,
        expected_script: str,
        min_mos: float = 3.5,
        max_wer: float = 0.10,
    ) -> QAResult:
        """Run complete QA pipeline on audio file."""

        issues = []
        recommendations = []

        # Stage 1: Format validation
        format_result = validate_audio_format(audio_path)
        if not format_result["valid"]:
            issues.append(f"Format validation failed: {format_result.get('error')}")

        # Stage 2: Transcription & WER
        transcript_result = self.transcribe_and_compare(audio_path, expected_script)
        if transcript_result["word_error_rate"] > max_wer:
            issues.append(f"WER too high: {transcript_result['word_error_rate']:.1%}")
            recommendations.append("Consider regenerating audio with different TTS settings")

        # Stage 3: MOS prediction
        mos_result = self.calculate_mos(audio_path)
        if mos_result["mos_score"] < min_mos:
            issues.append(f"MOS score below threshold: {mos_result['mos_score']:.2f}")
            recommendations.append("Audio quality may not meet user expectations")

        # Stage 4: Prosody analysis
        prosody_result = analyze_prosody(audio_path)

        # Determine overall pass/fail
        overall_passed = (
            format_result["valid"] and
            transcript_result["word_error_rate"] <= max_wer and
            mos_result["mos_score"] >= min_mos
        )

        return QAResult(
            job_id=job_id,
            audio_path=audio_path,
            format_valid=format_result["valid"],
            format_details=format_result,
            word_error_rate=transcript_result["word_error_rate"],
            transcription=transcript_result["transcription"],
            pronunciation_passed=transcript_result["passed"],
            mos_score=mos_result["mos_score"],
            quality_level=mos_result["quality_level"],
            quality_passed=mos_result["passed"],
            prosody_metrics=prosody_result,
            overall_passed=overall_passed,
            issues=issues,
            recommendations=recommendations,
        )
```

---

## Cost Analysis

### Option A: Fully Free (Recommended for Start)

| Component | Tool | Cost |
|-----------|------|------|
| Format Validation | ffprobe | $0 |
| Speech-to-Text | Whisper (local) | $0 |
| WER Calculation | jiwer | $0 |
| MOS Prediction | UTMOS/speechmos | $0 |
| Prosody Analysis | librosa | $0 |
| **Total** | | **$0/month** |

**Requirements**:
- GPU for Whisper inference (optional but recommended)
- ~4GB RAM for Whisper large-v3 model
- CPU-only mode available (slower)

### Option B: Hybrid (Free + Cheap API Fallback)

| Component | Primary (Free) | Fallback (Paid) |
|-----------|---------------|-----------------|
| Format Validation | ffprobe | - |
| Speech-to-Text | Whisper local | OpenAI Whisper API ($0.006/min) |
| MOS Prediction | UTMOS | - |
| Pronunciation Assessment | - | Azure Speech ($0.0167/min) |

**Estimated Monthly Cost** (1000 podcasts × 10 min avg):
- If 90% local, 10% API fallback: ~$6/month
- If 100% API: ~$60/month

### Option C: Premium (Maximum Accuracy)

| Component | Tool | Cost/Month (10K min) |
|-----------|------|---------------------|
| STT | OpenAI Whisper API | $60 |
| Pronunciation | Azure Speech | $167 |
| Quality | - | $0 (still free) |
| **Total** | | **$227/month** |

---

## Implementation Phases

### Phase 1: MVP (Week 1-2)
**Scope**: Basic validation pipeline

1. Format validation with ffprobe
2. Whisper local transcription
3. WER calculation with jiwer
4. Simple pass/fail reporting

**Deliverables**:
- Python module for QA pipeline
- CLI tool for manual testing
- Integration with worker pipeline

### Phase 2: Quality Scoring (Week 3)
**Scope**: Add perceptual quality metrics

1. Integrate UTMOS MOS prediction
2. Add prosody analysis
3. Build quality dashboard/reporting

**Deliverables**:
- MOS scoring integration
- Quality metrics in job results
- Firestore schema for QA results

### Phase 3: Advanced Features (Week 4+)
**Scope**: Enhanced capabilities

1. Word-level alignment (which words failed)
2. Automated regeneration triggers
3. A/B testing different TTS providers
4. Historical quality trending

---

## Database Schema

```yaml
# Collection: job_qa_results
document:
  job_id: string
  audio_url: string
  created_at: timestamp

  format:
    valid: boolean
    codec: string
    sample_rate: number
    bitrate: number
    duration_seconds: number

  transcription:
    text: string
    word_error_rate: number
    char_error_rate: number
    passed: boolean

  quality:
    mos_score: number
    quality_level: string  # Excellent/Good/Acceptable/Poor/Bad
    passed: boolean

  prosody:
    pitch_mean: number
    pitch_std: number
    speaking_rate_wpm: number
    dynamic_range_db: number

  overall:
    passed: boolean
    issues: array<string>
    recommendations: array<string>
```

---

## API Integration

### Worker Integration

Add QA stage after audio generation:

```python
# In audio worker, after generating audio
async def process_audio_stage(job: Job) -> None:
    # ... existing audio generation code ...

    # NEW: Run QA pipeline
    qa_result = qa_pipeline.run_qa(
        job_id=job.job_id,
        audio_path=local_audio_path,
        expected_script=job.script,
    )

    # Store QA results
    await store_qa_results(job.job_id, qa_result)

    # Check if regeneration needed
    if not qa_result.overall_passed:
        if qa_result.word_error_rate > 0.15:
            # Severe pronunciation issues - regenerate
            await trigger_regeneration(job, reason="pronunciation")
        elif qa_result.mos_score < 3.0:
            # Quality too low - regenerate
            await trigger_regeneration(job, reason="quality")
        else:
            # Minor issues - log warning but continue
            logger.warning(f"QA issues for {job.job_id}: {qa_result.issues}")
```

### API Endpoint

```python
@router.get("/jobs/{job_id}/qa")
async def get_job_qa_results(
    job_id: str,
    user: dict = Depends(get_current_user)
) -> QAResultResponse:
    """Get QA results for a completed job."""
    qa_doc = await db.collection("job_qa_results").document(job_id).get()

    if not qa_doc.exists:
        raise HTTPException(404, "QA results not found")

    return QAResultResponse(**qa_doc.to_dict())
```

---

## Dependencies

### Python Packages

```txt
# Core QA pipeline
openai-whisper>=20231117  # Local STT
jiwer>=3.0.0              # WER calculation
speechmos>=0.1.0          # MOS prediction (UTMOS)
librosa>=0.10.0           # Audio analysis
pyloudnorm>=0.1.0         # Loudness measurement
pydub>=0.25.0             # Audio file handling

# Optional for prosody
praat-parselmouth>=0.4.0  # Advanced prosody analysis
```

### System Dependencies

```bash
# FFmpeg (for ffprobe)
apt-get install ffmpeg  # Ubuntu/Debian
brew install ffmpeg     # macOS
```

---

## Monitoring & Alerting

### Metrics to Track

| Metric | Alert Threshold | Action |
|--------|-----------------|--------|
| QA Pass Rate | < 90% | Investigate TTS issues |
| Avg MOS Score | < 4.0 | Review TTS provider |
| Avg WER | > 8% | Check script quality |
| QA Processing Time | > 60s | Scale resources |

### Dashboard Queries (BigQuery/Firestore)

```sql
-- Daily QA summary
SELECT
  DATE(created_at) as date,
  COUNT(*) as total_jobs,
  COUNTIF(overall.passed) as passed,
  AVG(quality.mos_score) as avg_mos,
  AVG(transcription.word_error_rate) as avg_wer
FROM job_qa_results
GROUP BY DATE(created_at)
ORDER BY date DESC
```

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Whisper GPU memory | High | Use smaller model or batch processing |
| False positives in WER | Medium | Tune thresholds, add human review queue |
| MOS prediction inaccuracy | Medium | Validate against human ratings periodically |
| Processing time overhead | Low | Run QA async, don't block delivery |

---

## Recommendations

### Immediate Actions

1. **Start with Option A (Free)** - No cost, validate approach
2. **Integrate format validation first** - Quick win, catches obvious failures
3. **Add Whisper + WER second** - Core pronunciation checking
4. **MOS scoring third** - Quality perception layer

### Future Enhancements

1. **Word-level error highlighting** - Show exactly which words failed
2. **TTS provider comparison** - A/B test ElevenLabs vs OpenAI TTS
3. **User feedback loop** - Correlate QA scores with user ratings
4. **Automated reprocessing** - Retry with different voice/settings

---

## Conclusion

This proposal outlines a **cost-effective, scalable audio QA system** using primarily open-source tools:

- **$0/month** for the basic pipeline
- **< $10/month** with API fallbacks for edge cases
- **4-stage validation**: format, pronunciation, quality, prosody
- **Automated pass/fail** with detailed diagnostics
- **Easy integration** with existing worker pipeline

The system can start simple (format + WER) and grow to include advanced quality metrics as needed.

---

## Appendix A: Tool Comparison Matrix

| Tool | Type | Cost | Accuracy | Speed | Notes |
|------|------|------|----------|-------|-------|
| Whisper Large-v3 | STT | Free | 95-98% | Slow | Best free option |
| Whisper Medium | STT | Free | 90-95% | Medium | Good balance |
| OpenAI Whisper API | STT | $0.006/min | 98%+ | Fast | Cloud-based |
| jiwer | WER | Free | N/A | Fast | Standard library |
| UTMOS | MOS | Free | Good | Fast | TTS-focused |
| DNSMOS | MOS | Free | Good | Fast | Noisy speech |
| Azure Pronunciation | Pron | $0.0167/min | Excellent | Fast | Phoneme-level |

## Appendix B: Sample QA Report

```json
{
  "job_id": "job_abc123",
  "audio_url": "gs://kitesforu-audio/job_abc123/final.mp3",
  "created_at": "2026-01-04T10:30:00Z",

  "format": {
    "valid": true,
    "codec": "mp3",
    "sample_rate": 44100,
    "bitrate": 192000,
    "duration_seconds": 612.5
  },

  "transcription": {
    "word_error_rate": 0.032,
    "char_error_rate": 0.018,
    "passed": true
  },

  "quality": {
    "mos_score": 4.23,
    "quality_level": "Good",
    "passed": true
  },

  "prosody": {
    "pitch_mean": 165.2,
    "pitch_std": 42.8,
    "speaking_rate_wpm": 148,
    "dynamic_range_db": 8.5
  },

  "overall": {
    "passed": true,
    "issues": [],
    "recommendations": []
  }
}
```
