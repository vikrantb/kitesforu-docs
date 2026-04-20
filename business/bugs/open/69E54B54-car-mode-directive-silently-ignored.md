# 69E54B54 — Car Mode "update the episode" directive silently ignored

**Status**: DEPLOYED_NOT_VERIFIED
**Reported**: 2026-04-19 by product owner during live drive-test
**Priority**: P1 — assistant lies by omission (says yes, does nothing)
**Surface**: `/car-mode?topic=...` chat mode (voice orchestrator)
**Fix PRs (4 slices + SSE filter polish, all merged 2026-04-19/20)**:
- Slice 1 — kitesforu-frontend #459: intent classifier + curated patter; directives no longer reach the planner.
- Slice 2 — kitesforu-schemas #68 (`CarModeEditRequest` + `edit_generation`) and kitesforu-api #242 (`POST /v1/car-mode/session/:id/edit`, transactional bump + Pub/Sub publish).
- Slice 3 — kitesforu-workers #291: `execute_regenerate` truncates segments[from_index:], regenerates with hint threaded into previous-summaries, stamps `edit_generation` on every new segment, marks `edit_request.status='applied'` / `'failed'`.
- Slice 4 — kitesforu-frontend #460: wire directive path to the edit endpoint, speak patter immediately, fall back to `CAR_MODE_REGEN_FAILED` on error or 409.
- Polish — kitesforu-frontend #461: client-side `edit_generation` SSE filter. `useDriveSession` tracks `currentEditGenerationRef`; segment_ready events with a lower generation are dropped; higher generation prunes already-buffered episodes ≥ from_index so the regen replaces, not appends. Closes the "stale pre-regen audio briefly plays" window the original deferred list flagged.

**Awaiting**: beta verification on a fresh Car Mode session where the user says a directive like "make this shorter" or "focus on recursion". Move to `closed/` once:
1. The patter line is spoken immediately.
2. A Pub/Sub regen message is visible in worker logs.
3. `edit_request.status` transitions `pending → in_flight → applied` in Firestore.
4. Subsequent episode audio reflects the hint.

**Deferred follow-ups (not blocking closure)**:
- Visible regen progress UI (spinner badge) during the in_flight phase.
- Debounce / coalesce multiple rapid directives into one regen call.
- Q&A answer language parity (same enhancement Car Mode needs broadly — not regen-specific).
- ~~Client-side `edit_generation` filtering on SSE segment events.~~ **Shipped in PR #461 (2026-04-20).**

## Reported symptom (verbatim)

> I asked it, I want to slightly update the episode. The lady said yes it will do it but never changed. It waited for long time but then resumed the same podcast. This was odd. What should happen is the lady should say "we are updating podcast, hold on, bla bla or X Y Z" (some curated stuff) like "will search, recompile, transmute" etc (funky jargony words for fun just to buy us time to create podcast) and then we update the podcast completely under the hood. Now this needs careful planning and thorough job because if the previous state or corruption stays or is there even on the original place, we could see unpredictable results.

## Correct mental model

- User says "make this shorter" / "focus on recursion" / "slightly change to narrative" → this is a DIRECTIVE, not a question.
- Assistant acknowledges with a curated witty patter line ("Okay — recompiling with that lean, give me a sec").
- While patter speaks, backend kicks off regeneration of remaining segments (or current + forward) with the hint threaded into the prompt.
- Current segment's remaining audio + all future unplayed segments get replaced cleanly. No stale-state corruption.
- When regen finishes, episode resumes with new content.
- On failure, original episode resumes with "Couldn't apply that change, staying with the original."

## Root cause

The "yes I'll do it" reply is a pure LLM hallucination with zero side effect.

Exact path:
1. User speech in chat mode → `handleChatMessage` (`app/car-mode/page.tsx:320-327`) → `chat.sendMessage(text)`.
2. `useSmartCreateChat.ts:275-307` POSTs to `${API_BASE}/v1/create/chat/${sessionId}/message`.
3. Lands in `kitesforu-api/src/api/services/smart_create/chat/service.py:42` (`stream_chat_response`) — Smart Create **planner chat**. Knows about the draft plan; has **no binding** to the live `car_mode_sessions/{id}` or the segment streaming worker.
4. LLM, being agreeable, generates "Sure, I'll update it…" text; streamed back; auto-spoken by TTS effect (`app/car-mode/page.tsx:346-361`).
5. **Nothing else happens.** Grep confirms: no `regenerate` / `edit_hint` / `cancel_pending` in `services/smart_create/chat/` or `workers/stages/car_mode/`. `car_mode.py` exposes only `/session`, `/stream`, `/position`, `/recap`, `/question`, `DELETE` — no edit endpoint.

Quick Question path (`hooks/drive/useQuickQuestion.ts:225-355`) also pure RAG — no regen.

## Architecture

```
User speech ──► VoiceOrchestrator (chat mode)
                │
                ▼
   Intent classifier (frontend regex heuristic FIRST, backend LLM fallback)
   ├── question  ─► existing /question flow (unchanged)
   ├── command   ─► existing VoiceCommand switch (unchanged)
   └── directive ─► NEW: carMode.requestEdit({hint, scope})
                        │
                        ├─ 1. drive.pauseEpisode() + setState('regenerating')
                        ├─ 2. pick random patter line, speak()
                        ├─ 3. POST /v1/car-mode/session/:id/edit
                        │      {hint, scope, from_segment_index, from_position_seconds}
                        ▼
              API: stamp Firestore car_mode_sessions/{id}.edit_request = {
                     hint, scope, requested_at, from_index, from_position,
                     edit_generation: N+1  ← monotonic counter
                   }
                   → publish_car_mode_regenerate(session_id, edit_generation)
                        ▼
              Worker: on regen message:
                   - compare edit_generation to session.current_edit_generation
                   - CANCEL in-flight segment tasks (future > from_index)
                   - truncate session.segments to [0..from_index]
                   - regenerate outline + segments from from_index with hint threaded
                     into EpisodeProfile.style_notes / script prompt
                   - stamp each new segment with current gen
                        ▼
              SSE /stream emits segment_ready with edit_generation tag
                        ▼
   Frontend: drop cached audio whose edit_generation < current;
             on segment[from_index] ready, resume playback;
             speak short "Here we go." On failure: restore original segments,
             speak fallback line.
```

## Proposed change set (≤12 files)

1. `kitesforu-frontend/lib/intent-classifier.ts` (NEW) — cheap regex: verbs like "make it / shorter / focus on / change / update / redo / switch to" → directive; `?` / wh-words → question; else command.
2. `kitesforu-frontend/hooks/car-mode-v2/useCarVoiceOrchestrator.ts` — route `handleChatMessage` through `classifyIntent`; directives go to `carMode.requestEdit`, not `chat.sendMessage`.
3. `kitesforu-frontend/hooks/car-mode-v2/useCarModeEdit.ts` (NEW) — owns `state: 'idle'|'regenerating'|'applying'|'error'`, patter rotation, pause-on-request, resume-on-ready, SSE edit_generation tracking.
4. `kitesforu-frontend/hooks/useDriveMode.ts` — `currentEditGeneration` ref; drop SSE `segment_ready` whose `edit_generation < current`; invalidate cached audio for `segment_index >= from_index`.
5. `kitesforu-frontend/lib/car-mode-patter.ts` (NEW) — patter list + rotating picker (no repeat within session).
6. `kitesforu-frontend/app/car-mode/page.tsx` — wire `useCarModeEdit`; show regen badge; suppress chat auto-TTS while regenerating.
7. `kitesforu-api/src/api/routes/car_mode.py` — `POST /v1/car-mode/session/{id}/edit` body `{hint: str, scope: Literal["rest","current"], from_segment_index: int, from_position_seconds: float}`, rate limit `3/minute`.
8. `kitesforu-api/src/api/services/car_mode/edit.py` (NEW) — stamp `edit_request` + bump `edit_generation` atomically; call `publish_car_mode_regenerate`.
9. `kitesforu-api/src/api/services/car_mode/pubsub.py` — `publish_car_mode_regenerate(session_id, edit_generation, hint, scope, from_index)`.
10. `kitesforu-schemas .../car_mode.py` — `CarModeEditRequest`, `CarModeEditGeneration` (`ConfigDict(extra='ignore')`); add `edit_generation: int = 0` to `CarModeSegment`.
11. `kitesforu-workers .../segment_streaming_worker.py` — handle `message.attributes['op'] == 'regenerate'`: read `edit_generation`, abort in-flight tasks keyed by stale gen, truncate `segments[from_index:]`, re-run discovery + script with `user_hint` threaded, stamp each new segment.
12. `kitesforu-api/src/api/routes/car_mode.py` (SSE stream) — include `edit_generation` in `segment_ready`; emit `edit_started` / `edit_failed` from Firestore `edit_request.status`.

## Curated patter list

1. "On it — recompiling with that lean, give me a sec."
2. "Got it. Re-threading the takes now."
3. "Transmuting the pitch — one moment."
4. "Rewriting, just a sec. Don't go anywhere."
5. "Okay, remixing with that angle — breathe out once."
6. "Heard. Reshaping the story — won't take long."
7. "Nice note. Re-cutting the tape."
8. "Tuning the dials — back in a blink."

Rotate without repeating within last 2 uses; TTS-safe (no jargon that mis-pronounces); each ≤8 words.

## Risks + mitigations

- **LLM misclassifies question as directive** — regex heuristic first (high-precision directive verbs); ambiguous cases route to chat with soft confirm ("Sounds like you want a change — yes?"). Log classifications.
- **Regen cost / latency** — scope defaults to `rest` (remaining segments only). Rate limit `3/min`. 409 if regen in flight; speak "I'm still applying the last change."
- **Race with playing audio** — pause BEFORE POST (keeps user-gesture window on iOS). Patter uses bridge audio; never overlaps episode.
- **Stale SSE post-regen** — `edit_generation` guard at frontend + worker; older gen is dropped.
- **Regen failure mid-flight** — worker sets `edit_request.status = 'failed'`; frontend restores original segments (never deleted; only `segments[from_index:]` truncated in a shadow field), resumes, speaks failure line.
- **User mashes multiple directives** — debounce 1.5s; coalesce hints ("shorter" + "focus on recursion" both thread into prompt).

## Test plan

- Live: 3-segment episode, at segment 1 say "make it shorter and focus on recursion" → verify patter speaks, episode pauses, `edit_request.edit_generation` increments, worker logs abort of pending segments, new `segment_ready` carries new gen, playback resumes with visibly shorter/recursion-heavy content.
- Unit: `intent-classifier.test.ts` — 30 phrasings (15 directive / 10 question / 5 ambiguous).
- Worker: simulate Pub/Sub regen mid-generation, assert task cancellation + segment truncation.
- Failure: inject worker exception during regen, assert frontend restores originals + speaks failure line, no double-play.

## Files referenced

- `kitesforu-frontend/app/car-mode/page.tsx`
- `kitesforu-frontend/hooks/useSmartCreateChat.ts`
- `kitesforu-frontend/hooks/drive/useQuickQuestion.ts`
- `kitesforu-api/src/api/routes/car_mode.py`
- `kitesforu-api/src/api/services/smart_create/chat/service.py`
- `kitesforu-workers/src/workers/stages/car_mode/segment_streaming_worker.py`
