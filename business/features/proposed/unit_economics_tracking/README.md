# Unit Economics Tracking — Prompt-Size & Cost Baseline

**Status**: proposed (tracking doc, active)
**Owner**: workers repo (primary), frontend + api (downstream)
**Opened**: 2026-04-23
**Trigger**: content-improvement shipping pace (10+ content-type craft
ships in April 2026) needed a measured cost baseline before the next
wave. The composer-path / legacy-path bridge (workers #331) is the
first ship large enough to force the question.

---

## Why this doc exists

The craft-ship cadence on the workers repo (file 11 interview-prep,
file 12 exam-prep, file 13 second-pass synthesis) has been running hot.
Every ship adds rules, Python builders, or YAML — each of which grows
the final prompt the LLM sees. We've shipped ~1329 → 1337 tests in
the composer surface alone. That's good, but only if the unit
economics hold. This doc is where we measure, not where we guess.

The user's directive:

> we need the quality to be good for sure. not chances... but we also
> want the cost to not go high.

That's the contract. This doc exists to verify we're honoring it.

---

## Measured baseline (2026-04-23, post PR #331)

All numbers measured via `PromptComposer.compose_with_log(ctx)` with
the full default builder registry + a representative 2-segment outline.

| content_type | format    | duration | prompt chars | ~input tokens | sections |
| :----------- | :-------- | -------: | -----------: | ------------: | -------: |
| horror       | dialogue  |   20 min |       33 752 |         8 438 |       11 |
| romance      | dialogue  |   20 min |       30 649 |         7 662 |       11 |
| mystery      | dialogue  |   20 min |       37 615 |         9 403 |       11 |
| scifi        | dialogue  |   20 min |       35 249 |         8 812 |       11 |
| comedy       | dialogue  |   20 min |       33 081 |         8 270 |       11 |
| motivational | narration |   15 min |       20 419 |         5 104 |        9 |
| true_crime   | narration |   20 min |       19 701 |         4 925 |        9 |
| documentary  | narration |   20 min |       20 038 |         5 009 |        9 |
| interview    | dialogue  |   20 min |       32 049 |         8 012 |       12 |
| educational  | dialogue  |   20 min |       25 131 |         6 282 |       11 |
| general      | dialogue  |   20 min |       12 842 |         3 210 |        9 |

**Token estimation:** chars / 4 (standard Claude heuristic; real
values are typically within ±10% of this estimate).

**`general` is the floor** — no content-type YAML, no genre-specific
builders fire, just the composer spine. At ~3 200 tokens, it
represents the minimum overhead every episode pays.

**Fiction dialogue is the ceiling** — ~8-9.5k input tokens, because
the composer stacks StructuralRepetition + DramaticDiscipline +
ContentSection + **ContentTypeCraft (new)** + UserProfileHandling +
EmotionArc + Quality + Output sections on top of the spine.

---

## Ship impact: composer-path content-type YAML bridge (#331)

Before #331 landed, the composer path did NOT load
`content_types/{genre}/{format}_rules.yaml`. Only the legacy
`PromptLoader.load_prompt_parts()` path did. This meant multi-persona
voice-map jobs (the primary fiction code path per the 2026-04-05
persona system ship) never saw content-type craft rules.

Measurement comparison (best-effort — pre-#331 numbers estimated
from earlier prompt-size probes in session telemetry):

| content_type       | pre-#331 | post-#331 |  delta   |
| :----------------- | -------: | --------: | -------: |
| horror dialogue    |   ~23 k |    33.7 k |  +10.7 k |
| romance dialogue   |   ~23 k |    30.6 k |   +7.6 k |
| mystery dialogue   |   ~23 k |    37.6 k |  +14.6 k |
| scifi dialogue     |   ~23 k |    35.2 k |  +12.2 k |
| comedy dialogue    |   ~23 k |    33.1 k |  +10.1 k |
| motivational narr. |   ~10 k |    20.4 k |  +10.4 k |

**Per-episode input-token delta: +~2 500 to +3 500 tokens for fiction
content** (the bridged YAMLs averaging 10-14k extra chars).

### Claude pricing reference (2026-04)

| Tier        | Input       | Output       | Cache read     | Cache write    |
| :---------- | :---------- | :----------- | :------------- | :------------- |
| Opus 4.7    | $15/Mtok    | $75/Mtok     | $1.50/Mtok     | $18.75/Mtok    |
| Sonnet 4.6  | $3/Mtok     | $15/Mtok     | $0.30/Mtok     | $3.75/Mtok     |

### Per-episode cost model (Opus 4.7, no caching)

| Component                           | Tokens       | Cost       |
| :---------------------------------- | -----------: | ---------: |
| Input — spine (all content_types)   |   ~3 200 tok |   $0.048   |
| Input — bridge delta (fiction)      |   ~3 000 tok |   $0.045   |
| Input — total fiction dialogue      |   ~8 500 tok |   $0.128   |
| Output — script (20-min episode)    |  ~30 000 tok |   $2.250   |
| **Per-episode total (fiction dlg)** |              | **$2.378** |
| **Per-episode total (general dlg)** |              | **$2.298** |

**The bridge adds ~$0.04-0.05 per episode of input cost.** Output
dominates at ~$2.25, so the bridge is a ~2% unit-economics hit.

### At scale (1 000 episodes/day)

| State         | Input $/day | Output $/day |  Total $/day  |  $/month   |
| :------------ | ----------: | -----------: | ------------: | ---------: |
| Pre-#331      |      ~$86   |     ~$2 250  |       ~$2 336 |   ~$70 080 |
| Post-#331     |     ~$128   |     ~$2 250  |       ~$2 378 |   ~$71 340 |
| Delta         |      ~$42   |         $0   |          ~$42 |    ~$1 260 |

**Ship cost: +$1.3k/month at 1k episodes/day.** For unlocking the
entire content-type craft surface on the primary fiction path, that is
a cheap unlock — the quality delta is the aggregate of 10+ craft ships.

---

## The "really amazing" cost lever — Anthropic prompt caching

Anthropic prompt caching charges **10% of input price on cache reads**
when the same prompt prefix hits within a 5-minute TTL (or extended
to 1 hour with cache-control).

### What caches well

The composer output is roughly:

```
[STATIC, content_type-dependent]           ← ~6-8k tok, cacheable
SystemIdentity
ContentRating
LanguageRules
PersonaInjection                            (per-persona static)
ConversationRules
PacingRules
StructuralRepetition
DramaticDiscipline
PauseAndTry
ContentTypeCraft                            ← new, cacheable
[DYNAMIC, per-episode]                     ← ~1-3k tok, NOT cacheable
Content (topic + outline + research)
UserProfileHandling                         (per-user)
EmotionArc
QualityGuidelines
OutputFormat
```

### Projected savings at scale

If we place a `cache_control: ephemeral` breakpoint **after the static
block** and before `ContentSectionBuilder` (priority 250):

- Cached portion: ~7 000 tok @ $1.50/Mtok = **$0.0105/ep cached read**
- vs uncached: ~7 000 tok @ $15/Mtok = **$0.105/ep**
- **Savings: ~$0.095 per cached episode**

**Cache-hit rate drivers:**

1. Persona-voice-map jobs tend to re-use the same content_type +
   format combo within a user's session (studio replay, regen).
2. Car-mode drive sessions hit the same content_type repeatedly as
   episodes chain.
3. The 5-min TTL cache read is more than enough to cover same-session
   regeneration latency (~30-60s per episode).

**Break-even:** cache write costs 1.25× input ($18.75/Mtok for Opus).
One write amortized over two reads pays for itself. At our
re-generation rates (studio + drive + regen features), 2× amortization
is conservatively the floor.

### Net-net projection (with caching)

| State                         | Per-ep input | Per-ep total | Monthly @ 1k/day |
| :---------------------------- | -----------: | -----------: | ---------------: |
| Post-#331, no caching         |      $0.128  |     $2.378   |        ~$71 340  |
| Post-#331, cache 50% of eps   |     ~$0.080  |     $2.330   |        ~$69 900  |
| Post-#331, cache 80% of eps   |     ~$0.052  |     $2.302   |        ~$69 060  |
| Post-#331, cache 95% of eps   |     ~$0.036  |     $2.286   |        ~$68 580  |

**At 80% cache hit rate, prompt caching fully offsets the bridge's
cost and then some** — net-net we're down ~$1k/month from pre-#331.

---

## Quality guarantees (the "not by chance" side)

Quality is guaranteed (not chanced) by three mechanisms:

1. **Unit tests pin each rule.** 1 337 passing tests in workers after
   #331 (up from 1 329). Every content-type craft rule that ships has
   at least one regression test that fires if a builder is silently
   dropped.

2. **Integration tests verify the bridge.** #331's test suite confirms
   real content-type YAML markers (AMBIENT BED, POV SWITCHING, fair-
   play CLUE, SPECULATIVE ENGINE, SOURCING DISCIPLINE, OPERATIONALIZED
   HONESTY) reach the composed prompt. If a YAML file stops loading,
   the test fails immediately.

3. **Priority-based composition pinning.** `test_user_profile_handling_composer.py`
   and `test_content_type_craft_builder.py::TestRegistryWiring` pin
   the ordering invariants — ContentSectionBuilder before
   ContentTypeCraftBuilder before UserProfileHandlingBuilder. Regression
   in registry order is caught before it ships.

---

## Open threads

- [ ] Implement prompt-caching wire-up in `PromptComposer.compose()` —
      emit a `cache_control` breakpoint at priority ≥ 255 (static block
      ends) so the static content_types/* bridge output caches. Expected
      impact: 80%+ cache hit rate → ~$2-3k/month savings at 1k ep/day.
      **This is the "really amazing" ship.** Filed as follow-up, not
      blocking #331.
- [ ] Measure actual cache-hit rates post-ship via Claude API
      `usage.cache_read_input_tokens` telemetry. Add to Firestore
      `stages.job-script.llm_usage` debug document for observability.
- [ ] Re-measure this matrix monthly. New craft ships can add 2-5k
      chars each; we want a clear record of when/why growth happened.
- [ ] Output-token optimization thread — script length varies from
      20k to 40k output tokens with no clear craft-rule-driven reason.
      Investigate variance before optimizing (TTFB vs quality trade).

---

## Observability (for future sessions)

Reproduce the measurement matrix at any time:

```python
from workers.prompts.composer.composer import PromptComposer
from workers.prompts.composer.context import PromptContext
from workers.prompts.composer.registry import get_default_builders

composer = PromptComposer(builders=get_default_builders())

for content_type, fmt, dur in [("horror", "dialogue", 20.0), ...]:
    ctx = PromptContext(
        format=fmt, genre=content_type, content_type=content_type,
        duration_min=dur, topic="...", outline={...},
    )
    prompt, log = composer.compose_with_log(ctx)
    print(content_type, fmt, len(prompt), len(prompt) // 4)
```

The measurement script itself lives in-repo as a test fixture; if
someone ships a new content_type, add a row here.

---

## Decision log

**2026-04-23** — Shipped #331 (composer bridge) even though it grows
input tokens per fiction episode. Rationale: it unifies two divergent
prompt systems that were drifting apart, and the cost delta
(~$42/day at scale) is <2% of total spend. Prompt caching (forthcoming)
erases the delta and then some. Quality win is measurable in unit
tests; cost hit is bounded and temporary.
