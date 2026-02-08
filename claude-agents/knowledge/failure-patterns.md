# Failure Patterns — Diagnostic Playbook
<!-- Append new patterns as they're discovered -->
<!-- Format: Symptom → Root Cause → Fix Location → Fix Type → Verified Date -->

## Curriculum Issues

### Generic episode titles
- **Root cause**: Anti-generic guardrails in curriculum prompt not strong enough
- **Fix location**: `kitesforu-course-workers/src/workers/prompts/stages/syllabus/interview_prep_curriculum.yaml`
- **Fix type**: Strengthen ANTI-GENERIC GUARDRAILS section
- **Verified**: 2026-02-07

### No gap analysis in curriculum
- **Root cause**: User prompt doesn't include gap analysis instructions, or LLM ignores them
- **Fix location**: interview_prep_curriculum.yaml (user template section)
- **Fix type**: Make gap analysis instructions more prominent/required

### Wrong episode progression
- **Root cause**: Episode Progression Strategy section not specific enough
- **Fix location**: interview_prep_curriculum.yaml
- **Fix type**: Add explicit ordering rules

## Episode Content Issues

### Episodes don't reference candidate's resume
- **Root cause**: Context not being passed through orchestrator templates
- **Fix location**: `kitesforu-course-workers/src/workers/stages/orchestrator/worker.py` + episode type YAMLs
- **Fix type**: Code fix (template variable mapping) or template enhancement

### Robotic-sounding script
- **Root cause**: Script generation prompt lacks pacing cues
- **Fix location**: `kitesforu-workers/src/workers/prompting/templates.py` (script section)
- **Fix type**: Add [PAUSE], [EMPHASIS] markers to prompt instructions

### Factually thin content
- **Root cause**: Research prompt too shallow or research stage skipped
- **Fix location**: templates.py (research section) or episode custom_instructions
- **Fix type**: Enhance research instructions

## Infrastructure Issues

### Job stuck in processing
- **Root cause**: Worker timeout, Pub/Sub acknowledgment issue, or downstream service failure
- **Diagnosis**: Check Cloud Run logs, Pub/Sub message status
- **Fix type**: Varies — may need worker restart, message replay, or bug fix

### TTS fallback producing lower quality
- **Root cause**: Primary TTS provider unhealthy, circuit breaker opened
- **Diagnosis**: Check /admin/model-router health dashboard
- **Fix type**: Wait for provider recovery, or adjust circuit breaker thresholds

(Add new patterns as they're discovered during debugging sessions.)
