# Hyper-Personalized Intelligence Loop — Corrected Scalable Plan

**Status**: Post-audit redesign  
**Date**: 2026-04-13  
**Scope**: event-sourced learner intelligence for mock interview turns  
**Directive baseline**:

1. Immutable events only as source of truth
2. Immediate remediation from turn 1
3. Schema-first selector inputs before loop changes
4. Tenant-scoped immutable events with canonical skill namespaces

---

## What changes from the rejected design

Three hard corrections:

- **No mutable `AdaptivePlan` document as truth.** The truth is append-only turn evidence plus append-only session folds.
- **No early-turn drill suppression.** If the system sees a credible gap on turn 1, it shows its intelligence on turn 1.
- **No Firestore query strategy that depends on nonexistent tags.** `MockInterviewQuestion` now gets first-class selector tags before any reranking logic depends on them.

This design scales because it separates:

- immutable evidence
- rebuildable projections
- hot-path deterministic decisions

---

## 1. Source Of Truth

### Immutable event layers

The only authoritative learner data is:

1. `TurnTraceDoc`
   - one append-only event per answered question
   - contains tenant/org/user identity, rubric, hidden-state snapshots, question context, and canonical skill refs

2. `SessionFoldDoc`
   - one append-only event per completed session
   - compact aggregate to avoid replaying every turn forever
   - carries the same tenant/org/user contract as turn events

Everything else is a projection or cache.

### Rebuildable projections

Two optional read models exist for speed only:

- `DimensionLedgerDoc`
- `SkillGraphDoc`

If they drift, they are deleted and rebuilt from the immutable event stream.

That is the core scaling discipline: mutable docs can accelerate reads, but they never define reality.

---

## 2. Boot Flow That Actually Scales

### Session start

On `create_session`:

1. read `mock_interview_user_state.current_elo`
2. optionally read `DimensionLedgerDoc`
3. optionally read `SkillGraphDoc`
4. if projections are missing or stale:
   - start with Elo + defaults
   - enqueue projection rebuild in background

Critical point:

- the session can start without any projection doc
- no startup depends on a precomputed “next plan”
- stale projections degrade quality, not correctness

### Why this is better than `AdaptivePlan`

Because a stale projection can be replayed.
A stale “next plan” silently changes behavior and contaminates future sessions.

---

## 3. Schema First: Question Tags

`MockInterviewQuestion` now needs explicit selector-facing ontology refs in the canonical schema:

- `skill_refs`
- `diagnostic_skill_refs`
- `remediation_skill_refs`

These are not optional implementation details. They are the minimum contract for a remediation-aware selector.

### How they are used

- `skill_refs`: what this question exercises
- `diagnostic_skill_refs`: what weakness it is good at exposing
- `remediation_skill_refs`: what remediation contexts it pairs well with afterward

### What we do **not** do

We do **not** add a new Firestore index and query by tags + array + Elo range in the first version.

Instead:

1. query candidates with the existing stable path:
   - `is_active`
   - `focus_area`
   - `elo_rating` window
2. rerank that candidate set in memory using the new tag fields

This avoids the previous hallucination where the query strategy was designed before the schema and before the benchmark.

---

## 4. Hot Path

### Current reality

The judge call dominates latency.

That means the only safe hot-path additions are:

- pure functions
- cached lookups
- small point reads at most

### Correct hot path

After judge result:

1. compute `gap_vector` from:
   - rubric scores
   - STAR completeness
   - current question tags
   - recent turn history in session memory
   - optional projection snapshots if already loaded
2. pick a remediation artifact immediately
3. rerank the next-question candidate pool against:
   - dominant gap
   - remediation binding
   - question tag overlap
4. write scored turn
5. fire-and-forget append `TurnTraceDoc`

No LLM generation belongs in this path.
No mutable cross-session plan belongs in this path.

---

## 5. Immediate Magic

The first 3 turns are where the user decides whether KitesForU is generic or magical.

So the policy is:

- **turn 1 onward**: remediation is allowed
- **confidence gates intensity**, not existence

### Turn-1 policy

If diagnosis confidence is low:

- show a lighter artifact
- phrase it as “best next move”
- keep the next question only softly biased

If confidence is high:

- show full micro-drill or targeted concept card
- bind next question more strongly

### What we can experiment on

We can A/B:

- drill wording
- drill length
- whether to show both drill and concept together
- next-question bias strength

We do **not** A/B “show intelligence” vs “hide intelligence” in early turns.

That was the wrong experiment.

---

## 6. Selector Design

### Step 1: stable candidate retrieval

Use the existing repository query path:

- active questions
- focus area
- Elo window with widening fallback

No new Firestore shape yet.

### Step 2: in-memory rerank

For the returned candidate pool, compute:

`candidate_score = elo_fit + tag_overlap + remediation_fit - recency_penalty`

Where:

- `elo_fit`: closeness to current target difficulty
- `skill_overlap`: overlap with dominant `skill_refs` / `diagnostic_skill_refs`
- `remediation_fit`: overlap with the active remediation target
- `recency_penalty`: existing anti-repeat behavior

### Why this scales

Because Firestore does what it is already good at:

- cheap filtered retrieval

and Python does what is safer to change quickly:

- ranking logic

This also means we can evolve tag semantics without reindexing the entire bank every time.

---

## 7. Remediation Engine

### Allowed from turn 1

The remediation router runs every scored turn.

It outputs:

- `none`
- `micro_drill`
- `targeted_concept_card`

### Confidence-based ladder

1. low confidence:
   - one-line nudge or lightweight drill
2. medium confidence:
   - structured micro-drill
3. high confidence:
   - micro-drill + targeted concept card or stronger next-question binding

### Required property

Every remediation artifact must emit a `next_question_bias`.

Otherwise it is theater.

---

## 8. Tenant-Scoped Event Contract

### Storage hierarchy

Every immutable event and every derived projection lives under the tenant/org/user path:

- `tenants/{tenant_id}/orgs/{org_id}/users/{user_id}/mastery_trace_turns/{turn_id}`
- `tenants/{tenant_id}/orgs/{org_id}/users/{user_id}/mastery_trace_sessions/{session_id}`
- `tenants/{tenant_id}/orgs/{org_id}/users/{user_id}/mastery_trace/dimensions`
- `tenants/{tenant_id}/orgs/{org_id}/users/{user_id}/mastery_trace/skills`

This gives two safety rails:

- path-level partitioning for storage and export/delete
- in-document tenant/org/user fields for collection-group replay and audit

### `TurnTraceDoc`

Required fields:

- `event_schema_version`
- `turn_id`
- `session_id`
- `tenant_id`
- `org_id`
- `user_id`
- `skill_ontology_version`
- `role_code`
- `domain_code`
- `question_id`
- `question_focus_area`
- `question_difficulty_elo`
- `answer_duration_ms`
- `rubric_version`
- `scores`
- `hidden_state_before`
- `hidden_state_after`
- `elo_before`
- `elo_after`
- `skill_refs`
- `coaching_tone_used`
- `occurred_at_ms`
- `ingested_at_ms`

### `SessionFoldDoc`

Required fields:

- `event_schema_version`
- `session_id`
- `tenant_id`
- `org_id`
- `user_id`
- `skill_ontology_version`
- `role_code`
- `domain_code`
- `rubric_version`
- `total_turns`
- `start_elo`
- `end_elo`
- `max_pressure`
- `min_support`
- `avg_weighted_total`
- `weak_dimensions`
- `dominant_skill_refs`
- `session_started_at_ms`
- `session_ended_at_ms`
- `ingested_at_ms`

### Replay rule

Replay workers may only read immutable events under a single tenant/org prefix, and every replay query must validate:

- path `tenant_id == document.tenant_id`
- path `org_id == document.org_id`
- requested `skill_ontology_version` is understood by the replay code

If any check fails, the event is quarantined instead of merged.

---

## 9. Canonical Skill Namespace

No free-form tags.

Every skill reference is a versioned `SkillRef` with one of three namespaces:

1. `cross_industry`
   - stable skills shared across industries
   - key format: `ci.{family}.{skill}`
   - example: `ci.behavioral.quantified_impact`

2. `domain_specific`
   - industry or regulatory knowledge that must never collide with other industries
   - key format: `dom.{domain}.{family}.{skill}`
   - example: `dom.healthcare.compliance.hipaa_minimum_necessary`
   - example: `dom.fintech.risk.kyc_exception_handling`

3. `role_overlay`
   - role and seniority overlays independent of industry
   - key format: `role.{role}.{seniority}.{family}.{skill}`
   - example: `role.product_manager.senior.storytelling.executive_tradeoff_narrative`

### Collision rule

Two skills only share a key if they share the same meaning.

- healthcare privacy skills and fintech risk skills can never collide because `domain` is part of the key
- generic quantified-impact skills are shared intentionally through the `ci.` namespace
- role overlays are isolated by role and seniority, so `engineering_manager` and `account_executive` cannot alias each other

### Five-year replay example

Healthcare enterprise:

- tenant: `tenant_enterprise`
- org: `org_healthcare`
- turn skill refs:
  - `ci.behavioral.quantified_impact`
  - `dom.healthcare.compliance.hipaa_minimum_necessary`
  - `role.product_manager.senior.storytelling.executive_tradeoff_narrative`

Fintech enterprise:

- tenant: `tenant_enterprise`
- org: `org_fintech`
- turn skill refs:
  - `ci.behavioral.quantified_impact`
  - `dom.fintech.risk.kyc_exception_handling`
  - `role.account_executive.senior.discovery.executive_alignment`

These traces can be replayed for five years with zero bleed because:

- tenant and org are in the path and the document
- the ontology key itself encodes whether a skill is shared, domain-bound, or role-bound
- projections rebuild from immutable evidence without mutating cross-tenant state

---

## 10. Event Model

### Projections

`DimensionLedgerDoc` and `SkillGraphDoc` become:

- disposable
- recomputable
- non-authoritative

They get fields like:

- `projection_version`
- `rebuilt_from_session_id`

So drift is visible and recoverable.

---

## 11. Worker Topology

### End-of-turn

- append `TurnTraceDoc`
- do not block response on trace success

### End-of-session

1. append `SessionFoldDoc`
2. rebuild or incrementally update projections
3. publish metrics

No stage rewrites a per-user forward plan document.

If multiple sessions end in parallel:

- both append folds
- projection worker replays folds in order
- last successful projection write is safe because the projection is derived, not truth

That is the concurrency fix.

---

## 11. Product “Wow Factor” Rules

To preserve the magic:

- remediation is visible from turn 1
- remediation is specific to the exact miss, not generic coaching
- the very next question visibly reflects what just happened
- the user can feel the system paying attention immediately

To preserve B2B SaaS discipline:

- no mutable per-user plan as truth
- no unbenchmarked Firestore query complexity
- no product claims derived from speculative multiplicative ROI math

---

## 12. ROI Framing, Corrected

We are dropping the fake multiplicative 5.6× model.

For this feature, the architecture document should make **no blended ROI claim** until measured.

Instead, define success criteria as operational hypotheses:

- remediation improves the targeted dimension within the next 3 turns
- turn-1 visible remediation increases session continuation after a weak first answer
- tagged reranking reduces repeat wasted questions
- event-sourced projections remain replayable and drift-safe under parallel sessions

This is honest, measurable, and suitable for B2B buyers later.

---

## 13. Build Order

### Phase 0 — schema only

1. update `MockInterviewQuestion` with:
   - `skill_refs`
   - `diagnostic_skill_refs`
   - `remediation_skill_refs`
2. update tests
3. do not touch selector behavior yet

### Phase 1 — event model correction

1. add `tenant_id`, `org_id`, and `user_id` to every immutable event
2. add `skill_ontology_version`, `role_code`, and `domain_code` to every immutable event
3. mark `DimensionLedgerDoc` and `SkillGraphDoc` as derived projections only

### Phase 2 — selector rerank

1. keep existing Firestore retrieval
2. add in-memory rerank by new question tags
3. benchmark candidate pool size and latency before any new index discussion

### Phase 3 — immediate remediation

1. enable remediation from turn 1
2. confidence-gate intensity, not existence
3. bind next question to remediation outcome

### Phase 4 — projection rebuild + ops

1. add replay worker
2. add drift detection
3. add rebuild tooling

---

## 14. Non-Negotiables

- append-only turn events are the truth
- append-only session folds are the second truth layer
- all mutable docs are projections only
- turn-1 remediation is on
- query strategy follows schema, not the other way around
- tenant and org identifiers exist in both path and document
- no free-form skill tags in the canonical schema
- no new Firestore tag index until a benchmark proves it is needed

---

## Bottom line

The scalable version of this system is not “more AI.”

It is:

- **better contracts**
- **immutable evidence**
- **replayable projections**
- **immediate visible intelligence**

That is the architecture that can survive B2B scale and still feel magical in the first minute.
