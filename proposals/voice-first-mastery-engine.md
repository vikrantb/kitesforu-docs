# Voice-First Mastery Engine — Technical Proposal

**Status**: Awaiting strategy audit (Codex)
**Author**: Voice + Intelligence Loop integration
**Date**: 2026-04-13
**Depends on**: `hyper-personalized-intelligence-loop.md` (approved, schemas v1.45.0 shipping)
**Do not implement** until the audit returns.

---

## What this proposal decides

The Intelligence Loop (Mastery Trace + remediation engine) was audited and
approved for text mode. This proposal wires the same loop into the
voice-first mock interview pipeline without breaking any of the
promises the audit locked in:

- immutable events as truth
- rebuildable projections only
- tenant/org/user scoping everywhere
- canonical `SkillRef` ontology (no free-form tags)
- no mutable "next plan" document
- confidence-gated remediation from turn 1
- anti-hallucination on drill copy (templates only)

The voice pipeline already exists in `kitesforu-workers` with a
LiveKit + Silero VAD + OpenAI STT (gpt-4o-mini-transcribe) +
ElevenLabs Flash v2.5 stack, targeting p50 < 900 ms turn round-trip.
The question is: what does it take to make the Intelligence Loop's
MicroDrill, Targeted Concept Card, and event-sourced trace writes
work inside a sub-second voice turn without breaking the 900 ms
latency promise?

---

## 1. The latency budget reality

A voice turn's p50 budget is 900 ms from `user stopped talking`
to `first TTS audio frame at the user's ear`. The text mode
submit-answer path has ~15 s of judge latency and nobody notices —
voice mode has 900 ms and everybody notices. Every cell in the table
below must stay sub-budget.

| Stage | Current text-mode | Current voice-mode (stub) | Proposed voice-mode with IL |
|---|---|---|---|
| STT final partial | — | 80–150 ms (OpenAI gpt-4o-mini-transcribe) | unchanged |
| VAD turn-end detection | — | 40–90 ms (Silero) | unchanged |
| Rubric judge | 10–20 s (gpt-5-mini) | **blocking** today | **non-blocking**, see §3 |
| Dominant-weakness + drill pick | ~1 ms | — | ~1 ms, on critical path |
| Targeted concept lookup | 20–40 ms (Firestore point read) | — | ~30 ms, **parallel** to judge |
| TTS TTFB | — | 60–80 ms (ElevenLabs Flash v2.5) | unchanged |
| Event write (TurnTraceDoc) | ~20 ms | — | fire-and-forget, off-path |
| **Total on critical path** | 10–20 s | ~200 ms | **~220 ms stretch, ~170 ms typical** |

The audit's golden rule holds: **no LLM call lives on the critical
remediation path**. The drill pick is a hard-coded template lookup.
The concept lookup is a Firestore point read on an indexed
`remediation_tag` field. Both run in parallel with STT finalization
so they cost zero added latency if they complete before STT does
(which they always will — STT is the slower step).

---

## 2. What "voice-first IL" actually ships

The Intelligence Loop in voice mode is **not** a visual card between
turns. That would break the "never play speech over speech" rule.
Instead:

1. **The MicroDrill is spoken as the interviewer's natural
   follow-up**. The drill template is the transcript of what the
   interviewer says next, not a panel the user reads. Example: the
   `two_numbers_rule` drill's instruction
   _"Rewrite your closing sentence with one % improvement and one
   absolute number"_ becomes the interviewer's prosody-matched
   turn: _"Give me that again — I want one percentage and one
   absolute number."_ Same template, same anti-hallucination
   discipline, different channel.

2. **The Targeted Concept Card is silent** — it appears in the
   companion UI panel (tablet / second screen / car mode dashboard)
   but never interrupts the audio. If the user is audio-only (no
   screen), the concept card is skipped on that turn and enqueued
   for the session summary.

3. **The event trace writes happen exactly as in text mode**. Voice
   pipeline fires `TurnTraceDoc` to the same path under
   `tenants/{t}/orgs/{o}/users/{u}/mastery_trace_turns/{turn_id}`.
   The DLQ + retry + rebuild semantics are identical.

4. **Session-end rollup is unchanged**. When the LiveKit room
   disconnects, the session-end worker runs the same projection
   rebuild (DimensionLedger + SkillGraph). Voice adds no new
   worker — it consumes the one the text loop already has.

This is the critical design choice: the voice adapter is a
**channel**, the Intelligence Loop is the **brain**. The brain does
not care which channel fed it the turn.

---

## 3. Real-time routing — voice critical path

The voice critical path has the same shape as text but with three
structural differences:

1. **STT finalization replaces the "click submit" moment.** The
   turn-trace write fires when Silero VAD reports silence and OpenAI
   STT returns a final transcript — not when a button is clicked.
2. **The rubric judge runs async in parallel with the TTS response.**
   In text mode the judge blocks the response. In voice mode we
   cannot wait for the judge — we need TTS bytes flowing in 900 ms.
   So the voice worker:
   - picks a **provisional** drill based on a 1-shot heuristic
     (STAR completeness from the STT transcript + `skill_refs` on
     the current question)
   - speaks the next question immediately using the provisional
     drill's template as the interviewer's follow-up
   - runs the real gpt-5-mini rubric judge in the background
   - **reconciles** the real judge result with the provisional
     drill when the judge returns, writes the final `TurnTraceDoc`,
     and if the real weakness differs from the provisional one,
     the NEXT turn's drill absorbs the correction
3. **The session-end worker must also settle any in-flight judge
   calls before running the rollup.** We cannot project from
   unfinished evidence. This is a timing discipline, not a new
   worker.

### Provisional drill — how it's picked in ~5 ms

The provisional drill is a deterministic function of:

- STAR presence from the STT final transcript (client-side regex
  on the 4 STAR markers — "when I", "my task", "I did", "the
  result")
- `skill_refs` on the current question (from `MockInterviewQuestion`)
- The last-observed `DimensionLedger` projection from the session
  start (loaded once at session boot, cached in worker memory)

No LLM. No Firestore read on the critical path. Pure function over
in-memory data. The proposal locks this at ~5 ms p99.

When the real rubric judge returns (~12 s later, in the middle of
the NEXT turn), the worker writes the authoritative `TurnTraceDoc`
with the real scores. If the real dominant weakness matches the
provisional one, nothing changes — the user already got a
well-targeted drill. If the real weakness differs, the next turn's
drill absorbs the correction. The user perceives "the interviewer
adapts turn-to-turn," not a two-step reconciliation.

### What this costs vs what it buys

**Cost**: one additional in-memory state — a small
`pending_judge_resolutions` map in the voice worker process that
tracks which turns have provisional drills awaiting reconciliation.
If the worker crashes mid-session, the map is lost and the affected
turns get rewritten from the real judge result after session end.
The user sees nothing different. The trace is eventually correct.

**Buys**: the full Intelligence Loop active inside a sub-second
voice turn. No waiting for the judge. No compromise on the drill
template anti-hallucination rule. No compromise on the event-sourced
audit.

---

## 4. The four integration points

| # | Integration point | File owner | What changes |
|---|---|---|---|
| 1 | **Voice worker turn handler** | `kitesforu-workers/src/interview_worker/turn_handler.py` | Adds provisional drill picker (5 ms, pure function). Calls same drill template library as text mode. |
| 2 | **TTS prompt adapter** | `kitesforu-workers/src/interview_worker/voice/tts_adapter.py` | Receives the drill template as the interviewer's next-turn transcript. No generation — template only. |
| 3 | **Event writer** | `kitesforu-workers/src/interview_worker/mastery/event_writer.py` | **New file**. Writes `TurnTraceDoc` on STT final, `SessionFoldDoc` on session end. Same tenant-scoped paths as text mode. |
| 4 | **Judge reconciliation** | `kitesforu-workers/src/interview_worker/mastery/judge_reconciler.py` | **New file**. Awaits async judge results, rewrites turn trace with authoritative scores, updates the next turn's drill priority. |

No new service. No new worker. Two new files inside the existing
voice worker. The Intelligence Loop substrate (drill templates,
remediation lookups, event writers) is shared code that gets
imported by both the text-mode API route and the voice worker. No
duplication.

---

## 5. Tenant, org, user scoping in voice

The text mode path gets `tenant_id`, `org_id`, `user_id` from the
Clerk JWT on the HTTP request. Voice mode gets them from the
LiveKit room metadata. The LiveKit token minted at room join time
already carries Clerk claims — we need to extract them into the
worker's room context at connection time and attach them to every
turn's event write. This is a one-line read in the existing
`on_room_connected` handler.

If the metadata is missing (test rooms, dev runs), the event writer
falls back to `tenant_id="dev"`, `org_id="dev"`, `user_id="anon"`,
and logs a warning. Dev traces never pollute production tenant
paths — they're physically isolated by the path prefix.

---

## 6. The measurement — does it move the latency budget?

This is the single number I most want the audit to interrogate.

**Current voice turn p50**: ~200 ms from VAD turn-end to first TTS
audio frame (Phase 2 harness measurements, ElevenLabs Flash v2.5
TTS, ~80 ms TTFB, ~150 ms STT final, ~50 ms worker overhead).

**Proposed voice turn p50 with IL**: ~220 ms. The +20 ms is the
provisional drill picker (5 ms) + the template-to-prompt
composition (5 ms) + a wider prompt context (10 ms TTS prompt
processing). The Firestore concept lookup is parallelized with STT
— it does not touch the critical path on the p50.

**Proposed voice turn p99**: ~320 ms. The +100 ms over p50 is the
tail of the Firestore concept lookup when the project's Firestore
region is far from the worker (which has happened once — the
us-central1 worker hit a cold us-east4 read). The fix is a
regional read preference, which is independent of this proposal.

All three numbers stay under the 900 ms promise. **The p99 budget
has 580 ms of headroom even with IL wired in.** That matters
because the real voice engineering work (prosody-aware barge-in,
dropout recovery, room acoustics) can consume that headroom without
the Intelligence Loop becoming the bottleneck.

### What I will NOT claim

I will not claim the IL makes voice faster. It does not. It makes
voice **smarter** at a ~20 ms p50 / ~100 ms p99 cost. I'm willing
to bank those numbers as hard upper bounds — if the benchmark shows
higher, we stop and rethink.

---

## 7. Open questions I want the audit to kill or sharpen

1. **Provisional drill accuracy.** How often does the 5-ms pure-function
   picker agree with the real rubric judge's dominant weakness?
   Target: ≥70 %. Below that, the "next turn absorbs the correction"
   story breaks down because the correction is the dominant case.
   We need a synthetic test harness with 100 recorded turns and
   their judge results before we ship this to real users.

2. **Silent concept card vs audio-only users.** I said the concept
   card is "silent" (UI panel) and audio-only users get the card
   enqueued for the session summary. But is that actually valuable?
   A concept card 15 minutes after the fact may be forgotten. Do
   we read a 1-sentence version aloud between turns for audio-only
   users, or suppress entirely?

3. **Barge-in during interviewer follow-up.** If the interviewer
   speaks a MicroDrill ("Give me that again — one % and one
   number") and the user immediately starts talking over it, does
   the drill count as "served" or "aborted"? The event trace needs
   to record this clearly or the projection's drill-effectiveness
   metric is noise.

4. **Reconciliation window for judge results.** The provisional
   drill is speculative. If the real judge is 40 seconds late
   (pathological case), do we retroactively rewrite the trace for
   a turn the user finished 3 turns ago? What's the cutoff?

5. **Room-metadata tenant scoping failure.** If the LiveKit token
   doesn't carry Clerk claims for any reason — test runs, ngrok
   dev, a Clerk outage — the event writer falls back to `dev`
   paths. Is this safe enough, or do we want a hard fail-closed
   posture for production rooms?

6. **Dual-mode sessions.** If a user starts text mode and then
   switches to voice mid-session (or vice versa), do the two
   event streams land under the same `session_id`? Or does the
   mode change start a new session? The projection math assumes
   "one session = one mode" but nothing in the data model
   enforces it.

7. **Voice rubric version drift.** The voice judge might use a
   different rubric version than text (faster/cheaper model with
   fewer dims). If so, the DimensionLedger projection has to be
   keyed by mode, or we poison the cross-mode signal.

8. **ElevenLabs cost at session scale.** A 15-minute voice session
   is ~$0.72 in TTS at current rates. Adding a 1-sentence drill
   follow-up per turn (~8 turns × ~15 words × ~0.5 seconds) adds
   ~$0.10 per session. Roughly +14 % TTS cost. Acceptable at our
   current price point; flagging for the auditor because it
   compounds on the Intelligence Loop's 5.6× ROI claim.

---

## 8. What I already wrote (and will not push)

Nothing. This is proposal-only per the triangulation rule. No code
exists yet for the voice worker integration. The schemas package
(PR kitesforu-schemas #65) is the only thing on the wire and it
was audited and approved before this proposal was written.

---

## 9. Ship order (if approved)

**Cannot parallelize — strict dependency chain.**

1. **Audit this proposal.** Stop here until Codex signs off.
2. Wait for schemas v1.45.0 PR #65 to land and publish.
3. `kitesforu-workers` PR A: add provisional drill picker + template
   library + event writer, with tests. No reconciler yet. No real
   voice behavior change — the drill picker runs but its output is
   logged only, never spoken.
4. **Audit checkpoint.** Observe the logs for ≥ 100 real voice
   turns. Measure provisional vs real judge agreement rate.
5. `kitesforu-workers` PR B: wire the picker into the TTS prompt
   adapter. The interviewer now speaks the drill follow-up.
   Reconciler ships in the same PR.
6. **Audit checkpoint.** Observe p50 / p99 latency against the
   900 ms budget. If p99 exceeds 400 ms, stop and rethink.
7. `kitesforu-frontend` PR: render the silent concept card in the
   voice-preview companion panel. Optional per turn.
8. Playwright stress test across 10 voice scenarios mirroring the
   text-mode stress test.

---

## Anti-patterns the voice proposal explicitly rejects

- **No LLM on the voice critical path.** Drill picks are pure
  functions. Concept picks are Firestore point reads. Judge runs
  in the background.
- **No speech over speech.** The MicroDrill is the interviewer's
  natural next utterance, not a parallel narration.
- **No silent blocking on projection reads.** If `DimensionLedger`
  is not in the worker's warm cache at turn N, the picker uses
  session-only signal and moves on. Projection reads are best-effort.
- **No new mutable document.** The voice loop consumes the same
  event-sourced trace as the text loop. No per-session "voice
  plan" gets persisted.
- **No retroactive rewriting of TTS transcripts.** Once the
  interviewer has spoken a drill follow-up, the audit trail says
  what was spoken. Reconciliation only rewrites the SCORING of
  that turn, never the audio history.
- **No cross-tenant event writes.** Every write goes through the
  tenant-scoped path from room metadata. Fallback `dev` paths are
  physically isolated.

---

**End of proposal.** Hand this file to Codex alongside the approved
Intelligence Loop proposal. Nothing will be written until the audit
returns.
