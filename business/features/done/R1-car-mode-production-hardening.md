# R1 — Car Mode Production Hardening

**Status**: SUBSTANTIALLY DONE — P1 (UX) and P2 (course adjustment) shipped; P3 (backend pipeline parity) mostly shipped; P4 (polish) has remainders
**Priority**: P0 (core differentiator has user-visible regressions + architectural debt)
**Effort**: shipped in ~1 day of dense PR work (dramatically shorter than the original ~3-week estimate — most phases were scoped tightly and the foundation was more solid than the audit suggested)
**Origin**: 2026-04-19 audit — four parallel research agents on code, integration, and live UI; plus user-reported regression.

## Implementation summary (2026-04-19)

### ✅ P1 — Q&A feedback loop
- **kitesforu-frontend #451** — phantom Commands-mode transcript no longer rendered as a question; rapid pause/resume uses live `audio.paused` instead of React state mirror; ask-button is a proper toggle (dismiss on second click); `speechSynthesis.cancel()` in cleanup only fires when we own the synth; gesture-label setTimeout leak closed.
- **kitesforu-frontend #452** — Q&A 429 retry with exponential backoff; iOS auto-resume via player toggle on auto-dismiss; 4s auto-clear on stale `lastTranscript` / `lastCommand`.
- **kitesforu-frontend #453** — **Q&A answers are now spoken**. `useEffect` watches `drive.quickQuestion.answer` + `isThinking`; speaks via orchestrator once SSE stream completes (suggestions populated). Capped at 3 sentences to bound episode-pause time.
- **kitesforu-frontend #454** — debounce Q&A speak until SSE stream completes using `suggestions.length > 0` as the "done" signal (prevents mid-stream stutter).
- **kitesforu-frontend #455** — hide phantom voice-commands mic button when the inner `useVoiceCommands` hook is disabled (Car Mode owns voice via orchestrator).
- **kitesforu-frontend #456** — bug 69E54B51: **pause-on-assistant-speak, not on user-talk-start**. Orchestrator now exposes `onSpeakStart` / `onSpeakEnd` callbacks; Car Mode page wires them to `drive.toggle` with a `pausedByOrchestratorRef` guard so the episode only resumes if we paused it.

### ✅ P2 — Course adjustment from questions (bug 69E54B54, 4 slices)
- **kitesforu-frontend #459** — intent classifier + curated patter rotation; directives no longer hit the planner.
- **kitesforu-schemas #68** (v1.48.0) — `CarModeEditRequest` + `edit_generation` + `CarModeSession.current_edit_generation`.
- **kitesforu-api #242** — `POST /v1/car-mode/session/:id/edit` with Firestore transaction + Pub/Sub publish. Rate limit 3/min. 409 on concurrent edits.
- **kitesforu-workers #291** — `execute_regenerate` truncates segments[from_index:], regenerates with user hint in previous_summaries, stamps each new segment with `edit_generation`, lifecycles `edit_request.status` through pending → in_flight → applied / failed.
- **kitesforu-frontend #460** — directive path calls the edit endpoint, speaks patter immediately, falls back to failure patter on 409 / error.

### ✅ P3 — Backend pipeline parity (substantially shipped)
- **kitesforu-schemas #67** (v1.47.0) — `CarModeDialogueLine` + `CarModeSegment.dialogue` field.
- **kitesforu-workers #290** — worker persists per-line dialogue to every segment (bug 69E54B52 foundation).
- **kitesforu-api #241** — Q&A prompt now includes a `WHAT THE LISTENER JUST HEARD` verbatim dialogue block + rule telling the LLM to quote the exact line.
- **kitesforu-workers #292** — **Car Mode persona wiring**: `_select_persona_voice` runs once per session, cached on SegmentState; all three TTS call sites thread `persona_voice_config` so Car Mode finally picks up the 20-persona voice catalog + tier-aware cost gates (Inworld / Google for free tier; ElevenLabs only for paid).

### ✅ P4 — UX hardening
- **kitesforu-frontend #450** — persona-card solid-dark base (switch from `/10` alpha that read as pastel) + 10 adjacent pastel-50 surface sweeps.
- **kitesforu-frontend #457** — bug 69E54CAF: dark-mode toggle is now a proper 3-state cycle (light → dark → system → light) so the OS sync default is reachable after the first click.

### Deferred follow-ups (not blocking "done" closure)
- Frontend-side `edit_generation` SSE filtering — without it, a stale pre-regen segment can briefly play after a directive. Acceptable for v1. Tracked in bug 69E54B54's deferred list.
- Visible regen progress UI (spinner during `in_flight`).
- Per-line Q&A context with timestamps + "checking what you just heard…" state. Shipped segment-level precision in #241 / #290; per-line is a follow-up.
- Debounce multiple rapid directives (coalesce hints).
- Language parity for Q&A answer TTS — today answers still speak via browser `speechSynthesis` in default locale. Episode-voice backend TTS for answers is a future enhancement.

### Verification status
- All PRs merged.
- Beta revisions live since 2026-04-19.
- Bugs `69E54B51` + `69E54CAF` moved to `bugs/closed/`.
- Bugs `69E54B52` + `69E54B54` in `bugs/open/` with status `DEPLOYED_NOT_VERIFIED` — awaiting live listen-through confirmation before moving to `closed/`.

---


## Problem

Car Mode / Drive is documented in `business/features/done/car-mode-drive.md` as shipped. In practice, the live flow on beta has multiple user-visible regressions AND ships with an entirely separate backend pipeline that never received the 90+ PRs of voice intelligence, audio naturalness, and script craft work from April 2026.

User-reported symptoms (2026-04-19):
- Mic transcript is displayed ("horrible environment" rendered on screen) but no audible follow-up answer is produced.
- Course does not adjust based on user questions.
- Generation state shows "1/5 episodes ready" with no progress indicator.

## Root causes (evidence-based)

### A. Q&A flow is misleading by design

1. **Commands-mode mic is always listening and shows raw transcripts.** The on-screen italic text ("horrible environment") is a *Commands-mode* ambient capture, not a submitted question. Agent 4 independently reproduced this: "Phantom recognition strings persist in the Commands panel when no active Ask session." Commands only acts on hotwords (pause/next/etc.); everything else is displayed with no routing.
2. **Quick Question (dedicated Q&A button) is a side-channel chat, NOT a course-adjustment loop.** `POST /v1/car-mode/session/{sessionId}/question` calls OpenAI `gpt-4.1-mini` via `services/car_mode/quick_question.py:122` and streams back TEXT + follow-up suggestions. No hook back into segment regeneration.
3. **Q&A answer TTS is browser `speechSynthesis`, not the episode voice.** `hooks/car-mode-v2/useCarVoiceOrchestrator.ts:194` speaks the answer in the browser's default locale (hardcoded `en-US` recognition; TTS inherits OS default). No persona, no language parity, no error path if `speechSynthesis` is blocked — silent failure is indistinguishable from "I wasn't heard."

### B. Car Mode is an orphan backend pipeline

4. **No persona threading.** `segment_streaming_worker.py:822` `_tts_dialogue_items` calls `generate_multi_speaker_audio(dialogue=[item], model=tts_model, parallel=False, language=language, user_tier=tier)` — omits `persona_voice_config`. Contrast with `streaming_script_audio_worker` which reads `preferences["_persona_voice_config"]` and threads it through the full TTS chain. None of the 20 named personas (Vincent Graves, Dr. Sarah Chen, etc.) are ever selectable in Car Mode.
5. **No EpisodeProfile / genre-aware prompts.** Worker uses fixed prompts `QUICK_SCRIPT_PROMPT` and `FULL_SEGMENT_PROMPT` (`segment_streaming_worker.py:110-186`). Never calls the content_discovery pipeline, never resolves a genre profile, never uses the composable prompt system. Per workers rule #8, Car Mode is a THIRD isolated code path, so every April quality PR bypasses it.
6. **No naturalness validator / variable pauses / tension curves.** All of PR #226 (audio naturalness system) and follow-ups are inapplicable to Car Mode output.

### C. Streaming seams are fragile

7. **SSE is Firestore polling in disguise.** `api/routes/car_mode.py:229,307` polls `get_session()` every 1s for up to 10 minutes. Not true streaming; ~600 Firestore reads per minute of generation. Cost + divergence risk vs. Studio's real SSE.
8. **Divergent SSE event shape.** Car Mode uses `{type:"segment_ready", segment:{segment_index}}`; Studio uses `{type:"segment_ready", ..., index}`. Both work today but drift silently — any future consolidation will break one.
9. **Phantom "hot path" in the worker.** `segment_streaming_worker.py:382-404` checks for a pre-existing segment-0 claimed to be written by an "API hot path." Grep of `kitesforu-api/services/car_mode/` shows no such code exists. Advertised "<3s to first audio" is actually worker cold-start + Phase 1 ≈ 12s.
10. **Worker stall invisible for 10 min.** If Phase 2 fails after Phase 1 succeeds, session status stays `generating` until `max_empty_polls` (10 min) fires. Frontend sits on segment-0 with no indicator.
11. **Audio element errors never propagate.** A 403 from GCS or network drop on a single segment yields a silent stall; `_update_position` keeps persisting a position no user is actually hearing.

### D. Frontend state and lifecycle bugs

12. **Rapid pause/resume seeks to start.** Agent 4 reproduced: 5 pause/play toggles ~100ms apart rewound a 0:32 position to 0:02.
13. **Ask-a-question toggle desync.** Button keeps active styling + "Quick Question" label after being deactivated — user cannot tell whether Q&A mode is on.
14. **Mic-denied is a silent dead-end.** With `getUserMedia` set to `not-allowed`, the Ask button still flips to active with no error, no typed-input fallback.
15. **No progress indicator during generation.** On Slow 3G, UI shows "0/5 episodes ready" with no spinner / elapsed time / error path — indistinguishable from a broken session.
16. **`resumeEpisode` bound to `player.toggle()`.** `hooks/useDriveMode.ts:155-156`. Depends on `pausedByQQRef` guards to resume correctly; if the guard fails, toggle pauses a playing episode.
17. **Auto-dismiss does not resume on iOS Safari.** `hooks/drive/useQuickQuestion.ts:176-187` flips `pausedByQQRef.current = false` inside the countdown; iOS autoplay policy blocks `.play()` from `setTimeout`, leaving the episode silently paused forever until the user taps play.
18. **Array-index episode resume in playback mode.** `hooks/drive/useDriveAutoPlay.ts:121` — live mode correctly uses `find(ep.id === id)` but playback mode uses `episodes[idx]`. If episodes reorder, the wrong one plays.
19. **speechSynthesis.cancel() in unmount cleanup violates bridge contract.** `hooks/car-mode-v2/useCarVoiceOrchestrator.ts:287-288` cancels unconditionally — frontend rule explicitly forbids this when bridge has claimed audio.
20. **setTimeout leaks in gesture labels.** `components/drive/DriveMode.tsx:81-104` spawns timers without refs / cleanup. Rapid double-taps leak.
21. **Hardcoded `en-US` STT.** `useCarVoiceOrchestrator.ts:119` and `useQuickQuestion.ts:371`. Frontend never sends a `language` field in the create-session body, so backend `SupportedLanguage` resolution is never exercised for Car Mode.
22. **Q&A rate limit 10/min with no backoff/retry.** `api/routes/car_mode.py:425` + `hooks/drive/useQuickQuestion.ts:312` — first 429 surfaces as "Sorry, something went wrong," no retry.
23. **Car Mode page never starts bridge audio.** `app/car-mode/page.tsx:202` calls `useDriveMode` without `prepareDriveBridge`/`startBridge`, so the flow has a 10–15s silent dead zone before first worker segment. Only `/create-smart → Drive` kicks off bridge music.

## Proposal — four phases

### Phase 1 — Make Q&A actually answer the user (P0, ~1 week)

The user's most immediate pain: "I talk, nothing answers me back." Fix the feedback loop without changing product surface area.

- **Surface Commands vs. Quick Question clearly.** Commands mode mic is always-on but should NOT display a rolling transcript as if it were a question. Only display a transcript when the dedicated Quick Question button is pressed. Never render ambient voice command candidates as chat.
- **Add "Answer incoming…" + "Answer ready" states.** Visible chrome between "I heard you" and "audio starts playing" so silence is never the signal.
- **Add typed-input fallback** when `getUserMedia` permission is denied OR when `speechSynthesis` is unavailable. A keyboard-available text box inside the Quick Question overlay is acceptable.
- **Language parity for answer TTS.** Pass session language from frontend → backend → answer TTS. If the episode is Hindi, answer speaks Hindi. Until backend TTS is wired (Phase 3), at minimum pass the correct locale to `SpeechSynthesisUtterance.lang` and pick a voice that supports it.
- **Fix auto-dismiss resume on iOS Safari.** Do not mark `pausedByQQRef.current = false` inside the countdown. Require an explicit user gesture to resume, or attempt `.play()` and gracefully show "Tap to resume" if autoplay rejects.
- **Wire retry + backoff for 429 on Q&A endpoint.** Exponential backoff up to 3 attempts. Bump limit to `30/min`.

### Phase 2 — Course adjustment from questions (NEW product capability, ~1 week)

Answering the user's second observation: "no tweak of the course." Today this is an architectural gap, not a bug; Q&A is a side-channel chat and segment generation is one-shot.

- **User-hinted regeneration of remaining segments.** When user asks a follow-up that implies content change ("go deeper on X", "skip Y", "add example about Z"), append a hint to the remaining-segments prompt and regenerate them non-streamingly.
- **Intent classification at Q&A time.** `stream_quick_answer` already returns follow-up suggestions — add an `intent: "question" | "hint"` field. If `hint`, trigger the regen path; if `question`, answer as today.
- **Thread user hints into EpisodeProfile `user_directions: list[str]`.** Surface "Adjusting based on your question…" to the user during regen.
- **Hook into R2 quality auto-regen** (see `R2-quality-auto-regeneration.md`). The bad-path-serialize wire-in applies symmetrically: cancel in-flight TTS for remaining segments, regenerate script, re-run TTS. Car Mode becomes the second consumer of that loop, not a separate one.

### Phase 3 — Backend pipeline parity (~1 week)

Close the orphan-pipeline debt. Car Mode output should feel identical in quality to Studio output.

- **Thread `persona_voice_config` through `segment_streaming_worker`.** Run persona selection once per job; cache on `preferences["_persona_voice_config"]`; pass to `_tts_dialogue_items`. Mirror the exact contract `streaming_script_audio_worker` uses.
- **Use EpisodeProfile / genre detection.** Call `content_discovery` at session creation; attach the returned profile to the session document; consume in the worker's prompt composition.
- **Use the composable prompt system** (`PromptComposer`) for full segments, not the hardcoded `FULL_SEGMENT_PROMPT`. Quick Preview (Phase 1 output) can stay minimal, but full episodes should match Studio.
- **Backend TTS for Q&A answers.** Replace `speechSynthesis` with the same provider pipeline used for episode TTS so persona + language parity is automatic. Cache answer audio for 10 min so retries don't re-TTS.

### Phase 4 — UX hardening + observability (~3 days)

- Fix rapid pause/resume seek-to-start (debounce / idempotency on `player.toggle`).
- Fix Ask-a-question toggle state desync (unify button state with app state via a single `isQuickQuestionActive` source of truth).
- Fix setTimeout leaks in `DriveMode` gesture labels (refs + cleanup).
- Fix `speechSynthesis.cancel()` violation in `useCarVoiceOrchestrator` unmount (skip cancel when bridge is claimed).
- Fix array-index resume in `useDriveAutoPlay` playback mode (use `.find(ep.id === id)`).
- Remove phantom hot-path check in `segment_streaming_worker.py:382-404` — either implement the API hot path or delete the dead branch.
- Add stall detection: emit `{type:"slow_generation"}` SSE event if no new segment in 60s. Frontend shows a progress toast.
- Add `<audio>` onError → POST `/session/{id}/playback-error` so server can mark the segment bad and regenerate.
- Replace the 1s Firestore poll in Car Mode SSE with real-time listeners OR converge onto Studio's SSE dialect. Pick one.
- Bridge audio on entry: `app/car-mode/page.tsx` should call `startBridge` so the 10–15s wait for first segment is not silent.

## Acceptance criteria

- [ ] Phase 1: typed-input fallback works when mic denied; answer audio plays in session's language; 429s retry; no ambient transcripts rendered without active Q&A
- [ ] Phase 2: user saying "go deeper on recursion" triggers regen of remaining segments with a visible "adjusting…" state; text-only questions still route to fast OpenAI answer
- [ ] Phase 3: Car Mode episode quality matches Studio on the voice intelligence test suite; language parity across episode + Q&A; no duplicate prompt code
- [ ] Phase 4: all 12 listed bugs closed with regression tests where applicable; stall detected within 60s; Firestore read count per session drops ≥80%
- [ ] Playwright E2E: sign in → start car mode → trigger Quick Question → receive audio answer → ask directive "skip next episode" → observe regen → verify final audio uses persona system
- [ ] Beta verify per workers rule #11 after every phase deploy

## Key file references

Frontend:
- `app/car-mode/page.tsx:202` — entry point, missing `startBridge`
- `components/drive/DriveMode.tsx:81-104` — setTimeout leaks
- `hooks/car-mode-v2/useCarVoiceOrchestrator.ts:119, 194, 250-268, 287-288` — STT/TTS, mode sync, cleanup
- `hooks/drive/useQuickQuestion.ts:176-187, 231, 248, 312, 371, 466-478` — Q&A flow, auto-dismiss, retry
- `hooks/drive/useDriveSession.ts:117, 121, 175, 190-193, 296-303` — SSE, language body omission
- `hooks/drive/useDriveAutoPlay.ts:121` — array-index resume
- `hooks/useDriveMode.ts:155-156` — resumeEpisode bound to toggle
- `hooks/drive/useDriveContent.ts:63-66, 138-162` — poll timer cleanup
- `lib/drive-audio-bridge.ts` — bridge claim/release

API:
- `api/routes/car_mode.py:67, 202, 229, 255, 307, 424-425, 284-293` — create session, SSE, Q&A, rate limits
- `services/car_mode/session.py:44` — `current_segment_index` default
- `services/car_mode/quick_question.py:60, 122` — question handler, segment summary slice

Workers:
- `src/workers/stages/car_mode/segment_streaming_worker.py:110-186, 347, 382-404, 822` — prompts, failure, hot path, TTS call without persona

## Out of scope (explicit)

- Rebuilding Car Mode as a Studio wrapper (larger refactor; deferred).
- iOS native wrapper / CarPlay / Android Auto (separate platform scope).
- Offline Car Mode playback (overlaps with R1 P2 D5 Service Worker scope).
- Multi-speaker Q&A answer (single voice is acceptable for v1).
- Sharing Q&A transcripts.
