# P3: Log Audio Config Decisions to Firestore

## Problem
log_audio_config_decision() exists in debug_logging.py but is NEVER CALLED in the streaming
pipeline. Content type detection and provider selection reasoning invisible on debug page.

## Fix
Call the existing function in streaming_script_audio_worker.py after audio config is created.

## Code Paths
- src/workers/common/debug_logging.py:1212-1310
- src/workers/stages/combined/streaming_script_audio_worker.py:~340

## Priority: P3 | Impact: Low (debug only) | Effort: 1 hour
