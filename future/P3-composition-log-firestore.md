# P3: Store Prompt Composition Log in Firestore

## Problem
CompositionLog (which sections included, from which YAML, why) only in local logs. Not on debug page.

## Fix
Store log in Firestore after composing in streaming_generator._build_composed_prompt().

## Code Paths
- src/workers/prompts/composer/composer.py — CompositionLog
- src/workers/stages/script/streaming_generator.py — _build_composed_prompt()

## Priority: P3 | Impact: Low (debug only) | Effort: 1 hour
