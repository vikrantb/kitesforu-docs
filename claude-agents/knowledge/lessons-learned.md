# Lessons Learned
<!-- Append new lessons as they're discovered -->

## Deployment Lessons
- Always verify Cloud Run actually got new revisions after merging PRs — code on main != deployed
- Check revision timestamps: `gcloud run revisions list --service=<name> --region=us-central1`
- Course workers Cloud Build needs explicit cd + SHORT_SHA substitution
- Frontend uses pnpm — ALWAYS run pnpm install (not npm) when adding deps

## Code Lessons
- SupportedLanguage is a Literal type — use cast(SupportedLanguage, value) not direct assignment
- module_duration_min is a float (0.167 for 10 seconds), NOT int
- budget_by_stage values are in USD, not credits
- pipeline_config in Firestore toggles features at RUNTIME

## Architecture Lessons
- Two prompt systems exist: YAML (course workers) and Python dicts (podcast workers) — know the difference
- Episode type templates are NOT direct LLM prompts — they become custom_instructions
- Enriched context is built once in API and passed through Firestore to all workers
- Course workers on legacy Cloud Build — should migrate to GH Actions

## Quality Lessons
- Generic output is usually a prompt problem, not a model problem
- The 5-second rule works: every 5 seconds must deliver value
- Anti-generic guardrails need to be VERY explicit to work
- Temperature 0.7 is good for creative generation, 0.1 for extraction

(Add new lessons as they emerge from debugging and development sessions.)
