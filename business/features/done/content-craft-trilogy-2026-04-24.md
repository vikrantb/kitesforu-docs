# Content-Craft Trilogy: news + sports + tech_review (2026-04-24)

**Status**: SHIPPED across 3 stacked PRs.
**Shipped via**: workers #355 (news) → workers #356 (sports) →
workers #357 (tech_review). Each PR stacked on the previous.

## What this closes

The final three underserved content-types called out in the
2026-04-23/24 content-craft arc memory entry as "remaining
underserved content-types with real user audience: fantasy,
tech_review, sports, news." Fantasy shipped in workers #332.
This ship closes the remaining three.

Before these PRs, `content_types/news/`, `content_types/sports/`,
and `content_types/tech_review/` each had only `profile.yaml`
stubs — meaning every news, sports, or tech_review audio job
fell back to generic prompts with no genre-specific craft
discipline. After these PRs, each genre has both `narration_rules.yaml`
and `dialogue_rules.yaml` files plus a content-type-gated quality
gate check + regen wire.

## Shared architecture

Each of the three genres ships on the exact pattern the
2026-04-23/24 arc established (proven 10× across bedtime /
horror / romance / comedy / mystery / sci-fi / true-crime /
documentary / motivational / fantasy):

1. **YAML-first craft definition** — `narration_rules.yaml` +
   `dialogue_rules.yaml` authored offline, loaded by the
   existing composer bridge, auto-verified by the existing
   152-case loader sweep test.
2. **Content-type-gated quality gate** — single check function
   dispatched from `run_quality_gate` only when
   `content_type == <genre>`. Returns a metrics dict for the
   Firestore debug page.
3. **Regen wire** — `regen_policy.should_regenerate` picks up
   the metric, `build_regen_hints` turns findings into a
   targeted imperative hint for the retry.

Zero new inference calls, zero new persistence, zero new
services. YAML authored offline; gate + regen run
post-generation on already-produced scripts.

## What each PR ships

### workers #355 — news

- `content_types/news/narration_rules.yaml`:
  narrator_identity + narrative_structure (capsule inverted
  pyramid + feature lead/nut-graf/narrative/close) +
  attribution_discipline_guidance (the load-bearing trust
  lever) + hedging_guidance (five-register ladder:
  confirmed / reported / alleged / unconfirmed / speculation) +
  fact_analysis_opinion_axis + banned_patterns
  (hype-tells / stale attribution / manufactured-urgency /
  false-balance / unearned closers) + corrections_discipline +
  specificity_guidance + expression_guidance.
- `content_types/news/dialogue_rules.yaml`:
  reporter + clarifier dynamic (banned: cable-news reactor) +
  opening rules + handoff patterns + attribution-inside-dialogue
  discipline + pacing rules + banned patterns +
  expression_guidance.
- `_check_news_attribution_discipline`: three signals in one
  check (attribution density, cable-news hype-tells, stale
  attribution forms) so a single retry can address all three
  failure modes.

Working references: NYT The Daily, NPR Up First, Axios Today,
Reuters, BBC World Service, Post Reports.

### workers #356 — sports

- `content_types/sports/narration_rules.yaml`:
  narrator_identity + narrative_structure (game recap + career
  feature + live-adjacent shapes) +
  specific_moment_anchoring_guidance (THE load-bearing lever:
  player + time + play + film-or-stat) +
  statistical_honesty_guidance (opponent quality / sample size /
  pace / availability / garbage-time context flags) +
  narrative_overlay_discipline (steelman rule) +
  banned_patterns (cable-sports hype / AI-sports tells /
  narrative-overlay-as-analysis / manufactured-controversy
  labels) + expression_guidance.
- `content_types/sports/dialogue_rules.yaml`:
  analyst + clarifier dynamic (banned: cable-sports reactor) +
  specific-moment-anchoring-inside-dialogue +
  statistical-honesty-inside-dialogue (clarifier MUST ask for
  context flags) + handoff patterns + pacing rules + banned
  patterns.
- `_check_sports_performance_tells`: banned-phrase check
  (`clutch gene`, `wanted it more`, `hot take` etc.). Regen
  hint prescribes specific-moment anchor antidote.

Working references: The Lowe Post (Zach Lowe), The Ringer NBA
Show, The Athletic film-breakdown podcasts, Pardon My Take at
its sharpest, Slow Burn sports-adjacent narrative series.

### workers #357 — tech_review

- `content_types/tech_review/narration_rules.yaml`:
  narrator_identity + narrative_structure (capsule + long-form
  review arc from single-use anchor through buy-condition
  verdict) + spec_vs_experience_guidance (THE load-bearing
  craft lever) + benchmark_honesty_guidance (tool + score +
  comparison + variance) + comparative_framing_discipline
  (one-closest-competitor + steelman) +
  user_segmentation_guidance (audience + counter-audience +
  upgrade-from segment + cross-shop segment) +
  banned_patterns + expression_guidance.
- `content_types/tech_review/dialogue_rules.yaml`:
  primary-reviewer + devil's-advocate dynamic (banned:
  reviewer + hype-partner) +
  spec-vs-experience-inside-dialogue +
  benchmark-honesty-inside-dialogue + handoff patterns + pacing.
- `_check_tech_review_hype_tells`: banned-phrase check
  (hype: `game-changing`, `revolutionary`, `blazingly fast`;
  unsupported-absolute: `nothing else like it`, `it just works`,
  `unparalleled`; performance-hype: `buttery smooth`,
  `absolutely flies`). Regen hint prescribes
  spec-vs-experience antidote.

Working references: Waveform (MKBHD), Accidental Tech Podcast,
The Vergecast, Upgrade, Hard Fork, The Ringer's Tech Podcast.

## Tests

- workers #355: 20/20 new `test_quality_gate_news_attribution.py`
- workers #356: 18/18 new `test_quality_gate_sports_performance_tells.py`
- workers #357: 18/18 new `test_quality_gate_tech_review_hype_tells.py`
- Existing 152-case `test_content_types_loader_coverage.py`
  sweep picks up every new YAML section automatically for all
  three genres.
- Broader quality-gate + regen + content-types cluster:
  448/448 pass after all three PRs.

## Unit economics

Zero new inference calls. Zero new persistence. Zero new
services. All three genres use the existing composer bridge,
the existing quality gate, and the existing regen policy
machinery. No operator flag flip required — the craft rules
ship live on merge.

## What this leaves open

- **Operator-flag activation** of the 2026-04-23/24 arc's
  prompt-caching + episode_type-classifier remains pending per
  `activation_runbook_2026_04_24.md`.
- **Mystery / scifi / fantasy gate checks needing LLM-as-judge**
  still pending an explicit cost-benefit decision (directive
  violates "no extra per-request inference" rule).
- **End-to-end content-quality measurement** — still a
  multi-session thread; tonight's trilogy extends the rules-
  based-gate surface area but doesn't change the "did the gate
  fix the violation vs did the listener prefer it" measurement
  gap.

## Post-merge verification (pending operator / beta access)

Per the `kforu-workers-engineer` runbook and CLAUDE.md rule 11
(Playwright verification after deploy):

- [ ] Create a test news job on beta with a script seeded with
  hype-tells + stale attribution; confirm the gate fires and
  regen hint surfaces in the Firestore debug page.
- [ ] Create a test sports job with `clutch gene` / `hot take`
  seeded; confirm gate + hint.
- [ ] Create a test tech_review job with `game-changing` /
  `it just works` seeded; confirm gate + hint.
- [ ] For each, verify the regenerated attempt produces
  a script that avoids the seeded violations.
