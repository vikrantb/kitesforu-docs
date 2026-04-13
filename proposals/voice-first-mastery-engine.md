# Voice-First Mastery Engine — Silence Rules Rewrite

**Status**: Post-audit rewrite  
**Author**: Voice + Intelligence Loop integration  
**Date**: 2026-04-13  
**Depends on**: `hyper-personalized-intelligence-loop.md` (approved, schemas v1.45.0 shipping)  
**Directive baseline**:

1. Audio responsiveness is sacred
2. The audio thread gets no Firestore reads, no Firestore writes, and no LLMs
3. Firestore is downstream durability, never inline coordination
4. Mastery features must shed before voice quality sheds

---

## What changed

The previous proposal failed one core test: it still assumed the voice
path could safely reuse text-mode state access patterns.

That is rejected.

This rewrite defines a **true voice-first conversational engine**:

- the audio thread operates only on a bounded in-memory room snapshot
- turn evidence leaves the room process as queue-backed event envelopes
- all judging, reconciliation, projection rebuilds, and Firestore writes
  happen downstream of the audio path
- if the system is under load, mastery disappears before audio does

The intelligence loop is now a **best-effort augmentation layer** on top of
the conversation, not a dependency of the conversation.

---

## 1. Silence Rules

These rules are non-negotiable.

### Rule 1: zero-latency audio thread

The audio thread may do only:

- VAD turn-end detection
- STT finalization
- bounded in-memory state lookup
- deterministic provisional drill selection
- next-utterance assembly
- TTS request dispatch

It may do **none** of the following:

- Firestore reads
- Firestore writes
- queue drains
- LLM judging
- projection reads
- retry logic
- tenant lookup by remote call

### Rule 2: bounded join-time snapshot only

At room join, the worker loads a single bounded snapshot into memory.

That snapshot is the only state the audio thread may consult during turns.

### Rule 3: asynchronous durability only

Every turn emits an immutable event envelope to a queue.

The queue consumer, not the audio thread, owns:

- Firestore durability
- judge calls
- reconciliation
- session folds
- projection rebuilds

### Rule 4: mastery sheds before audio

If the system is hot, drop features in this order:

1. targeted concept card
2. drill personalization intensity
3. spoken provisional drill
4. event enrichment fields
5. background reconciliation speed

Never drop:

- turn-end detection
- STT final transcript
- next-turn speech
- TTS start

---

## 2. Room Snapshot Contract

The room snapshot is loaded once at join time by a **control-plane loader**
before the room is marked ready for conversation.

The audio thread receives the fully materialized snapshot in memory.

### Snapshot shape

- `tenant_id`
- `org_id`
- `user_id`
- `session_id`
- `snapshot_version`
- `skill_ontology_version`
- `current_question_id`
- `current_question_focus_area`
- `current_question_skill_refs`
- `question_bank_hint`
- `provisional_policy`
- `recent_weakness_summary`
- `recent_drill_outcomes`
- `role_code`
- `domain_code`
- `voice_mode_flags`

### Snapshot constraints

- max size is fixed and small
- no unbounded history
- no raw turn replay
- no dynamic Firestore fetch during a turn
- if a field is missing, the worker falls back to a simpler behavior in memory

### Snapshot refresh model

The snapshot may refresh only at safe boundaries:

- room join
- after a completed turn, if the refresh is already available
- after a reconnect

The audio thread never blocks waiting for refresh.

---

## 3. Critical Path

### Hard latency definition

Budget: `user stopped talking` to `first TTS audio frame`

Target:

- p50 < 300 ms
- p95 < 600 ms
- p99 < 900 ms

### Critical-path stages

| Stage | Owner | Allowed dependencies |
|---|---|---|
| VAD turn-end | audio thread | local only |
| STT final | audio thread | STT provider only |
| provisional drill pick | audio thread | in-memory snapshot only |
| next utterance assembly | audio thread | local template library only |
| TTS dispatch | audio thread | TTS provider only |

Everything else is off-path.

### Provisional drill policy

The spoken follow-up is selected by a deterministic function over:

- STT final transcript
- current question skill refs
- recent weakness summary already present in the room snapshot
- room mode flags

No datastore.
No LLM.
No queue round-trip.

If the snapshot is stale or incomplete, the provisional drill degrades to:

- generic follow-up
- or no drill at all

The user hears a less personalized interviewer, not a slower interviewer.

---

## 4. Async Durability Pipeline

The room process emits immutable envelopes and returns to conversation.

### Envelope flow

1. audio thread creates `TurnEventEnvelope`
2. envelope is pushed to queue
3. queue consumer persists `TurnTraceDoc`
4. async judge computes authoritative scores
5. reconciler emits correction event if needed
6. session-end consumer writes `SessionFoldDoc`
7. projection workers rebuild `DimensionLedgerDoc` and `SkillGraphDoc`

### Queue contract

The queue payload must include:

- event type
- envelope version
- tenant/org/user/session identifiers
- turn identifier
- room identifier
- audio mode metadata
- transcript
- question context
- provisional drill served
- timestamps

### Durability principle

The queue is the first durable handoff.

Firestore is downstream storage for:

- append-only event truth
- append-only session folds
- rebuildable projections

If Firestore is degraded, the room still runs.
If the queue is degraded, the room falls back to minimal audio mode and drops
mastery features before it risks TTS latency.

---

## 5. New Runtime Topology

This is no longer “reuse the text brain.”

It is a split execution model.

### A. Room process

Owns:

- LiveKit room lifecycle
- VAD
- STT finalization
- bounded snapshot
- provisional drill picker
- TTS dispatch
- queue publish

Does not own:

- Firestore client
- judge client
- projection client

### B. Mastery ingest consumer

Owns:

- queue consumption
- `TurnTraceDoc` persistence
- delivery metrics
- dead-letter routing

### C. Judge consumer

Owns:

- async rubric judging
- authoritative turn scoring
- reconciliation event emission

### D. Projection consumer

Owns:

- `SessionFoldDoc`
- `DimensionLedgerDoc`
- `SkillGraphDoc`
- snapshot refresh source material

This is the separation that preserves the 900 ms promise.

---

## 6. Spoken Intelligence

### What is spoken

Only the provisional drill follow-up may be injected into the next interviewer
utterance, and only if it clears the latency and confidence gates.

Example:

- base next-turn opener: "Let's try that again."
- provisional drill add-on: "This time give me one percentage and one absolute number."

### What is not spoken

- concept card body
- judge rationale
- reconciliation correction
- projection-derived narration

Those remain UI-only or summary-only.

### Confidence gate

The room process speaks a drill only if:

- snapshot is present
- heuristic confidence clears threshold
- no shed mode is active
- no barge-in penalty is active

Else it speaks the ordinary next turn.

---

## 7. Shed Order

Under load or degraded dependencies, the engine must shed in this exact order.

### Shed level 0: full voice-first intelligence

- spoken provisional drill on
- queue publish on
- concept card enqueue on
- async judge on
- reconciliation on

### Shed level 1: no concept card

- spoken drill stays on
- concept card suppressed

### Shed level 2: generic spoken drill only

- no personalized drill
- use generic short coaching phrases only

### Shed level 3: no spoken drill

- pure interview continuity
- queue still attempts minimal event envelope

### Shed level 4: minimal durability

- queue only essential envelope fields
- no enrichment
- no concept
- no drill metadata

### Shed level 5: audio-only survival mode

- no mastery features
- no queue dependency on the critical path
- conversation continues with plain interviewer prompts

The rule is simple:

**a less intelligent interviewer is acceptable; a slower interviewer is not.**

---

## 8. Tenant Scoping In Voice

Tenant integrity remains mandatory.

### Source of tenant context

The room token must carry:

- `tenant_id`
- `org_id`
- `user_id`
- `session_id`

These are resolved at room join and frozen into the room snapshot.

### Fail-closed production rule

For production rooms:

- if tenant metadata is missing, the room must not start mastery mode
- the worker may still run plain interview audio if product explicitly allows it
- it must not emit production mastery events with guessed identifiers

### Dev rule

Only explicitly marked dev rooms may use:

- `tenant_id="dev"`
- `org_id="dev"`
- `user_id="anon"`

No silent fallback for production traffic.

---

## 9. Measurement

The old proposal was too optimistic because it budgeted datastore work into the
voice path.

This rewrite measures only what the room process actually does.

### Primary metrics

- TTFB p50
- TTFB p95
- TTFB p99
- percent of turns with spoken provisional drill
- percent of turns where shed mode activated
- percent of queue publishes succeeding within local timeout
- provisional drill agreement rate with authoritative judge
- barge-in interruption rate on spoken drill turns

### Hard release gates

Do not ship spoken drills if any of these fail:

- TTFB p95 > 600 ms
- TTFB p99 > 900 ms
- spoken drill turns increase interruption rate materially
- provisional drill agreement rate is too low to preserve trust

### What we do not count as success

- Firestore eventual consistency
- projection freshness
- concept card fill rate

Those matter, but never more than conversational speed.

---

## 10. Ship Order

This remains staged, but PR B is now split correctly.

### PR A — log-only substrate

Scope:

- room snapshot loader
- deterministic provisional drill picker
- local template library
- event envelope builder
- queue producer
- shed-mode controller
- metrics only, nothing spoken

Exit criteria:

- queue publish path proven off the audio thread
- no measurable TTFB regression
- provisional drill agreement benchmark collected

### PR B1 — spoken provisional drills with hard latency gates

Scope:

- inject spoken provisional drill into next utterance
- enforce shed levels
- enforce confidence gate
- enforce latency gate kill-switch

Exit criteria:

- TTFB p95 and p99 remain inside budget
- interruption rate acceptable
- drill behavior trusted enough to expose to users

### PR B2 — reconciliation and projection consumption

Scope:

- async judge consumer
- reconciliation events
- session-fold writes
- projection rebuilds
- snapshot refresh source integration

Exit criteria:

- no effect on audio-path latency
- eventual trace correctness proven
- projection rebuilds remain tenant-safe

### PR C — companion UI

Scope:

- silent concept card rendering
- debug visibility for provisional vs authoritative outcome

This is optional for voice launch.

---

## 11. Anti-Patterns Explicitly Rejected

- no Firestore client in the audio thread
- no LLM client in the audio thread
- no text-route repository reuse inside the room process
- no synchronous “quick lookup” exceptions
- no silent production fallback to fake tenant IDs
- no spoken mastery behavior without a kill-switch
- no reconciliation dependency before spoken drills are latency-safe

---

## 12. Bottom Line

The true voice-first version of the Intelligence Loop is not a richer API path.

It is a stricter runtime:

- **sealed audio thread**
- **bounded in-memory snapshot**
- **queue-first durability**
- **mastery features that can disappear instantly**

If the system is under pressure, the intelligence must disappear before the audio does.

