# R2 — Voice Consolidation: LiveKit Adapter (replaces MockVoiceController)

**Status**: PROPOSED (awaiting Codex audit before any code per triangulation rule)
**Priority**: P1 (closes R2-phase1 D1; unblocks R2-phase2 D3 last AC "voice mock uses real speech recognition")
**Effort**: ~2-3 weeks (frontend + voice-worker + infra; phased)
**Affected repos**: `kitesforu-frontend` (LiveKit client + adapter), `kitesforu-workers` or new `kitesforu-voice-worker` (LiveKit room + agent), `kitesforu-api` (room-token endpoint), `kitesforu-infrastructure` (LiveKit deployment + secrets)
**Depends on**: R2-phase1 D3 content rating (shipped) for safety constraints in voice prompts.
**Blocks**: R2-phase2 D3's voice-mock real-STT AC; future voice-first interview-prep ships.
**Origin**: R2-phase1 D1 close-out summary at `done/R2-phase1-content-quality-voice.md` named the LiveKit adapter as the dedicated thread; `MockVoiceController.tsx` docstring confirms "real voice session will replace this with a LiveKit event adapter, but every component it drives is production code."

---

## 1. One-paragraph thesis

The voice-first mastery surface (`/interview-prep/mock/voice-preview`) is fully built — RoomSurface morph, ListeningHalo (4 states + degraded), STAR shadow rail, OneFix replay card, CarryForwardChip, MasteryTransitionToast — and driven by a Zustand store with six slices (`voiceSessionSlice`, `masterySlice`, `roomAtmosphereSlice`, `starShadowSlice`, `replayCardSlice`, `coachingTargetSlice`). All state mutations are pure (every visual is a function of store state per the proposal). What's missing is the input source: today `MockVoiceController` (383 lines, preview-only theater) flips store state on a setTimeout schedule to demo the surface; production needs a `useLiveKitVoiceAdapter` that bridges LiveKit room events (transcripts + agent responses + VAD + latency) into the same store mutations. This proposal scopes that adapter, the LiveKit room + agent backend that produces the events, and the deprecation path for `MockVoiceController` (kept as a flag-gated debug harness, not extracted into a shared hook). It does NOT consolidate `useCarVoiceOrchestrator` (Web Speech API for car-mode commands) into this stack — that orchestrator solves a different problem (offline-capable command parsing in a moving car) and stays as-is.

## 2. Goals / Non-goals

### Goals

1. Replace `MockVoiceController` as the production driver of the voice-mock surface with a LiveKit-backed adapter that emits the same store mutations from real STT + agent events.
2. Build the LiveKit agent (server-side) that runs interview-prep turns: prompts the user, captures STT, scores against the 5-dim rubric (specificity / quantified_impact / ownership / role_outcome_clarity / level_appropriate — same rubric `services/mock_interview/evaluation/prompt.py` already uses), emits turn events back to the room.
3. End-to-end latency budget: < 900 ms turn-end → TTS-start (the `degraded` halo threshold the slice already exposes).
4. Honor `content_rating` from R2-phase1 D3 in agent prompts — bedtime / G-rated never reaches voice mock, but the rating system propagates so future voice content (e.g. voice-first storytelling) inherits the constraint.
5. Keep `MockVoiceController` available behind `feature_voice_mock_harness` for dev / E2E so the visual proof-of-concept doesn't decay.

### Non-goals

- **NOT consolidating `useCarVoiceOrchestrator` into the LiveKit adapter.** Car-mode runs Web Speech API in-browser for command parsing while driving; LiveKit requires a sustained WebRTC connection that drains battery and breaks on intermittent connectivity. The two stacks serve genuinely different use cases.
- **NOT extracting a shared `useVoiceOrchestrator` hook** as the original R2-phase1 D1 proposal framed it. Per the 2026-04-21 strategic-state correction, MockVoiceController is preview-only theater that will be REPLACED by a LiveKit adapter, not refactored into a shared hook with the car-mode orchestrator.
- **NOT shipping voice-first storytelling, voice-first study, or any non-mock-interview voice surface in this proposal.** Each is a follow-up that consumes the same adapter.
- **NOT replacing the existing text-mock evaluation pipeline.** Text mock keeps the JSON-based prompt path in `services/mock_interview/evaluation/prompt.py`; voice mock uses the same rubric but runs the agent server-side.
- **NOT building a barge-in implementation in v1.** LiveKit's VAD already exposes the events; the adapter scopes barge-in as an explicit Phase 2 follow-up after baseline turn-taking is solid.

## 3. Current surface — file:line references

| Concern | Location |
| --- | --- |
| Voice-first preview page | `app/interview-prep/mock/voice-preview/page.tsx` |
| Mock driver (replaced by adapter) | `components/voice/MockVoiceController.tsx` (383 lines) |
| Production voice store | `hooks/voice/useVoiceStore.ts` (6 slices) |
| Voice session slice (halo / latency / turn) | `hooks/voice/slices/voiceSessionSlice.ts` |
| Mastery slice | `hooks/voice/slices/masterySlice.ts` |
| Replay card slice | `hooks/voice/slices/replayCardSlice.ts` |
| Star shadow slice | `hooks/voice/slices/starShadowSlice.ts` |
| Room atmosphere slice | `hooks/voice/slices/roomAtmosphereSlice.ts` |
| Coaching target slice | `hooks/voice/slices/coachingTargetSlice.ts` |
| Car-mode Web Speech orchestrator (NOT touched) | `hooks/car-mode-v2/useCarVoiceOrchestrator.ts` (354 lines) |
| Text-mock evaluation prompt | `kitesforu-api/src/api/services/mock_interview/evaluation/prompt.py` |
| Mock evaluation rubric (5 dims) | same path, system message structure |

Search for `livekit` returns ZERO matches in `package.json` or code today — this is greenfield client + server work.

## 4. Architecture

### 4.1 Frontend adapter

New file: `hooks/voice/useLiveKitVoiceAdapter.ts`

```ts
export interface VoiceAdapterConfig {
  roomToken: string         // short-lived JWT minted by /v1/voice/sessions
  serverUrl: string         // wss://livekit.kitesforu.app
  sessionId: string         // mock_interview session id
  contentRating: string     // 'PG' | 'PG_13' | 'R' (G excluded — agent refuses to run for G)
}

export function useLiveKitVoiceAdapter(config: VoiceAdapterConfig | null) {
  // 1. connect to LiveKit room
  // 2. publish local mic track
  // 3. subscribe to agent's TTS track + data channel
  // 4. on data channel events:
  //    - transcript_partial         → update vadRms
  //    - transcript_final           → setTurnPhase('between_turns')
  //    - agent_thinking             → setHaloState('thinking')
  //    - agent_response_starting    → setHaloState('asking') + setLatency(measured)
  //    - tts_audio_start            → setTurnPhase('in_turn')
  //    - turn_score                 → setStarFill + emit narrativeLabel via masterySlice
  //    - one_fix                    → replayCardSlice.setOneFix
  //    - chip_update                → coachingTargetSlice.setChip
  //    - phase_transition           → roomAtmosphereSlice.setPhase + masterySlice.setLabel
  // 5. on disconnect: setHaloState('idle'), setTurnPhase('ended')
}
```

The adapter has the **exact same shape** as `MockVoiceController.tsx`'s effect — same store getters, same mutation signatures. The only difference is the source of events (LiveKit data channel vs. setTimeout script).

### 4.2 Backend agent

LiveKit's "agent" framework runs a Python (or Node) worker that joins the room, listens to the user's audio track, runs STT, calls the LLM, runs TTS, publishes back. Two deployment options:

**Option A — LiveKit Cloud (managed)**: bypass infrastructure work, pay per minute. Pros: zero ops, fastest time to ship. Cons: vendor lock, ongoing cost scales with usage.

**Option B — Self-hosted on Cloud Run**: deploy LiveKit server + agent worker as Cloud Run services. Pros: control, fits existing infra pattern, cheaper at scale. Cons: WebRTC NAT traversal complexity, TURN server, more ops.

**Default proposal**: ship Phase 1 on **Option A (LiveKit Cloud)** to validate the product loop, then migrate to **Option B** in Phase 3 once daily voice-mock minutes justify the ops cost.

### 4.3 Agent loop pseudocode

```python
# kitesforu-voice-worker/agent.py (new repo or new service in workers)
class InterviewPrepAgent(VoiceAgent):
    async def on_user_speech_committed(self, msg: ChatMessage):
        # measure t0 = msg.committed_at
        score = await self.evaluator.score(msg.text, rubric=FIVE_DIMS)
        narrative_label = self.mastery.advance(score)
        next_question = await self.planner.next_question(narrative_label)
        # latency budget: t1 - t0 < 900ms before TTS audible
        await self.publish_data({
            'type': 'turn_score',
            'star': score.star,
            'dims': score.dims,
            'narrative_label': narrative_label,
        })
        if score.has_fixable_dim():
            await self.publish_data({
                'type': 'one_fix',
                'sentence': score.weakest_sentence,
                'rewrite': score.suggested_rewrite,
                'dimension': score.weakest_dim,
            })
        await self.tts(next_question)
```

The evaluator + mastery + planner reuse logic already in `kitesforu-api`'s `mock_interview` services — voice-worker imports them as a shared package (or via HTTP if isolation is preferred).

### 4.4 Token endpoint

New api route: `POST /v1/voice/sessions` → mints a 1-hour LiveKit JWT scoped to a single room name (`mock_interview_{user_id}_{ulid}`). The room is created lazily on first connect; the agent worker auto-joins via LiveKit's dispatch mechanism.

```python
@router.post("/v1/voice/sessions")
async def create_voice_session(req: CreateVoiceSessionRequest, user: AuthedUser):
    if user.content_rating_max == "G":
        raise HTTPException(403, "voice mock not available for G-rated accounts")
    room_name = f"mock_interview_{user.id}_{ulid()}"
    token = mint_livekit_token(room_name, user.id, ttl=3600)
    return {"room_name": room_name, "token": token, "url": settings.LIVEKIT_URL}
```

### 4.5 Latency budget — the load-bearing UX constraint

The `voiceSessionSlice` already exposes `latencyMs` and the `degraded` halo state at >900 ms. Hitting that budget end-to-end:

| Stage | Target | Dominant cost |
| --- | --- | --- |
| User speech-end → STT final | ~150-250 ms | Streaming STT (Deepgram / Soniox) |
| STT final → evaluator score | ~200-400 ms | LLM call (Sonnet 4.6 cached) |
| Score → next-question synth | ~100-200 ms | Cached planner |
| TTS first audible byte | ~100-200 ms | Streaming TTS (ElevenLabs / Inworld) |
| **Total** | **~550-1050 ms** | <900 ms achievable with caching + streaming |

Anthropic prompt caching (the `unit_economics_tracking` 78%-cacheable shape) directly applies — the system prompt + rubric + content-rating block is the static prefix; the user's transcript is the dynamic suffix. Cache hits drop evaluator latency by ~60% in steady state.

## 5. Phased scope

### Phase 1 — adapter + LiveKit Cloud + minimal agent (1 week)

- LiveKit Cloud account, project, server URL.
- New api route `/v1/voice/sessions` mints tokens.
- Voice agent (Python, deployed standalone on Cloud Run as `kitesforu-voice-worker`).
  - Streaming STT (Deepgram by default — cheapest streaming option with <250 ms first-final).
  - Anthropic Sonnet 4.6 evaluator (cached).
  - ElevenLabs streaming TTS.
  - Publishes data-channel events listed in §4.1.
- Frontend adapter `useLiveKitVoiceAdapter`.
- `app/interview-prep/mock/voice/page.tsx` (new, NOT the preview path) wires the adapter.
- `MockVoiceController` stays in voice-preview path behind `feature_voice_mock_harness` (default ON in dev, OFF in prod).
- Latency telemetry: every turn writes (stt_ms, eval_ms, tts_ms, total_ms) to Firestore for ops visibility.

### Phase 2 — barge-in + degraded-state recovery + analytics (1 week)

- Barge-in: when user starts speaking during agent TTS, agent stops TTS within 200 ms and returns to listening. Publishes `interrupted` event so the UI shows the partial response was cut.
- Degraded-state recovery: if 3 consecutive turns hit >900 ms latency, auto-fall back to text mock with a "voice connection unstable, switched to text" toast.
- Analytics events: `voice_session_started` / `voice_session_completed` / `voice_turn_completed` (with latency dims) / `voice_session_aborted` (with reason).

### Phase 3 — production hardening + Cloud Run migration (1 week)

- Migrate from LiveKit Cloud to self-hosted LiveKit on Cloud Run + TURN server.
- Add reconnect logic to the adapter (network blip → resume room).
- Mute / push-to-talk button for noisy environments.
- Persist session transcript to Firestore for the user's mastery-trace history.

## 6. Acceptance criteria

- [ ] `POST /v1/voice/sessions` mints a LiveKit JWT and returns `{ room_name, token, url }`. Rate-limited per-user. Refused for `content_rating_max=G`.
- [ ] Frontend adapter connects to LiveKit Cloud, publishes mic, subscribes to agent's TTS + data channel.
- [ ] Adapter mutates `useVoiceStore` via the SAME interface as `MockVoiceController` — no new slices needed.
- [ ] Agent's evaluator uses the SAME 5-dim rubric and `top_fix` / `rewrite` shapes as the text-mock evaluation prompt.
- [ ] Median turn latency < 900 ms in steady state (after first cached call). Measured via Firestore-persisted telemetry.
- [ ] `/interview-prep/mock/voice` (new production path) drives all 5 Gold Standard surfaces (RoomSurface morph, ListeningHalo, OneFix card, CarryForwardChip, StarShadow) from real events.
- [ ] `MockVoiceController` continues to work in `/interview-prep/mock/voice-preview` behind `feature_voice_mock_harness` (default ON in dev, OFF in prod).
- [ ] Content rating ≥ PG enforced at the token-mint route; G accounts get 403 with a friendly message.
- [ ] Latency telemetry persists `(stt_ms, eval_ms, tts_ms, total_ms)` per turn.
- [ ] Phase 2: barge-in stops TTS within 200 ms; UI surfaces the interrupted state.
- [ ] Phase 2: 3 consecutive >900 ms turns trigger auto-fallback to text mock with a clear toast.
- [ ] Phase 3: reconnect logic recovers from a 30-second network blip without losing session state.
- [ ] Playwright E2E on `beta.kitesforu.com` per CLAUDE.md rule 11: new voice mock → say one STAR answer aloud → verify halo states cycle through listening → thinking → asking, STAR fills, score chip shows.
- [ ] Both repos hard-block test gates pass (frontend + voice-worker).

## 7. Risks & mitigations

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Latency budget unachievable in real network conditions | Medium | Phase 1 uses LiveKit Cloud (lowest-latency managed). Telemetry tells us if budget holds; degraded-state fallback to text in Phase 2. |
| LiveKit Cloud cost runs hot if usage scales | Medium | Phase 3 self-hosts on Cloud Run + TURN. Already scoped. |
| Agent crashes mid-session, user stuck | Low | LiveKit dispatch auto-restarts the agent; adapter handles agent-disconnect by setting `haloState=idle` + showing a "reconnecting…" pill. |
| STT misrecognition rate ruins the evaluator | Medium | Deepgram nova-2 is the default (best in class for English). Use punctuation + diarization. Fall back to text mode if confidence < 0.7 on 3+ consecutive turns. |
| Sonnet 4.6 cache miss on every turn → cost spike | Low | The system+rubric prefix is byte-identical across turns within the same session; cache hits at 5-min TTL are guaranteed. Cold-start cost is ~$0.02/session, hot is ~$0.001/turn. |
| Browser autoplay policy blocks TTS audio | Low | LiveKit client handles autoplay; require an explicit "Start session" button click before connecting (the gold-standard surface already has this pattern). |
| `MockVoiceController` decays as the production adapter evolves | Medium | Add a Jest test that asserts both the mock and the adapter call the same store mutations with the same shapes. If a slice gains a new mutation, both implementations must add it. |
| Privacy / safety: LiveKit Cloud transit logs the user's voice | Medium | Disable LiveKit Cloud session recording. Voice transit only — no persistence on the LiveKit side. STT transcripts persist to Firestore (mastery trace) but raw audio does not. |

## 8. Out of scope (recap)

- Car-mode Web Speech orchestrator consolidation.
- Voice-first storytelling / study / non-mock-interview surfaces.
- Multi-party voice rooms.
- Voice authentication.
- Live transcription rendering in the UI (the adapter has access to partial transcripts but the gold-standard surface is intentionally minimalist; transcript rendering is a follow-up).

## 9. Codex audit asks

1. **LiveKit Cloud vs self-hosted for Phase 1?** Default proposal: LiveKit Cloud. Cost vs ops trade — confirm or flip.
2. **Deepgram vs Soniox for STT?** Default Deepgram (nova-2). Soniox is comparable but newer. Worth a price + accuracy comparison before committing.
3. **`MockVoiceController` retention strategy?** Keep behind a flag (default proposal) vs delete entirely once production adapter ships. Keeping costs ~400 lines of code; deleting risks losing the visual-proof harness for design iteration.
4. **Agent worker repo placement?** Default: new `kitesforu-voice-worker` repo. Alternative: add to `kitesforu-workers` as another stage (cheaper bootstrap, harder to evolve agent independently).
5. **Should `useCarVoiceOrchestrator` be touched at all?** Default: no (different use case). But voice-mock-on-mobile-while-walking is plausibly an overlap. Confirm whether car-mode → mock-mode session migration is in scope here or a separate proposal.
6. **G-rating refusal at token mint vs at voice surface?** Default: token mint (cleanest). Alternative: surface a friendlier "this isn't available on your plan" UX upstream. Both can ship — token mint is the load-bearing safety guard.

## 10. Rollback

Phase 1 ships behind `feature_voice_mock_production` (default OFF). Flag flip restores the existing voice-preview-only state — the production `/interview-prep/mock/voice` route 404s when the flag is off; the preview route continues to work. Agent worker can be torn down independently (no Firestore schema changes; data-channel events do not persist). Token endpoint is additive.

## 11. Sources

- `kitesforu-frontend/components/voice/MockVoiceController.tsx` — the code being replaced (lines 1-15 docstring confirms LiveKit-adapter destination)
- `kitesforu-frontend/hooks/voice/slices/voiceSessionSlice.ts:5-12` — confirms LiveKit is the documented event source
- `kitesforu-frontend/hooks/voice/useVoiceStore.ts` — store the adapter mutates
- `kitesforu-frontend/hooks/car-mode-v2/useCarVoiceOrchestrator.ts` — INTENTIONALLY untouched
- `kitesforu-frontend/app/interview-prep/mock/voice-preview/page.tsx:14` — confirms "Real LiveKit wiring ships as a follow-up — the components themselves are production code"
- `kitesforu-api/src/api/services/mock_interview/evaluation/prompt.py` — rubric the agent reuses
- `done/R2-phase1-content-quality-voice.md` — phase close-out that named this thread
- `proposed/unit_economics_tracking/README.md` — caching shape that makes <900 ms latency budget achievable
- LiveKit Cloud + Agents framework documentation
