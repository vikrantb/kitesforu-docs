# R2 — Scenario / Sub-Genre Guidance Calibration + Live Coverage Ticks

**Status**: PROPOSED
**Priority**: P1 (quality debt on a shipped feature — user-reported as misleading for ~5 of 9 scenarios)
**Effort**: 1.5 weeks (Phase 1 research 3–4 days, Phase 2 implementation 3–4 days, Phase 3 ticks 1–2 days)
**Affected repos**: kitesforu-docs (this), kitesforu-frontend (primary), kitesforu-workers (read-only)
**Origin**: User report 2026-04-21: "for things like horror/romance/comedy and all that the typing thing in the empty box is not correct. Also even for the others. Think through the proper example... first double check for each thing what do we really need to make the best thing. Work back from the final prompts and experience and topic etc and then [ensure] the examples are correct and appropriate."
**Builds on**: `done/R2-scenario-type-guidance.md` (shipped 9 scenarios, frontend PRs #490/#491)

---

## 1. The problem — what's wrong today

The shipped scenario-guidance (frontend PRs #490/#491) writes ONE `typedExemplar` per scenario id. Nine scenarios, nine exemplars. That's too coarse when the scenario subsumes a wide genre spread:

- **`storytelling`** (one exemplar: Shirley Jackson cozy-dread anthology in Crownpoint, Vermont) — this is literary horror with a narrator-unreliability angle. A user wanting **comedy** sees the horror exemplar and thinks "this tool doesn't understand my intent." A user wanting **bedtime stories** or **romance** or **sci-fi** hits the same mismatch. ~5 sub-genres underserved.
- **`explainer-podcast`** (one exemplar: Treasury basis trade Odd Lots-style) — works for business/finance, fails for **science explainer**, **health**, **history**, **true crime**, **technology**. ~5 sub-genres underserved.
- **`exam-prep`** (one exemplar: USMLE Step 1 biochem) — works for medical, fails for **bar**, **CFA**, **AP exams**, **standardized tests (GRE/GMAT)**, **tech certs (AWS/GCP)**.
- **`skill-mastery`** (one exemplar: Rust async Pin) — works for software engineering, fails for **cooking/craft**, **language learning**, **music theory**, **investing basics**.
- **`corporate-training`** (one exemplar: GDPR Art 17) — works for data-privacy compliance, fails for **cybersecurity awareness**, **DEI**, **manager onboarding**, **sales enablement**.

**Root cause**: scenarios are a coarse routing axis. Within each scenario, sub-genres are what actually control what the planner rewards. The planner/prompt system already consumes a `content_type` and a `genre` (see `kitesforu-workers/src/workers/prompts/content_types/*/`); the guidance UI ignored that axis.

**Secondary gap**: the 3–5 schemaHint chips below the box are STATIC — they sit there as a checklist, but nothing confirms whether the user's input (or the typed exemplar) covers them. The user has to visually cross-reference. A **live tick** on each chip as the typed example types past the phrase that satisfies it closes the teaching loop — users SEE the example hit each requirement, which teaches the requirement by demonstration.

---

## 2. The fix — reverse-engineered, not authored

**Core principle** (from user message): *work backward from the final prompt consumption*. For every sub-genre, the research brief starts with "what does the workers-side prompt ACTUALLY read from `custom_instructions` for this genre?" and derives the exemplar from there. NO top-down copy.

### 2.1 Phase 1 — per-sub-genre research

For each of the sub-genres listed below, a dedicated deep-research-agent produces a brief following the R2-scenario-type-guidance template. **The research is mechanical, not creative**:

1. Read the relevant workers prompt at `kitesforu-workers/src/workers/prompts/content_types/{genre}/{profile,outline_rules,dialogue_rules}.yaml`
2. List every variable the template substitutes from `custom_instructions` (grep `{variable_name}` placeholders)
3. Read the prompt composer's `ContentSectionBuilder` to see how `custom_instructions` enters the composed prompt
4. Identify the 3–5 fields the planner rewards MOST — the ones that, if missing, cause the planner to ask clarifying questions OR produce generic output
5. Pick ONE named real-world anchor per sub-genre (same quality bar as the existing 9 exemplars — Stripe L5, NGSS 5-PS1-1, MIT 6.006, etc.)
6. Draft a 400–800 char typedExemplar that demonstrates density across those 3–5 fields

**Full matrix of sub-genres to research**:

| Scenario | Sub-genre | Named anchor candidate (for research — NOT final) |
|---|---|---|
| storytelling | **horror-folk** | Shirley Jackson anthology (existing) — keep as the `horror` default |
| storytelling | **horror-modern** | Black Mirror tech-dystopia |
| storytelling | **comedy** | Sitcom pilot (Abbott Elementary / What We Do In The Shadows) |
| storytelling | **romance** | Hallmark holiday arc OR literary (Sally Rooney) |
| storytelling | **bedtime** | Goodnight Moon-style descending energy; Oliver Jeffers texture |
| storytelling | **sci-fi** | Octavia Butler near-future; The Expanse political |
| storytelling | **drama** | Mike White Social Dynamics (White Lotus, Enlightened) |
| storytelling | **mystery** | Golden-age whodunit (Christie) or noir (Chandler) |
| explainer-podcast | **business-deep-dive** | Odd Lots basis trade (existing) |
| explainer-podcast | **science** | Radiolab / Quanta Magazine — paper-anchored |
| explainer-podcast | **true-crime** | Serial Season 1 structural anchor |
| explainer-podcast | **history** | Hardcore History / The Rest Is History |
| explainer-podcast | **health** | Huberman Lab / Peter Attia structure |
| explainer-podcast | **technology** | Stratechery (existing podcast hero fallback) |
| exam-prep | **medical-step1** | USMLE Step 1 biochem (existing) |
| exam-prep | **bar** | Multistate MBE torts, UBE |
| exam-prep | **cfa** | CFA Level II item sets |
| exam-prep | **standardized** | GRE / GMAT — quant pattern recognition |
| exam-prep | **ap-high-school** | AP Chem equilibrium, AP Bio photosynthesis |
| exam-prep | **tech-cert** | AWS SAA-C03 / Google Cloud Architect |
| skill-mastery | **software-engineering** | Rust async Pin (existing) |
| skill-mastery | **language-learning** | Japanese kanji — CJK spacing + mnemonics |
| skill-mastery | **cooking-craft** | Knife skills / sourdough timing |
| skill-mastery | **music-theory** | Harmonic function, jazz chord substitutions |
| skill-mastery | **investing-personal-finance** | Options Greeks, mortgage amortization |
| corporate-training | **data-privacy** | GDPR Art 17 (existing) |
| corporate-training | **cybersecurity** | Phishing pattern recognition; NIST SP 800-53 |
| corporate-training | **dei** | Bystander intervention training |
| corporate-training | **manager-onboarding** | Crucial conversations framework |
| corporate-training | **sales-enablement** | MEDDIC qualification |

Other scenarios (**interview-prep**, **k12-lesson**, **university-lecture**, **writeup**) keep their existing single exemplar as the default — those sub-domains are narrow enough that one anchor works. The research pass SHOULD re-audit them but is unlikely to need multiple sub-genre entries.

**Deliverable per brief**: one TypeScript object literal matching the existing `ScenarioGuidance` shape, plus a `ticks` annotation (see Phase 3) marking which character offsets in `typedExemplar` cover each schemaHint.

### 2.2 Phase 2 — data shape + routing

**Current shape** (`kitesforu-frontend/lib/scenario-guidance.ts`):
```ts
export const SCENARIO_GUIDANCE: Partial<Record<ScenarioId, ScenarioGuidance>>
```

**New shape** — two layers:
```ts
export type SubGenreId = 'horror-folk' | 'horror-modern' | 'comedy' | 'romance' | /* ...30+ ... */

export interface ScenarioGuidanceBucket {
  default: ScenarioGuidance            // fallback when no sub-genre resolves
  subGenres?: Partial<Record<SubGenreId, ScenarioGuidance>>
}

export const SCENARIO_GUIDANCE: Partial<Record<ScenarioId, ScenarioGuidanceBucket>>
```

**Routing** — how does the UI pick the right sub-genre entry?

Three signals, in order of precedence:

1. **Template id** — if the user clicked a template (e.g. `horror-series`, `comedy-set`, `ap-chem`), the template already pins the sub-genre. Template-coaching already wins over scenario-guidance for this case, so this is the existing path.
2. **User-typed hint** — as the user types, run a lightweight keyword classifier against the sub-genre tags. "vampires" → horror, "comedy about dating" → comedy, "Radiolab" → science. Debounced ~300ms. Swap the active exemplar on classification change. Do NOT force a classification if confidence is low — fall back to the scenario's `default` entry.
3. **Sub-genre chips** — above the input, render a horizontal row of sub-genre chips (e.g. Horror / Comedy / Romance / Bedtime for storytelling). Click → lock the exemplar to that sub-genre until the user unlocks via another chip or clears the input.

Defaults:
- No chip clicked, no confident classification → show the `default` entry
- User can always clear by picking the active chip again (toggle off)

### 2.3 Phase 3 — live coverage ticks

**UX**: each schemaHint chip gets a small checkmark (✓) that appears when the typed exemplar passes the offset that "covers" that hint. Demonstrates BY EXAMPLE that a strong input hits each schemaHint.

**Mechanism**:
```ts
interface ScenarioGuidance {
  typedExemplar: string
  schemaHints: string[]
  // NEW: parallel to schemaHints — each entry is the char offset in
  // typedExemplar at which that hint becomes 'covered'. `null` = not
  // covered by the exemplar (rare — should be the exception).
  coverageOffsets?: (number | null)[]
  // ... existing fields
}
```

The deep-research-agent produces `coverageOffsets` at the same time as `schemaHints` — reads through the typedExemplar, marks where each hint is first demonstrated, records the char offset. Static, deterministic, no runtime classification.

**Animation integration**: `IntentSection.tsx` tracks `currentCharIdx` in the typewriter state (it already does via `coachedPlaceholder.length`). When `currentCharIdx >= coverageOffsets[i]`, the chip for `schemaHints[i]` flips to the ticked state. Smooth fade-in, 150ms.

**Edge cases**:
- User starts typing their own input before the animation completes → reset all ticks, revert to plain chips
- User clicks a sub-genre chip mid-animation → reset typewriter + ticks, restart for new sub-genre
- `coverageOffsets` missing on a legacy entry → fall back to static chips (backward-compat with the 9 already-shipped entries until Phase 1 re-audits them)

---

## 3. Implementation phases

**Phase 1 — research (3–4 days)**

- Spawn 1 deep-research-agent per sub-genre in the matrix (30+ briefs). Run in parallel batches of 4–6.
- Each brief follows the process from §2.1.
- Output: TypeScript object literals + coverageOffsets, one per sub-genre, pasted into a new module `lib/scenario-guidance/sub-genres/{scenario}.ts`.
- Also re-audit the 9 already-shipped scenarios — do their existing exemplars still pass the reverse-engineered bar? Mark any that don't for Phase 2 replacement.

**Phase 2 — data + routing (3–4 days)**

- Refactor `lib/scenario-guidance.ts` to the two-layer shape in §2.2.
- Implement the keyword classifier (`lib/scenario-guidance/classifier.ts`) — a small static trie of sub-genre → keyword lists, tested independently. Returns `{ subGenreId, confidence }`; confidence threshold 0.6 for auto-swap.
- Add sub-genre chip row to `IntentSection.tsx` above the textarea (below the tab toggle). Render only when the active scenario has `subGenres`.
- Wire template-coaching / scenario / sub-genre precedence in the existing `deriveScenario` helper.

**Phase 3 — live ticks (1–2 days)**

- Extend `ScenarioGuidance` type with `coverageOffsets`.
- New `SchemaHintChipTicks` component that reads the current typewriter offset and renders per-chip ticks.
- Fade-in animation with `prefers-reduced-motion` fallback.
- Unit tests: offset-reached flips tick; reset on user typing; reset on sub-genre swap.

---

## 4. Acceptance criteria

- [ ] Storytelling shows a different exemplar per sub-genre (horror / comedy / romance / bedtime / sci-fi / drama / mystery), each grounded in a named real anchor and reverse-engineered from the matching `kitesforu-workers/src/workers/prompts/content_types/{genre}/` consumption.
- [ ] Explainer-podcast shows a different exemplar per sub-genre (business / science / true-crime / history / health / technology).
- [ ] Exam-prep shows a different exemplar per sub-genre (medical / bar / CFA / standardized / AP / tech-cert).
- [ ] Skill-mastery shows a different exemplar per sub-genre (software / language-learning / cooking-craft / music / investing).
- [ ] Corporate-training shows a different exemplar per sub-genre (data-privacy / cybersecurity / DEI / manager-onboarding / sales-enablement).
- [ ] Sub-genre row appears above the textarea when the active scenario has sub-genres; clicking a chip swaps the exemplar.
- [ ] Lightweight keyword classifier swaps the exemplar when the user's typing confidently matches a sub-genre.
- [ ] Each schemaHint chip shows a live ✓ tick as the typewriter passes the offset at which that hint is covered by the exemplar.
- [ ] Ticks reset cleanly when the user (a) starts typing their own input, (b) swaps sub-genre, (c) clears the input.
- [ ] Legacy entries without `coverageOffsets` render as static chips (no crash, no ticks).
- [ ] Per-sub-genre research briefs (30+) live in the git history as commit notes or linked to a tracking issue, so future auditors can see the "work backward from the prompt" trail.

---

## 5. Risks + mitigations

- **Copy drift** — 30+ per-sub-genre exemplars is a lot to keep fresh. Mitigation: each entry has `researchedBy` + `researchedAt` + `version` metadata (same as today's scenario-guidance entries). Quarterly staleness check.
- **Classifier false positives** — "vampires" in a comedy pitch ("vampires trying to get a mortgage") might misclassify as horror. Mitigation: confidence threshold + user-initiated override via the chip row.
- **Tick-fire noise** — if the exemplar types fast (45ms/char) through a coverage offset, the chip ticks faster than the user can notice. Mitigation: small pause (~300ms) after a chip ticks before continuing the typewriter, creating a perceptible rhythm.
- **Bundle size** — 30+ exemplar strings + coverage data is ~50KB gzipped. Mitigation: lazy-load per scenario via dynamic `import()` on scenario resolution, matching the existing pattern for RichComposer.

---

## 6. Non-goals

- **LLM-based classification** of the user's typed input — too heavy for a typing-animation debounce window. Keep it to static keyword tries.
- **Full content-type taxonomy unification with kitesforu-workers** — the workers repo has its own 19-genre content-types directory; this proposal uses a SUBSET tailored to what users typically type. Future consolidation is out of scope.
- **Tick-based scoring** — we show ticks as coverage confirmation, not as a scored "quality gate." The user always controls submit; ticks are feedback, not gatekeeping.

---

## 7. Open questions (for reviewer)

- Should the **sub-genre chips row be optional** behind a feature flag for phased rollout, or replace the current behavior wholesale? Recommendation: flag (`feature_sub_genre_chips_live`) default ON after 50% of sub-genres are populated.
- Does the **classifier run on every keystroke or only after typing pauses**? Recommendation: debounce 300ms to match the existing library-search pattern.
- For scenarios that may not need sub-genres (**interview-prep, k12-lesson, university-lecture, writeup**), should Phase 1 still re-audit them against the reverse-engineered bar, or only revisit if they're reported as miscalibrated? Recommendation: re-audit as part of the same pass; it's cheap and the agent is already primed.
