# R2 — Curated Per-Content-Type Example System

**Status**: PROPOSED
**Priority**: P2 (user-visible win on the highest-intent surface)
**Effort**: Framework ~1 week (1 engineer) + ~1 day/type for canonical prompts (agent-produced; parallelizable)
**Origin**: 2026-04-19 product-owner review — "under 'say it' we should have a very very very good and well curated example… per type we should have a deep research agent that comes up with that."

## 1. Problem

The central "Say it" input is the single highest-intent surface on the product — it is the mouth of the funnel for every creation (podcast, course, writeup, class, Car Mode, interview prep). Today it shows:

- **Home hero**: a typing rotation of **4 shallow, generic placeholders** — `"Prepare for executive leadership interviews..."`, `"Master the Google STAR method..."`, `"Engineer a curriculum from my resume..."`, `"Convert technical notes into professional audio..."` (`components/home/HeroSection.tsx:8-13`).
- **Create-smart "Say it" tab**: a **single static placeholder** — `"Tap the mic or type what you want. E.g. 'A horror story about vampires in 5 episodes' or 'Interview prep for a senior PM role at Google'"` (`components/smart-create/IntentSection.tsx:283`).

These are under-designed for a product whose differentiator is depth. A novice user reading "Master the Google STAR method..." learns nothing about what the platform can actually produce — they do not see that a prompt can become a 5-episode horror series, a 10-module onboarding course, a 30-minute Car Mode drill, or a branded writeup. The placeholder is a **lost education moment** at the point of highest attention.

## 2. Goal

Replace the generic placeholder rotation with a **curated, per-content-type example library** — one deeply exemplary prompt per type — and route type selection to the correct planner path. When the user has not picked a type, rotate **across types** (not within shallow variants) so every placeholder teaches a different capability.

## 3. Content-Type Taxonomy (current)

Inventoried from `lib/templates.ts` (34 templates across 8 personas) and routing surfaces. The system must ship with curated prompts for each **top-level content type**:

| Content type | Evidence |
|---|---|
| Interview Prep (tech, behavioral, case, 30-min mock) | `lib/templates.ts:61-126` + `/interview-prep/mock/text` route in `HeroSection.tsx:22` |
| Podcast — Topic Explainer / News Digest / Pep Talk / Team Update | `lib/templates.ts:129-191` |
| Study Audio — Exam Review / Concept Deep Dive / Flash Cards / Lecture Supplement | `lib/templates.ts:193-260` |
| Story Series — Horror / Comedy / Sci-Fi / Bedtime / Romance Drama | `lib/templates.ts:264-353` |
| K–12 Course (Elementary, Middle, High, Quick Review) | `lib/templates.ts:356-446` |
| University Course (Intro, Advanced, Graduate, Exam Review) | `lib/templates.ts:448-529` |
| Corporate Training (Onboarding, Compliance, Skills, Leadership) | `lib/templates.ts:531-612` |
| Self-Learning (Curious, Skill, Exam, Quick Overview) | `lib/templates.ts:614-692` |
| Writeup / Blog / Newsletter / Article | `lib/search-items.ts:33` |
| Classes (teacher-led) | `lib/search-items.ts:31` |
| Car Mode live session | product memory — Car Mode audio bridge, Q&A flow |

For the MVP framework we consolidate into **~10 top-level types**; sub-variants (horror vs. comedy) live within the type's curated bank.

## 4. Proposal

### 4.1 System shape (not content)

> **Scope note**: This proposal specifies the *system* — schema, storage, UI, routing. The canonical prompts themselves are produced by **per-type deep research agents** spawned during implementation (one agent per content type), not here. Each agent outputs 5–10 exemplary prompts following the schema below.

### 4.2 Data model — versioned TS constants

Add `lib/curated-examples.ts`:

```ts
export interface CuratedExample {
  id: string                    // "horror-vampires-5ep"
  contentType: ContentType      // "story-series" | "podcast" | "course" | ...
  subVariant?: string           // "horror" | "comedy" for story-series
  title: string                 // "Vampire horror, 5 episodes"
  goal: string                  // one-line user intent
  exemplaryPrompt: string       // 200–500 chars, the thing we type into the box
  outcomePreview: string        // "Becomes a 5-episode audio series with layered sound design, ~$0.08/ep"
  tags: string[]                // for search + chip routing
  routeOnSelect: string         // "/create-smart?template=horror-series&text=..."
  version: number               // bump on copy change; used for A/B
  researchedBy?: string         // "deep-research-horror-v1" — audit trail
  researchedAt?: string         // ISO date
}

export const CURATED_EXAMPLES: Record<ContentType, CuratedExample[]> = {
  'interview-prep': [/* 5–10 from interview-prep research agent */],
  'podcast': [/* 5–10 from podcast research agent */],
  // ... per type
}
```

**Why TS constants (not backend)**: copy iterates weekly; a backend round-trip per edit is overkill and adds failure surface. TS constants are reviewed via PR, shipped on next build, versioned in git, and don't require schema migrations. If we later need per-user A/B, we layer a feature flag over the constants — not the other way round.

### 4.3 UI — three affordances

1. **Bigger input area when type is selected.** Today the `/create-smart` textarea is `rows={3}` (`IntentSection.tsx:284`). When the user has a type pinned, auto-expand to `rows={6}` so the exemplary prompt (200–500 chars) is visible in one glance without scrolling. The home hero input (currently `px-6 py-5`, `HeroSection.tsx:113`) gets a second "expanded" mode triggered by a type chip click.
2. **One deep typing effect, not a rotation of shallow.** The typing animation (`HeroSection.tsx:37-51`) stays, but types **one** exemplary prompt per type cycle (180–220 chars shown, continues to full 500). Current 45ms/char + 3s pause (`HeroSection.tsx:42,15`) feels good; keep.
3. **Type selector routes to planner.** The existing `QUICK_CHIPS` (`HeroSection.tsx:21-26`) expands to 8–10 type chips. Selecting a chip: (a) pins the type above the input, (b) swaps the typing placeholder to that type's canonical prompt, (c) on Go routes to the matching planner — mirrors current `?template=` pattern (`HeroSection.tsx:172`).

### 4.4 Per-type deep research agents (implementation time)

When a content type is implemented, spawn a dedicated deep research agent whose job is narrow: **produce 5–10 canonical prompts for THIS type only**. Agent brief template:

- Study the 3 best examples of this content type on the platform (Firestore top-rated)
- Study 5 external reference exemplars (NYT mini-documentaries for news podcast, Serial for true-crime, MasterClass for course, etc.)
- Produce 5–10 prompts that each: (a) teach a distinct capability, (b) are specific enough to be obviously non-generic, (c) compile to the platform's planner output
- Include `outcomePreview` showing what the user gets (episode count, duration, cost, voice style)
- Output as `CuratedExample[]` ready to drop into `CURATED_EXAMPLES`

Agents are independent — horror research agent does not see course research agent's brief. This keeps each type's voice distinct.

## 5. Effort

- **Framework**: ~1 week (1 engineer) — schema + `curated-examples.ts` skeleton + UI rewire of `HeroSection` and `IntentSection` + typing-engine refactor + chip routing + tests.
- **Per content type**: ~1 day per type — deep research agent run + prompt curation + review + merge. 10 types = ~2 weeks of agent work, parallelizable across 2–3 days wall-clock.

## 6. Acceptance Criteria

1. `lib/curated-examples.ts` exists with the `CuratedExample` schema and at least one type populated (pilot: `story-series` → horror) before framework PR merges.
2. `HeroSection` placeholder rotation reads from `CURATED_EXAMPLES` — no literal strings in the component.
3. `IntentSection` "Say it" tab textarea auto-expands to `rows >= 6` when a type is pinned and collapses on clear.
4. Clicking a type chip pins the type badge above the input AND swaps the typing animation to that type's canonical prompt within 200ms.
5. Submitting with a pinned type routes to `/create-smart?template={id}&text=...` — matches existing routing contract (`HeroSection.tsx:172`).
6. Typing animation plays exactly ONE full prompt per type before cycling to the next type (no mid-sentence interruption on cycle).
7. Each `CuratedExample` has `outcomePreview` visible as subtitle below the input when its prompt is shown (teaches capability inline).
8. `researchedBy` and `researchedAt` populated for every example so we can audit which agent produced what, and re-run when stale.
9. Playwright E2E test on beta.kitesforu.com: load `/`, click each type chip, verify placeholder swaps and Go routes correctly.
10. Lighthouse: typing animation CPU cost < 2% on mid-tier mobile; no jank on type switch.

## 7. Out of Scope

- Backend API for serving examples (TS constants are deliberate — revisit only when we need per-user personalization).
- The canonical prompt **content** — produced by per-type research agents during implementation, not in this proposal.
- i18n of prompts (English-only MVP; i18n = follow-up once the system is proven).
- Dynamic LLM-generated examples at page load (cost + latency; canonical curated is the bar).
- Replacement of the `/create-smart` "I have notes" or "Starting points" tabs (`IntentSection.tsx:253-254`) — those are separate surfaces with their own UX contracts.

## 8. Open Questions

1. Should the home hero rotate through all 10 types, or only the top 3 by usage? (Telemetry needed.)
2. When a user lands with `?text=` from external deep-link, do we suppress the typing animation entirely, or briefly show then yield? Current code suppresses on any query (`HeroSection.tsx:55`) — preserve.
3. Do we expose `outcomePreview` as a live tooltip on hover, or always inline? Inline is clearer; hover is less noisy.

## 9. References

- `components/home/HeroSection.tsx:8-13` — current PLACEHOLDERS
- `components/home/HeroSection.tsx:21-26` — current QUICK_CHIPS (type chip prior art)
- `components/home/HeroSection.tsx:37-51` — typing effect engine to refactor
- `components/home/CentralChatLauncher.tsx` — new unified home entry (same typing engine)
- `components/smart-create/IntentSection.tsx:283` — current single static placeholder
- `components/smart-create/IntentSection.tsx:248-260` — TabToggle: "Say it" / "I have notes" / "Starting points"
- `lib/templates.ts:1-750` — 34 templates, 8 personas (source for content-type taxonomy)
- `lib/search-items.ts:24-40` — `classifyInput` keyword routing (interview/podcast/course/writeup/classes)
