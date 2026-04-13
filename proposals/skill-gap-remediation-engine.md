# Skill-Gap Remediation Engine

**Status**: Proposal  
**Date**: 2026-04-12  
**Scope**: hot-path remediation between mock interview turns

## Thesis

When the intelligence loop detects a specific gap, KitesForU should not just say "fix this next time."

It should immediately do one of two things:

1. inject a **1-minute micro-drill** that repairs the exact weakness fast
2. surface a **Targeted Concept Card** that gives the user the missing mental model before the next question

The system should feel like this:

- weak answer lands
- system diagnoses the exact miss
- user gets a short, high-signal remediation
- next question tests the repaired skill immediately

That creates a closed learning loop instead of a scoring loop.

## Product Principle

Remediation must obey four constraints:

- **instant**: hot-path artifact should appear inside the current turn transition
- **small**: one gap, one drill, one concept, not a lesson
- **high-stakes**: examples must feel interview-real, not classroom-generic
- **causal**: the next question must test the repaired skill, or the drill has no teeth

## The Two Remediation Artifacts

### A. 1-minute micro-drill

Use when the user already knows the domain but missed an execution habit.

Best for:

- `quantified_impact`
- `star.result`
- `ownership_signal`
- `story_structure`
- `confidence_stability`

The drill is an action, not content explanation.

Examples:

- "Add the missing metric"
- "Rewrite only the Result sentence"
- "Turn vague outcome into before/after business impact"
- "Split Situation from Action in one sentence each"

### B. Targeted Concept Card

Use when the miss implies a missing mental model, not just a weak answer habit.

Best for:

- `role_calibration`
- `ambiguity_handling`
- `tradeoff_reasoning`
- `domain_depth`
- `system_design.capacity_estimation`

The card is a compact teaching object:

- what the concept is
- why interviewers care
- what a strong answer demonstrates
- one high-stakes example question

This extends the current `ConceptCard` system instead of inventing a second teaching format.

## Remediation Decision Policy

After hot-path turn analysis computes the `gap_vector`, run a remediation router.

```ts
type RemediationDecision = {
  artifact_type: "micro_drill" | "concept_card" | "none"
  target_skill_key: string
  urgency: number
  confidence: number
  should_block_next_question: boolean
  rationale: string
}
```

### Routing rules

Choose `micro_drill` when:

- the gap maps directly to an answer habit
- the user has previously demonstrated the concept elsewhere
- confidence in diagnosis is high
- the repair can be practiced in 30-60 seconds

Choose `concept_card` when:

- the gap suggests missing conceptual framing
- the same weakness appears across multiple question shapes
- the user is role-miscalibrated
- the next question would otherwise be unfair

Choose `none` when:

- the miss is too ambiguous
- the session is already under high pressure
- the next question itself is the better probe

## Hot-Path Architecture

Do not generate remediation from scratch every turn. That will drift generic and blow latency.

Use a three-layer system:

### Layer 1: deterministic template registry

Maintain a curated registry keyed by `skill_key`.

```ts
type RemediationTemplate = {
  skill_key: string
  artifact_type: "micro_drill" | "concept_card"
  prompt_style: string
  required_inputs: string[]
  default_timebox_sec: number
  evaluation_target: string
}
```

Examples:

- `behavioral.quantified_impact.micro`
- `behavioral.star.result.micro`
- `system_design.tradeoff_reasoning.card`
- `role.staff.level_calibration.card`

This gives you instant structure and consistent quality.

### Layer 2: slot-filling generator

The hot path only fills the template with live context:

- user’s weak sentence or rewrite
- role and seniority
- question focus area
- JD/company context if already cached
- prior strongest skill, so feedback stays motivating

This can be done with:

- deterministic string assembly for the simplest drills
- a small LLM call for polished phrasing when needed

### Layer 3: next-question binding

Every remediation artifact must emit:

```ts
type RemediationBinding = {
  target_skill_key: string
  next_question_policy_bias: {
    support_mode: "scaffold" | "balanced"
    preferred_skill_tags: string[]
    preferred_question_shape: Record<string, number>
    expires_after_turns: number
  }
}
```

This is how the system proves the drill mattered.

## Micro-Drill Format

Keep the object brutally small.

```python
class MicroDrill(BaseModel):
    skill_key: str
    title: str
    why_now: str
    instruction: str
    prompt: str
    ideal_answer_shape: list[str]
    timebox_sec: int = 60
    success_check: str
    next_question_bias: dict = Field(default_factory=dict)
```

### Recommended shape

- `title`: "Quantify The Result"
- `why_now`: "Your story had action but no measurable business outcome."
- `instruction`: one sentence
- `prompt`: a rewrite task, not an essay prompt
- `ideal_answer_shape`: 2-4 bullets
- `success_check`: the one thing we are looking for

### Example: Quantifying Impact

```json
{
  "skill_key": "behavioral.quantified_impact",
  "title": "Quantify The Result",
  "why_now": "Your answer explained what you did, but the interviewer still cannot tell if it mattered.",
  "instruction": "Rewrite only the Result portion in 2 sentences using baseline -> action effect -> business outcome.",
  "prompt": "Complete: 'Before this work, ___. After my change, ___. This mattered because ___.'",
  "ideal_answer_shape": [
    "baseline metric or pain",
    "measured change",
    "business or user impact"
  ],
  "timebox_sec": 45,
  "success_check": "Must include at least one concrete metric or bounded estimate."
}
```

### Example: STAR Result section

```json
{
  "skill_key": "behavioral.star.result",
  "title": "Land The Result",
  "why_now": "Your answer stopped at the action. Interviewers score the outcome.",
  "instruction": "Say the final Result out loud in one sentence. No setup, no action recap.",
  "prompt": "Finish this sentence: 'The result was ___, which proved ___.'",
  "ideal_answer_shape": [
    "explicit outcome",
    "evidence it worked",
    "why it mattered"
  ],
  "timebox_sec": 30,
  "success_check": "Result must be outcome-oriented, not action-oriented."
}
```

## Targeted Concept Card Format

Extend the existing `ConceptCard` contract with remediation-specific fields instead of creating a separate incompatible model.

```python
class TargetedConceptCard(BaseModel):
    skill_key: str
    title: str
    one_liner: str
    why_this_gap: str
    interviewer_signal: str
    high_stakes_example: str
    repair_heuristic: str
    anti_pattern: str
    next_question_bias: dict = Field(default_factory=dict)
```

### Example: Staff-level calibration

- `title`: "Org-Scale Ownership"
- `one_liner`: "At staff level, interviewers expect system and org leverage, not just strong individual execution."
- `why_this_gap`: "Your answer described solid IC work, but not cross-team influence or durable leverage."
- `interviewer_signal`: "Can this person shape systems, standards, and decisions beyond their own task?"
- `high_stakes_example`: "Tell me about a time you changed an engineering direction across multiple teams."
- `repair_heuristic`: "Frame scope as decision surface -> stakeholders -> leverage -> durable change."
- `anti_pattern`: "Listing only what you coded yourself."

## How To Keep Drills High-Stakes But Short

Short does not mean generic.

The system should compress **stakes**, not inflate explanation.

### High-stakes compression formula

Every drill/card should anchor on:

- real scope
- measurable consequence
- interviewer judgment

Template:

`"Because you are interviewing for {role}/{seniority}, the interviewer is testing whether you can {signal}. Your last answer missed {gap}. Repair it by {micro_action}."`

That gives urgency without needing a long lesson.

## Using Agentic Research Without Slowing The Hot Path

The deep layer should pre-enrich the remediation system, not generate every drill on demand.

### What agentic research produces

For each role/JD/company fingerprint, generate and cache:

- expected interviewer signals per key skill
- role-specific anti-patterns
- high-stakes example stems
- acceptable proxy metrics when exact numbers are unavailable
- follow-up probes that expose the gap fast

### Cache object

```python
class RemediationResearchCache(BaseModel):
    cache_key: str  # role + seniority + focus_area + jd_hash + company_hash
    skill_profiles: dict[str, dict] = Field(default_factory=dict)
    generated_at: datetime
    expires_at: datetime
    source_analysis_ids: list[str] = Field(default_factory=list)
```

For `behavioral.quantified_impact`, cache:

- metrics interviewers expect for this role family
- examples of good business/user/engineering impact framing
- "acceptable estimate" phrasing when user lacks exact numbers

For `system_design.tradeoff_reasoning`, cache:

- common tradeoff axes for the stack/domain
- strong answer shapes
- likely follow-up traps

### Runtime use

Hot path:

- read cached skill profile
- fill the template
- return the drill/card

Async:

- refresh caches when role/JD changes or uncertainty is high

This is how you keep the artifacts sharp without paying research latency every turn.

## Research Triggers

Use agentic research when:

- new role or JD fingerprint appears
- gap is repeated 2-3 times and still unresolved
- high-value session: senior/staff, enterprise, company-specific prep
- cached profile confidence is low

Do not use it when:

- the gap is mechanical and template-covered
- user is in a rapid-fire practice loop
- the artifact only needs the existing judge rewrite

## Remediation Ladder

Not every gap deserves the same intervention.

### Ladder

1. `nudge`
   - current `CoachFeedback.micro_drill`
   - 1 sentence

2. `micro_drill`
   - 30-60 second action
   - one-field JSON object

3. `targeted_concept_card`
   - 20-40 second read
   - one mental model

4. `drill_then_probe`
   - remediation + immediately biased next question

5. `deep_repair`
   - only when repeated failure persists
   - uses agentic-research-enriched concept and follow-up stack

The hot path should usually stay in levels 1-3.

## UX Sequence

Ideal between-turn sequence:

1. user answers
2. judge scores
3. intelligence loop detects dominant gap
4. system chooses artifact
5. UI shows:
   - top strength
   - one remediation artifact
   - explicit timebox
6. next question arrives biased toward the repaired skill

### Example

Feedback card:

- `Strength`: "Good ownership and action clarity."
- `Micro-drill`: "Quantify The Result"
- `Timebox`: "45 sec"
- `Prompt`: "Before this work, ___. After my change, ___. This mattered because ___."
- `Next up`: "The next question will test whether you can land impact clearly."

That creates anticipation and coherence.

## API And Data Model

### Route response changes

Extend `SubmitAnswerResponse` so either assessment or practice mode can optionally include remediation.

```python
class TurnRemediation(BaseModel):
    artifact_type: Literal["micro_drill", "concept_card"]
    target_skill_key: str
    urgency: float
    confidence: float
    micro_drill: Optional[dict] = None
    concept_card: Optional[dict] = None
```

Add to `SubmitAnswerResponse`:

```python
remediation: Optional[TurnRemediation] = None
```

### Persistence

Store the remediation artifact in the `MasteryTraceEvent` proposed in the intelligence-loop doc:

- chosen artifact
- target skill
- whether user completed it
- whether next turn improved

This lets you measure remediation efficacy by skill.

## Measuring If The Engine Works

Success is not "users clicked the drill."

Success is:

- next-turn improvement on targeted skill
- reduced repeat-gap frequency
- shorter time-to-repair for common weaknesses
- higher session continuation after a bad turn

### Core metrics

- `repair_rate_1`: targeted skill improves on next turn
- `repair_rate_3`: targeted skill improves within next 3 turns
- `repeat_gap_decay`: probability same gap recurs after remediation
- `session_salvage_rate`: sessions that continue after low-scoring turn
- `artifact_latency_p95`
- `artifact_accept_rate`

### Per-skill dashboard

Track by `skill_key`:

- drill frequency
- repair success
- median latency
- template vs LLM-generated performance
- research-enriched vs plain performance

## Recommended Build Order

1. Formalize `skill_key` taxonomy
2. Add deterministic micro-drill registry for 10-15 highest-frequency gaps
3. Extend `CoachFeedback` / `SubmitAnswerResponse` with `remediation`
4. Add next-question policy binding so drills actually influence the next turn
5. Extend concept generation to targeted remediation cards
6. Add agentic-research cache for role/JD/company skill profiles
7. Measure repair rates and prune weak templates

## Immediate Practical Reuse

You already have two useful starting points:

- `JudgeResult.top_fix` and `rewrite`
- `CoachFeedback.micro_drill`

Version 1 can be:

- replace the current generic micro-drill string with structured `MicroDrill`
- generate `TargetedConceptCard` by adapting the existing Teach-phase concept generator
- bind the next selector to the targeted skill

That gets you live remediation fast without rewriting the full loop first.

## Bottom Line

The remediation engine should behave like a compact tutor:

- diagnose one miss
- prescribe one short repair
- test that repair immediately

Agentic research is not there to make drills longer. It is there to make them **role-real, JD-aware, and interview-sharp** while the hot path stays fast.

That is the right shape: short in UX, deep in infrastructure.
