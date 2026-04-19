# 69E54B52 — Car Mode Q&A returns generic answers despite having segment context

**Status**: OPEN
**Reported**: 2026-04-19 by product owner during live drive-test
**Priority**: P1 — precision is the core promise of per-episode Q&A
**Surface**: `/car-mode?topic=...` Quick Question overlay

## Reported symptom (verbatim)

> Since the episode generated was about quicksort, I asked a question about the thing what the episode was talking on that line (which meant it should have been very very precise and specific about that), but it didn't clarify and gave general answer (so maybe it feels, we should have a very quick call giving context about this is the topic, this episode and the current line is X and the current question bla — the podcast could remain paused till then). Lets say there is error or whatever or if the response is delivered then the current episode can resume. This needs lot of proper thinking through and carefully architecting.

## Correct mental model

- When the user asks a question, the backend already knows which segment + position the user is at.
- The LLM should receive a structured context pack: topic + episode outline + **the current line verbatim + surrounding N lines**, plus the question.
- The answer should quote or reference the specific line the user just heard.
- While building the context takes a moment, episode stays paused with a "looking at what you just heard…" state.
- On error or completion, episode resumes.

## Root cause

The LLM sees a one-sentence summary of an entire 3-5 minute segment and zero per-line text. When asked "what about that pivot thing you just said", it has no "line" to quote — so it fabricates a general answer. Specific gaps:

- **Frontend payload** (`hooks/drive/useQuickQuestion.ts:266-270`): sends only `{question, segment_index, position_seconds}`. No line hint.
- **API context assembly** (`kitesforu-api/src/api/services/car_mode/quick_question.py:39-84, 152-167`): loads topic, content_title, outline.points (5 bullets), segment_summaries (one sentence per segment), qa_history. Nothing at line granularity.
- **Worker drops the script** (`kitesforu-workers/src/workers/stages/car_mode/segment_streaming_worker.py:540-582, 921-956`): generates `dialogue: [{speaker, text, emotion}]`, TTS's it, persists only `{segment_index, title, audio_url, duration_seconds, quality, outline_point, summary, bridge_to_next}`. The dialogue array is discarded after TTS.
- **Schema gap** (`kitesforu-schemas .../models.py:1746-1758` `CarModeSegment`): no `dialogue_lines`, no per-line timestamps.

Current system prompt (verbatim from `quick_question.py:68-84`):
```
You are a helpful learning assistant embedded in an audio learning app.
The user is currently listening to an educational audio episode and has a quick question.
Answer concisely in 2-3 sentences. Be direct and informative — the answer will be read aloud.

Today's date: {today}

{context_block}          ← Content/Topic/Outline (7 points)/Summaries (last 3)

RULES:
- Answer in 2-3 sentences maximum. Be crisp and direct.
- Use the episode context to give relevant, specific answers.
...
```

`{context_block}` never contains the user's current line.

## Proposed change set (≤10)

### kitesforu-schemas (1)
1. `models.py:1746` — add `dialogue_lines: List[CarModeDialogueLine] = []` to `CarModeSegment`; define `CarModeDialogueLine(text, speaker, start_s, end_s)` permissive.

### kitesforu-workers (2)
2. `segment_streaming_worker.py:797` — `_tts_dialogue_items` returns `(bytes, per_item_durations)` by measuring each `audio_parts[i]` length.
3. `segment_streaming_worker.py:540-582` + `:921` — persist `dialogue_lines` built from `dialogue[i].text` + cumulative duration offsets.

### kitesforu-api (4)
4. `car_mode/quick_question.py:39` — new `_locate_current_line(segments, segment_index, position_seconds, window=2) → (prev_lines, current_line, next_line)`.
5. `car_mode/quick_question.py:68` — system prompt gets a new `WHAT THE LISTENER JUST HEARD (verbatim, line they're on marked with >>>):` block; add rule: "Quote or reference the marked line when the question is about 'this', 'that', or 'what was just said'."
6. `car_mode/quick_question.py:152` — load `segments` from session, call locator, pass verbatim window into `_build_system_prompt`.
7. `routes/car_mode.py:448` — emit new SSE event `context_loading` before awaiting LLM so frontend can show the pre-answer state.

### kitesforu-frontend (2)
8. `hooks/drive/useQuickQuestion.ts:300` — handle `context_loading` → set `isLookingUp=true`; on `answer_start`, flip to existing thinking state.
9. Q&A panel component — render "Checking what you just heard…" when `isLookingUp`.

### kitetest (1)
10. `apitests/api-client.ts:276` — mirror `dialogue_lines` schema.

## Risks + mitigations

- **Context window blow-up** — only inject current segment ±2 lines (~1KB). Never the full transcript.
- **Stale timestamps** — TTS byte→duration is approximate. If the combiner adds crossfades, offsets drift. Accept ±1s drift; take a 3-line window to absorb.
- **Hot-path vs worker race** (`segment_streaming_worker.py:385-404`): segment-0 written by hot path has no `dialogue_lines` today. Either have hot path write them too, or gracefully degrade.
- **Listen mode** (pre-existing podcasts, `routes/car_mode.py:94-105`): `resolve_existing_content` returns podcast-pipeline segments. Add a separate resolver or degrade to summary-only context.
- **Latency budget** — line lookup is O(log N). `context_loading` SSE event is UI-only, no extra server wait.

## Test plan

- Generate Car Mode episode on "quicksort", let it reach segment 2 at ~45s, ask "what did you just say about the pivot?" — answer must quote the pivot sentence verbatim; Firestore `qa_history[*].question_context` logs the exact line.
- Listen-mode (pre-existing podcast) — graceful degradation: no `dialogue_lines`, no crash, old summary-only prompt path.
- Playwright: fire Q&A while playing, assert panel shows "Checking what you just heard…" before "Thinking…", episode paused through both, resumes on `answer_complete`.

## Files referenced

- `kitesforu-frontend/hooks/drive/useQuickQuestion.ts`
- `kitesforu-api/src/api/routes/car_mode.py`
- `kitesforu-api/src/api/services/car_mode/quick_question.py`
- `kitesforu-api/src/api/services/car_mode/session.py`
- `kitesforu-workers/src/workers/stages/car_mode/segment_streaming_worker.py`
- `kitesforu-schemas/src/kitesforu_schemas/models.py`
