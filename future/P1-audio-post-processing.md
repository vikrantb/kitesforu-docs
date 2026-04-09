# P1: Audio Post-Processing Warmth Chain

## Problem
TTS output sounds "clean but clinical." Professional podcasts sound warmer and more intimate.
Missing: EQ, compression, room reverb, de-essing.

## Fix: Single ffmpeg Filter Chain (~200ms overhead)
```
highpass=f=80:p=2,
equalizer=f=150:t=h:w=200:g=2.5,     # Warmth (proximity effect)
equalizer=f=300:t=q:w=1.5:g=-2,       # Anti-mud
equalizer=f=3000:t=q:w=2:g=1.5,       # Presence
equalizer=f=6500:t=q:w=1.5:g=-2.5,    # De-ess
equalizer=f=12000:t=h:w=1:g=-3,       # Air rolloff
acompressor=threshold=0.1:ratio=2:attack=10:release=150:makeup=2,
acompressor=threshold=0.125:ratio=3:attack=30:release=250:makeup=3:knee=6,
aecho=0.8:0.75:6|12|18:0.12|0.08|0.04,  # Subtle room
loudnorm=I=-16:TP=-1.5:LRA=11
```

Gender-specific variants (male: boost 140Hz, female: boost 250Hz + 11kHz air).
Genre-specific reverb (horror: more, comedy: less).

## Code Path
- src/workers/stages/audio/audio_combiner.py → _lufs_normalize() becomes _post_process()

## Priority: P1 | Impact: 20-30% warmth improvement | Effort: 2-3 hours
