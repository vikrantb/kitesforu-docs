# R2 — Scenario / Type Guidance UI

**Status**: PROPOSED
**Priority**: P1
**Effort**: 2 weeks (1 week research, 1 week implementation)
**Affected repos**: kitesforu-frontend, kitesforu-docs
**Depends on**: R2-template-input-coaching (shipped), R2-curated-type-examples (shipped)
**Adjacent to**: R2-rich-input-with-attachments.md, R1-phase3-ux-creation-home.md

---

## Problem

The Smart Create entry point asks the user to describe what they want in a single textarea. Today the placeholder animates one example string (via `lib/template-coaching.ts` when a template is picked, or `lib/curated-examples.ts` when no template is picked). That shipped under R2-template-input-coaching, and it pulls the average input quality up — users who copy the depth of the example produce richer plans and skip clarifying questions.

But three things are still wrong:

1. **Textarea is sized for a tweet.** `min-h-[80px]` tells the user "one sentence is fine," when a sentence rarely carries the specificity the planner needs. The animation shows a 400-char example typing itself into a 2-line box — the user sees the end of the string cut off and anchors to the input's size, not the animation's implied density.
2. **Guidance is per-template, not per-scenario.** A user who lands on the page without picking a template sees one generic example. A user who types free-form text never sees scenario-specific guidance unless they first click a template. The "interview-prep" scenario has very different input needs than "storytelling," but the textarea treats them identically.
3. **Examples don't teach the "highly specific data" the planner actually wants.** The placeholders are concrete ("Senior PM at Google — focus on cross-functional leadership") but they don't surface the *schema* of a strong input. A user sees the string but doesn't learn that they should provide: role seniority, company, interview focus (behavioral / system design / case), whether they have a resume to upload, and target timeline.

The result is users who keep asking "what should I type?" and a planner that keeps asking clarifying questions because the input is under-specified. Both are fixable in the same surface.

## Scope

**In scope**
- Scenario-aware input sizing: the textarea grows (≥ 6 visible rows) when the active scenario benefits from more detail, and keeps the tweet-sized default when it doesn't
- Scenario-specific "active typing" affordance: the current animation lifts from per-template to per-scenario (mapped through the template it resolves to, or a cross-scenario default when no template is selected)
- Exemplary *input schema* overlay: below the animation, render a short structured hint — "Strong inputs usually say: role + seniority, target company (optional), interview focus, topics to cover, resume link" — so the user learns *what* to include, not just *a good string*
- Deep-research per scenario to produce both the animation strings and the schema hints grounded in how the corresponding planner actually uses the input
- Copy discipline: specific named examples (Stripe L5, Yale PHIL 176, GDPR Art 17) not synthetic filler

**Out of scope**
- Structured form inputs (role / company / focus as separate fields) — this proposal keeps the single textarea
- Chat-style multi-turn capture — R2-rich-input-with-attachments covers that axis
- Backend changes — every signal is already available from `lib/templates.ts` + `lib/curated-examples.ts`
- New animation systems — extends the existing `lib/typewriter.ts`

## Scenarios (the taxonomy)

These are the nine scenario buckets the proposal targets. Each needs its own per-scenario deep-research pass (one agent, one brief) before the implementation ships. The template → scenario mapping lives in the proposal so the UI can pick the right scenario even when the user has picked a template.

1. **interview-prep** — candidate preparing for a specific interview (behavioral, technical, system design, case)
2. **skill-mastery** — self-directed learner mastering a topic (concept-deep-dive, tutorial-series)
3. **exam-prep** — student prepping for a named exam (MCAT, bar, AP)
4. **storytelling** — creative fiction (horror-series, bedtime-stories, drama)
5. **explainer-podcast** — factual podcast on a topic (news-analysis, explainer, business-deep-dive)
6. **k12-lesson** — K-12 teacher producing a lesson aligned to a standard
7. **university-lecture** — professor producing a lecture / lecture series
8. **corporate-training** — L&D producing a compliance / onboarding module
9. **writeup** — non-audio artifact (blog post, brief, study guide)

Each scenario's deep-research brief must answer:
- What 3–5 pieces of information does the planner for this scenario *actually* consume? (grep the planner / curriculum-builder code)
- What named real-world examples are concrete enough to teach density without dictating the user's topic?
- What's the cheapest way for the user to provide each piece of info without breaking the single-textarea UX?

Research output lands as a per-scenario entry in a new `lib/scenario-guidance.ts` keyed on the scenario id, with:
- `typedExemplar`: the 400–800-char animated string (density lesson)
- `schemaHints`: 3–5 short phrases the user can tick off ("role + seniority", "target company", "interview focus")
- `bigBox`: boolean — does this scenario expand the textarea or keep the default?
- `researchedBy` / `researchedAt` / `version`: audit trail, same shape as `template-coaching.ts`

## Architecture

```
IntentSection.tsx
  ├── existing: tab toggle (Say it | I have notes | Starting points)
  ├── existing: template chip + per-template animation (lib/template-coaching.ts)
  └── NEW: scenario resolution
         ├── derive scenario from (selectedTemplate, profile.primary_persona)
         ├── read lib/scenario-guidance.ts[scenario]
         ├── apply `bigBox` → grow textarea rows 2 → 6
         ├── swap animation source to scenario entry (template still wins when present)
         └── render `schemaHints` chips below the textarea
```

Key design rules:
- **One source of truth per scenario.** `lib/scenario-guidance.ts` is the per-scenario bundle (typedExemplar + schemaHints + bigBox). Template-specific overrides still live in `lib/template-coaching.ts` and take precedence when a template is selected; the scenario layer is the fallback the user sees before picking a template.
- **No change to lib/templates.ts schema.** Scenario is derived at render time from `template.content_type + template.category`, not stored on the template. This keeps the template registry stable and lets scenarios evolve independently.
- **No new API routes.** Everything is client-side constants; the existing feature-flag layer can A/B this surface without a backend round-trip.
- **Respect prefers-reduced-motion.** The typewriter already does; the larger textarea must not trigger a layout-shift animation for reduced-motion users.

## Research plan (week 1)

Nine parallel deep-research briefs, one per scenario above. Each brief:
1. Reads `lib/templates.ts` for the templates that map to the scenario
2. Reads the relevant planner/curriculum/executor code in kitesforu-api to find out what the planner actually uses (same pattern as the R3 Phase 2 personalization work)
3. Produces ONE typedExemplar + 3–5 schemaHints + a recommendation on `bigBox`
4. Cites 2–3 named examples (company, course, incident) the exemplar can hook onto so copy stays specific

Output format matches the existing `template-coaching.ts` entries so the implementation pass is mechanical.

## Implementation plan (week 2)

1. Create `lib/scenario-guidance.ts` with the 9 entries from research
2. Add `deriveScenario(template, persona)` helper in the same file
3. Extend `IntentSection.tsx` to resolve scenario, pass `bigBox` → dynamic `rows`, render schemaHints
4. Feature-flag `feature_scenario_guidance_live` (default OFF) — ship behind flag, flip after copy review
5. Analytics: fire `scenario_guidance_shown` (carries scenario id) and `scenario_guidance_hint_clicked` events (an optional click-to-expand on a schemaHint chip), so we can measure which hints users consume

## Acceptance Criteria

- [ ] Nine scenarios each have a typedExemplar + schemaHints + bigBox flag in `lib/scenario-guidance.ts`, each produced by a dedicated deep-research agent
- [ ] `IntentSection.tsx` grows the textarea to ≥ 6 rows when the active scenario's `bigBox=true`
- [ ] Scenario-specific animation runs when no template is selected (today it shows only the generic curated example)
- [ ] Schema hints render as chips under the textarea for the active scenario; clicking a chip expands a ≤ 2-line gloss (no heavy tooltip)
- [ ] Feature flag `feature_scenario_guidance_live` gates the whole surface
- [ ] Analytics events `scenario_guidance_shown` + `scenario_guidance_hint_clicked` wired via `lib/analytics.ts`
- [ ] Per-scenario copy passes a "named example" smell test (references a real Stripe L5 / Yale PHIL 176 / GDPR Art 17-level concrete anchor)
- [ ] Existing 34-template coaching still wins when a template is selected (scenario layer is the fallback, not the override)

## Why this sits behind template-coaching + curated-examples

Those two shipped a density lift on the animation and the home hero, but they left the scenario-agnostic case unhandled. A user who lands on /create-smart without a template pre-selected sees one cycling example across all types — which teaches one example per type, not one *scenario-specific* example per type. This proposal closes that gap and adds the schema hints that the prior work deliberately left out.

## Non-goals / deferred

- Per-scenario structured intake (role + company + focus fields) — explicitly rejected; keeps single-textarea
- Per-scenario tone / style presets (already handled by template defaults + R3 profile preferences)
- A/B testing scaffold for individual schema-hint phrasings — ships on the existing feature-flag layer; no per-string experimentation until v1 data lands

## Risks

- **Copy churn.** Nine scenarios × ~6 short strings each = ~54 copy lines that need to stay grounded. Weekly copy review is cheap; churn is only a risk if no one owns it
- **Schema-hints feel like form fields by accident.** The chips need to look like teaching hints, not toggles — implementation must render them with a "Suggested: ..." label and no affordance that implies interaction other than the ≤ 2-line gloss expansion
- **Scenario layer and template-coaching drift.** Mitigated by letting template-coaching win when present and having the scenario layer's `typedExemplar` be a shorter, cross-template version
