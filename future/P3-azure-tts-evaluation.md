# P3: Evaluate Microsoft Azure TTS for Enterprise Tier

## Problem
Azure has richest expression control (15+ styles with intensity), SOC 2/HIPAA compliance,
150+ languages, on-prem deployment, visemes for future avatars.

## Why
Enterprise customers need compliance. Azure's explicit SSML control is ideal for
deterministic compliance training output.

## Code Paths
- src/workers/stages/audio/providers/ — add azure/ package
- src/workers/stages/audio/voice_intelligence/adapters/ — add azure_adapter.py
- config/voice_intelligence/providers/ — add azure.yaml

## Priority: P3 | Impact: Strategic (enterprise) | Effort: 1-2 days eval, 3-5 days integration
