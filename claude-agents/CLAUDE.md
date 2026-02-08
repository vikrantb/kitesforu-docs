# KitesForU Agent Ecosystem

This project has 15 specialized Claude Code agents and 16 knowledge files for the KitesForU platform.

## Quick Routing Guide

### "I want to improve interview prep quality"
→ Use `interview-prep-lead` — orchestrates workers-engineer, api-engineer, debugger, content-designer

### "Something is broken / job is stuck"
→ Use `kforu-debugger` — diagnostic specialist with GCP log queries, debug page guidance

### "Production is down / critical error"
→ Use `kforu-incident-responder` — 5-step incident response protocol

### "I want to change a prompt"
→ Use `kforu-workers-engineer` — knows every prompt file location (YAML for course workers, Python dicts for podcast workers)

### "I want to change the API / extraction logic"
→ Use `kforu-api-engineer` — FastAPI specialist, knows extraction prompts, credit system, Stripe

### "I want to change the frontend / UI"
→ Use `kforu-frontend-engineer` — Next.js App Router, MUST use pnpm

### "I need UX design / redesign a flow"
→ Use `kforu-ux-expert` — produces design specs (not code), knows brand, navigation, UX issues

### "I need to write user-facing copy / error messages"
→ Use `kforu-content-designer` — microcopy, marketing, brand voice

### "I want to change model selection / add a provider"
→ Use `kforu-model-expert` — model_catalog.csv, circuit breaker, quota, routing algorithm

### "I need to change infra / deploy something"
→ Use `kforu-infra-engineer` — Terraform, GCP, CI/CD matrix

### "I need to check Firestore / data issues"
→ Use `kforu-data-engineer` — 14 collections, schemas package, Pub/Sub

### "Audio quality is bad / TTS issues"
→ Use `kforu-audio-expert` — 3 TTS providers, voice matching, pacing, quality diagnostics

### "How much does this cost / pricing question"
→ Use `kforu-finance-manager` — credit formula, tiers, stage budgets, SaaS economics

### "Run tests / check quality"
→ Use `kforu-qa-engineer` — Jest, Playwright, pytest across all repos

### "Cross-feature decision / strategy / audit"
→ Use `kforu-product-lead` — coordinator, NOT for daily tasks

## Agent Architecture

```
Layer 1: Feature Leads (PM + Tech Lead combined)
  interview-prep-lead — Interview prep feature owner

Layer 2: Platform Specialists (shared across features)
  kforu-workers-engineer    — Worker systems (course + podcast)
  kforu-api-engineer        — FastAPI backend
  kforu-frontend-engineer   — Next.js frontend
  kforu-data-engineer       — Firestore + schemas
  kforu-infra-engineer      — Terraform + GCP + CI/CD
  kforu-model-expert        — Model selection + providers
  kforu-audio-expert        — TTS + voice + audio quality
  kforu-finance-manager     — Credits + pricing + cost

Layer 3: Design & Content
  kforu-ux-expert           — UX design (specs, not code)
  kforu-content-designer    — Microcopy + brand voice

Layer 4: Operations
  kforu-debugger            — Diagnostics + debug pages
  kforu-qa-engineer         — Testing across all repos
  kforu-incident-responder  — Production emergencies

Layer 5: Coordination
  kforu-product-lead        — Cross-cutting decisions only
```

## Knowledge Base

All agents reference files in `.claude/knowledge/`:
- `architecture-overview.md` — System map, repos, tech stacks
- `prompt-architecture.md` — Every LLM prompt in the system (45+ files)
- `firestore-schema.md` — All 14 Firestore collections
- `model-system.md` — Model catalog, selection algorithm, circuit breaker
- `cost-reference.md` — Pricing tiers, credit formula, budgets
- `deployment-guide.md` — Service deployment matrix, CI/CD
- `interview-prep-domain.md` — Deep coaching expertise (10 geographies, 8 industries)
- `audio-production.md` — TTS providers, voice matching, quality standards
- `quality-rubrics.md` — Scoring criteria for curriculum + content
- `quality-baselines.md` — Good vs bad output examples
- `ux-patterns.md` — Brand, navigation, design system, known issues
- `content-guidelines.md` — Brand voice, terminology, microcopy standards
- `prompt-changelog.md` — Track prompt changes over time
- `failure-patterns.md` — Diagnostic playbook for known issues
- `lessons-learned.md` — Accumulated learnings from past work

## Critical Rules

1. **pnpm only** — Frontend Docker/CI uses `pnpm install --frozen-lockfile`. Never use npm.
2. **PR workflow mandatory** — Never push to main. Always feature branch + PR.
3. **Two prompt systems** — Course workers: YAML files. Podcast workers: Python dicts in templates.py.
4. **Duration is float** — `module_duration_min` is float (0.167 for 10s), not int.
5. **SupportedLanguage** — Literal type, use `cast()` not direct str assignment.
6. **Verify deployments** — Code on main != deployed. Always check Cloud Run revisions.
