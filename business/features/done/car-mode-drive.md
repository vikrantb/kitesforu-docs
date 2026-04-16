# Car Mode / Drive

**Status**: DONE
**Shipped**: March-April 2026
**Repos**: kitesforu-frontend, kitesforu-workers
**Key files**: app/car-mode/, components/drive/, hooks/car-mode-v2/, lib/drive-audio-bridge.ts

## What it does

Instant audio playback during generation. Users hit "Drive" instead of "Create" and the first episode starts playing within seconds while remaining episodes generate in the background. Designed for hands-free listening (commute, gym, walk).

## Key capabilities

- **Instant first-episode playback**: intro hook audio plays immediately while full generation runs
- **Sequential segment streaming**: episodes stream to the player as they complete
- **Voice command integration**: useCarVoiceOrchestrator with real SpeechRecognition/SpeechSynthesis
- **Mode switching**: commands vs. chat modes
- **Quick Question overlay**: ask questions about the content mid-listen
- **Audio bridge pattern**: module-level state for cross-page audio continuity
- **Volume ducking**: episode audio ducks when Q&A speech is active

## User impact

Users can start listening within seconds of creating content, rather than waiting 2-5 minutes for full generation. The audio plays continuously like a podcast app, and voice commands allow hands-free control.
