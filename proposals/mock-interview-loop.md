# Mock Interview Loop — Architecture Proposal

**Status:** Draft v1
**Author:** Engineering (with Deep Research on voice turn-taking, LLM-as-judge, and adaptive difficulty)
**Date:** 2026-04-12
**Target:** Pro tier first, voice-first interactive mock interview with between-question feedback

---

## 1. Scope

### What this is
A new product line inside KitesForU: an interactive voice mock interview where the candidate speaks their answers, receives structured between-question feedback, and the system adapts question difficulty to keep them in flow. It is **not** an extension of the current coaching-podcast product; it is a new pipeline that reuses the existing TTS/STT providers and parts of the Car Mode audio state machine.

### What this is NOT (v1)
- Video analysis (posture, eye contact, facial expression) — Phase 3 at earliest
- In-answer real-time feedback (interrupting the user mid-sentence) — explicitly rejected; research shows it hurts both flow and performance
- Multi-interviewer panel simulation — Phase 2
- Recruiter-side analytics / team dashboards — B2B feature, Phase 3
- Non-English languages — v1 ships English only; reuses existing `SupportedLanguage` mechanism for Phase 2

### Primary success metric
**Completion-weighted improvement**: percentage of users whose rubric score improves by ≥0.5 points (on 5-point scale) between session 1 and session 3, among users who complete all three sessions.

### Guardrails
- Cost per 15-minute session: ≤ $0.40 (Pro tier economics)
- Latency p50: < 900ms mic-silence → first TTS byte
- Latency p95: < 1500ms; alert at p95 > 2s
- Zero mid-answer interruptions of the user

---

## 2. User Journey (15-minute Pro session)

```
0:00  Landing → /interview-prep/mock/new
       - Declare role, seniority (cold-start Elo seed)
       - Optional: upload/paste JD for company-specific calibration
       - Pick focus area: behavioral | system design | technical | mixed

0:30  Mic check + calibration
       - Browser getUserMedia, AEC on, Silero VAD warm-up
       - 3-second "say hello" to tune endpointing threshold

0:45  Q1 begins
       - TTS plays question (ElevenLabs Flash v2.5, ~75ms TTFB)
       - "Listening…" visual backchannel (pulsing mic indicator)
       - User answers
       - VAD + turn-detector endpoint → eval stage

[per-turn budget: TTS-out + think + STT + eval + TTS-in ~= 90-120s typical]

  ↓
  Between-question feedback card (silent, ~3s read time)
    - Weighted score (1-5)
    - STAR gate indicator [S✓ T✓ A✓ R✗]
    - One concrete fix ("No numeric result. Try: 'cut latency from 2s to 400ms.'")
    - Rewrite of weakest sentence
  ↓

12:00  Session ends after ~8 questions OR 15 min OR frustration ladder terminal state
       - End-of-session synthesis report (generated async, arrives in 10-15s)
       - Trend across questions per rubric dimension
       - Top 3 drill-down patterns
       - Next-session recommendation (saved to user profile)
```

Between-question feedback is visual + silent. The audio channel stays clear so the candidate can mentally reset. The interviewer voice resumes with the next question.

---

## 3. System Architecture

```
Browser (Next.js, LiveKit SDK)
  ├─ LiveKit room (WebRTC full-duplex)
  ├─ Mic track → VAD (Silero local) → turn-detector (LiveKit)
  └─ Audio playback → AEC via getUserMedia(echoCancellation:true)
         │
         ▼
LiveKit Cloud (WebRTC SFU)
         │
         ▼
kitesforu-interview-worker  [NEW Cloud Run service]
  ├─ Session state machine (Pydantic)
  ├─ STT adapter (gpt-4o-mini-transcribe streaming)
  ├─ Question selector (Elo + 6-axis tagger)
  ├─ Rubric judge (structured LLM call with logprobs)
  ├─ TTS adapter (ElevenLabs Flash v2.5 primary, Inworld fallback)
  └─ Telemetry writer (Firestore + Cloud Logging)
         │
         ▼
kitesforu-api (extended)
  ├─ POST /v1/interview-prep/mock/sessions      (create, cold-start)
  ├─ GET  /v1/interview-prep/mock/sessions/:id  (resume, read-only)
  ├─ POST /v1/interview-prep/mock/sessions/:id/end (force-end + report)
  └─ GET  /v1/interview-prep/mock/questions (admin, question bank CRUD)
         │
         ▼
Firestore collections:
  - mock_interview_sessions
  - mock_interview_turns (subcollection per session)
  - mock_interview_questions (the tagged question bank)
  - mock_interview_user_state (per-user Elo + history)
```

**Why a new worker, not an extension of `kitesforu-worker-audio`**: the podcast audio worker is optimized for batch generation. Mock interviews are stateful, long-lived sessions with conversational turn budgets; mixing them forces one to compromise. Separate service, separate scaling knobs, shared schemas package.

**Why LiveKit and not raw WebSocket**: per voice turn-taking research, raw WebSocket fights you on AEC and barge-in timing. LiveKit provides full-duplex, handles RTC negotiation, ships Silero VAD + turn-detector as first-class components, and has a Python Agents SDK that pairs with our FastAPI stack.

---

## 4. Data Model

### Firestore collections

#### `mock_interview_sessions` (top-level)
```python
class MockInterviewSession(BaseModel):
    model_config = ConfigDict(extra='ignore')  # per Pydantic-Firestore rule
    session_id: str
    user_id: str
    tier: Literal["trial", "pro", "ultimate"]
    status: Literal["created", "active", "completed", "abandoned", "failed"]
    focus_area: Literal["behavioral", "system_design", "technical", "mixed"]
    role: str                    # "Senior Software Engineer"
    seniority: Literal["junior", "mid", "senior", "staff"]
    starting_elo: int            # seeded from seniority
    current_elo: int             # updated after each turn
    jd_context: Optional[str] = None
    target_duration_min: int = 15
    questions_planned: int = 8
    questions_asked: int = 0
    created_at: datetime
    started_at: Optional[datetime] = None
    ended_at: Optional[datetime] = None
    end_reason: Optional[Literal[
        "user_ended", "completed", "frustration_cap", "time_cap", "error"
    ]] = None
    end_report_status: Literal["pending", "ready", "failed"] = "pending"
    end_report_url: Optional[str] = None  # GCS path
```

#### `mock_interview_sessions/{id}/turns` (subcollection)
```python
class MockInterviewTurn(BaseModel):
    model_config = ConfigDict(extra='ignore')
    turn_id: str
    turn_index: int                        # 0-based
    question_id: str
    question_text: str                     # denormalized for replay
    question_difficulty_elo: int
    question_tags: Dict[str, int]          # 6-axis snapshot
    asked_at: datetime
    user_answer_transcript: str = ""       # STT final
    partial_transcripts: List[str] = Field(default_factory=list)  # for replay
    answer_started_at: Optional[datetime] = None
    answer_ended_at: Optional[datetime] = None
    answer_duration_ms: Optional[int] = None
    endpoint_type: Literal["vad_timeout", "turn_detector", "barge_in"] = "vad_timeout"
    # Rubric scoring (filled by judge)
    star_gate: Dict[str, bool] = Field(default_factory=dict)  # {S,T,A,R}
    scores: Dict[str, float] = Field(default_factory=dict)    # 5 dimensions
    weighted_score: Optional[float] = None
    top_fix: Optional[str] = None
    rewrite: Optional[str] = None
    judge_model: Optional[str] = None
    judge_ensemble_disagreement: Optional[float] = None       # for calibration
    # Elo update
    elo_before: Optional[int] = None
    elo_after: Optional[int] = None
    elo_delta: Optional[int] = None
```

#### `mock_interview_questions` (top-level question bank)
```python
class MockInterviewQuestion(BaseModel):
    model_config = ConfigDict(extra='ignore')
    question_id: str
    text: str
    focus_area: Literal["behavioral", "system_design", "technical", "mixed"]
    role_tags: List[str]            # ["software_engineer", "product_manager"]
    seniority_min: Literal["junior", "mid", "senior", "staff"]
    seniority_max: Literal["junior", "mid", "senior", "staff"]
    # 6-axis difficulty per Agent 3
    axis_conceptual_depth: int       # 1-5
    axis_domain_specificity: int     # 1-5
    axis_ambiguity: int              # 1-5
    axis_follow_up_pressure: int     # 0-3
    axis_structure_load: int         # 1-5
    axis_time_boxed_complexity: int  # 1-5
    # Elo seed (computed from axes, drifts with usage data)
    elo_rating: int
    elo_responses: int = 0           # count of scored answers against this Q
    ideal_answer_outline: Optional[str] = None  # for judge few-shot anchor
    anchor_example_strong: Optional[str] = None
    anchor_example_weak: Optional[str] = None
    company_context: Optional[str] = None  # e.g., "amazon_lp_customer_obsession"
    is_active: bool = True
```

#### `mock_interview_user_state` (doc per user)
```python
class MockInterviewUserState(BaseModel):
    model_config = ConfigDict(extra='ignore')
    user_id: str
    current_elo: int                            # drifts across sessions
    sessions_completed: int = 0
    questions_seen: Dict[str, int] = Field(default_factory=dict)  # recency penalty
    weak_axes: Dict[str, float] = Field(default_factory=dict)     # avg scores per dim
    last_session_at: Optional[datetime] = None
    last_session_report_path: Optional[str] = None
```

All data models use `ConfigDict(extra='ignore')` per the Pydantic-Firestore boundary rule in `.claude/knowledge/engineering-wisdom.md`. No `max_length` on any `List` field. Request models (for `POST /sessions`) will be defined separately and can be strict.

---

## 5. API Surface

```
POST /v1/interview-prep/mock/sessions
  Body: CreateMockSessionRequest
    - role: str (required)
    - seniority: Literal[...] (required)
    - focus_area: Literal[...] (default "mixed")
    - jd_text: Optional[str] (optional, max 10KB)
    - target_duration_min: int (default 15)
  Response: { session_id, livekit_room_name, livekit_token, starting_elo }
  Credits: deducted pre-session (15min × pro_multiplier)

GET /v1/interview-prep/mock/sessions/{session_id}
  Returns session + turns (denormalized)
  Used for review mode after session ends

POST /v1/interview-prep/mock/sessions/{session_id}/end
  Force-end session, trigger end-of-session report generation
  Report generation is async; client polls or receives SSE

GET /v1/interview-prep/mock/sessions/{session_id}/report
  Returns end-of-session synthesis (polled after /end)

GET /v1/interview-prep/mock/questions  (admin only)
POST /v1/interview-prep/mock/questions  (admin only)
  Question bank CRUD
```

Static routes before path parameters (per FastAPI gotcha in engineering-wisdom.md). SlowAPI body params not named `request`.

---

## 6. Voice Pipeline (per-turn)

### Latency budget: p50 < 900ms, p95 < 1500ms

| Stage | Target (ms) | Component |
|-------|------------|-----------|
| User stops speaking | 0 | — |
| VAD silence detection | 150-300 | Silero VAD, 800ms silence threshold for interview mode |
| Semantic endpoint confirmation | +30-50 | LiveKit turn-detector model (rejects mid-thought cuts) |
| STT finalization | +100-200 | `gpt-4o-mini-transcribe` streaming (partial transcripts committed every 200-300ms) |
| Judge LLM first token | +300-500 | Rubric prompt with logprobs; pre-streamed context |
| Next-question selection | +50 | Elo lookup + recency filter |
| Question TTS first byte | +75-300 | ElevenLabs Flash v2.5 (~75ms TTFB) or Inworld fallback |
| **Total p50 target** | **~900** | |

### Critical design choices (from voice research)

1. **VAD silence threshold: 800-900ms** (vs 400-500ms for casual chat). False cuts in an interview are catastrophic — users need thinking pauses.
2. **Endpoint is VAD + turn-detector, not VAD alone.** Silero gates on energy; turn-detector makes the semantic "are they done" call. Stacking reduces false cuts ~40% on thoughtful audio.
3. **Filler tolerance**: if last spoken token is "um/uh/so/like/you know", extend VAD silence by 400ms before endpointing.
4. **Barge-in grace window: 400-500ms**. Short utterances (<500ms) during TTS playback are treated as backchannels (cough, "uh-huh") and do NOT cancel TTS. Anything longer triggers barge-in: flush TTS buffer, cancel inflight LLM generation, feed truncated assistant turn back to context ("interviewer was saying: '...' [interrupted]").
5. **Visual backchannels only in v1**. Pulsing mic indicator, "listening…" caption, subtle nodding avatar. No audio "mm-hmm" — research shows it's high-risk to implement without prosody-aware TTS (Hume-level) and easy to have it sound creepy.
6. **Streaming everything**. STT emits partial transcripts continuously; judge LLM begins pre-filling context before the user finishes speaking; TTS starts on first complete clause of the next question, not first complete question.
7. **Truncation-aware context**: when user barges in, the partial AI turn that actually played is what gets added to context, not the planned full turn.

### Failure recovery
- If judge LLM call exceeds 1500ms, use a cached generic acknowledgment ("Got it — next question") while the scoring finishes async.
- If user speech resumes within 500ms of AI response start, treat the AI response as a failed turn, stop immediately, wait.
- If STT returns empty transcript 2x in a row, check mic permission and prompt user.

---

## 7. Evaluation Rubric

### Gate + 5 scored dimensions (from LLM-as-judge research)

**Binary gate (categorical, not scored):**
```
STAR presence: {S_present, T_present, A_present, R_present}
```

**5 dimensions, 1-5 each, weighted:**
```
specificity          weight 0.25   named entities, concrete nouns, hedge penalty
quantified_impact    weight 0.25   numeric result in R (regex + LLM check)
ownership            weight 0.15   I-verb ratio vs we-verb in Action span
role_outcome_clarity weight 0.20   explicit role + outcome scope
level_appropriate    weight 0.15   calibrated vs declared seniority
weighted_total = Σ(dim × weight)   # computed in code, NOT by LLM
```

**Why 5 dimensions not 3 or 7**: 3 conflates signals (specificity ≠ quantification); 7+ shows calibration degradation in position-bias literature. 5 is the sweet spot for rubric-first prompting.

**Why pointwise not pairwise**: pointwise + rubric + 2 anchor examples has a 9% flip rate under adversarial phrasing; pairwise has 35%. We're scoring one answer against an ideal, not ranking two candidates.

### Judge prompt mitigations (all required)

1. **Rubric-first**: rubric text comes BEFORE the answer in the prompt. Confirmed to reduce drift.
2. **Anchor examples**: 2 few-shot examples per seniority level (one strong, one weak). Pulled from the question bank's `anchor_example_strong` / `anchor_example_weak` fields.
3. **Independent scoring**: judge outputs one score per dimension, then code computes the weighted total. Never ask the LLM for the total directly.
4. **Logprobs on score tokens**: request logprobs, compute expected value from score distribution. Reduces variance.
5. **Dimension order randomization**: shuffle the dimension order across calls to counter position bias.
6. **Ensemble for critical calls**: run on both `gpt-5-mini` (primary) and `claude-haiku-4-5` (secondary) when session is user's first-ever. Flag disagreements > 1 point for human review (post-hoc, not blocking).
7. **Fixed temperature 0**, fixed rubric version (hash in session metadata for replay).

### Feedback format

**Per-question card (shown between questions, ~3s read time):**
```
┌─────────────────────────────────────┐
│  Q2 score: 3.4 / 5                  │
│  [S✓  T✓  A✓  R✗]                   │
│                                     │
│  Top fix: Your Result had no        │
│  numbers. Try:                      │
│  "cut latency from 2s to 400ms."    │
│                                     │
│  Rewrite of weakest sentence:       │
│  "We reduced customer churn"        │
│   → "I reduced churn from 12% to    │
│      7% in Q3 by launching the      │
│      cancellation save flow."       │
└─────────────────────────────────────┘
```

The rewrite is load-bearing. Generic "be more specific" is what users hate about Yoodli. Concrete worked-example feedback is what Big Interview gets right and what the "corrective feedback with worked example" literature supports.

**End-of-session report (async, ~10-15s to generate):**
- Trend chart per rubric dimension across questions
- STAR gate frequency (how often was each component missed?)
- Top 3 drill-down patterns
- Comparison against starting Elo trajectory
- Specific homework: "Practice 3 quantified-result behavioral answers before next session"
- Saved to GCS, linked from the session doc

---

## 8. Difficulty Adaptation

### Algorithm: Elo rating with LLM-judged rubric score
- **K-factor: 24** (not 32; capped per-session movement matters more than fast convergence)
- **Starting Elo**: seeded from declared seniority
  - junior: 1200
  - mid: 1400
  - senior: 1600
  - staff+: 1800
- **Optional JD adjustment**: -50 Elo if candidate is targeting a "reach" role (LLM infers from JD vs resume).

### Question selection
- Sample from questions within ±100 Elo of current user Elo
- Recency penalty: subtract `50 × exp(-days_since_seen / 7)` from a question's selection weight if user has seen it before
- Focus-area filter: respect user's chosen focus unless frustration ladder triggers
- Guarantee at least 1 "win-rate high" question in first 3 questions (confidence building)

### Elo update (after each scored turn)
```
expected = 1 / (1 + 10^((question_elo - user_elo) / 400))
actual   = weighted_score / 5                # normalize to [0, 1]
new_user_elo     = user_elo + K * (actual - expected)
new_question_elo = question_elo - K * (actual - expected)
```

User Elo persists in `mock_interview_user_state`. Question Elo updates per scored interaction; starts to converge around 30 responses.

### 6-axis difficulty tagging (v1 = manual)
Each question in the bank is tagged on:
1. **Conceptual depth** (1-5): factual recall → synthesis → novel framework design
2. **Domain specificity** (1-5): general/behavioral → broad technical → niche
3. **Ambiguity** (1-5): one correct answer → deliberately underspecified
4. **Follow-up pressure** (0-3): standalone → expected drill-down → adversarial
5. **Structure load** (1-5): one-shot → STAR → multi-step framework
6. **Time-boxed complexity** (1-5): 60s → 3-5 min thinking → 10+ min deep-dive

LLM-assisted bulk tagger seeds the bank; engineers spot-check. Tags feed into the end-of-session report ("you struggled on high-ambiguity questions — here's why") but do NOT directly pick questions in v1. Elo alone drives selection.

### Frustration ladder (response to disengagement)
Precursor signals (from Duolingo / Khan Academy retention research):
- Two consecutive turns with weighted_score < 2.0 on the same dimension
- Session elapsed > 20 min without any turn scoring > 0.7
- Abandoned answer (started, stopped, < 30% of expected length)
- Rapid-fire low-effort answers (time-to-submit collapsing)

Response ladder (escalating):
1. Drop selection band by 150 Elo for next question
2. Switch question category (behavioral ↔ technical)
3. Offer a "review your last answer" moment instead of a new question
4. Suggest break at 25-min hard cap
5. End session with partial report; never end silently

---

## 9. Question Bank

### Seeding v1
- 200 questions total for launch
- 80 behavioral (20 each at junior/mid/senior/staff Elo bands)
- 60 system design (mid to staff)
- 40 technical fundamentals (junior to senior)
- 20 mixed / curveball / recovery questions
- Manually authored by the interview-prep domain expert, tagged on all 6 axes
- Each question has `anchor_example_strong` and `anchor_example_weak` written by hand (required for judge few-shot)
- Each question has `ideal_answer_outline` for the judge context
- `company_context` optional, for tiering ("Amazon LP", "Google-style", etc.)

### Long-term
- Admin UI for adding/editing questions (Phase 2)
- LLM-assisted question generation seeded by JD + company context (Phase 3)
- User-contributed questions (rejected for v1 — quality risk)

---

## 10. Telemetry

All go to `mock_interview_turns` docs and Cloud Logging structured events.

**Per-turn metrics:**
- `endpoint_type`: did Silero VAD fire, turn-detector confirm, or barge-in trigger?
- `answer_duration_ms`
- `stt_final_latency_ms`
- `judge_latency_ms`
- `tts_ttfb_ms`
- `total_turn_latency_ms`: silence → first TTS byte (THE number)
- `judge_ensemble_disagreement`: only populated when ensemble ran

**Per-session metrics:**
- `turns_completed`
- `elo_delta_session`
- `frustration_ladder_steps_triggered`
- `end_reason`
- `rubric_version_hash`

**Dashboards (Grafana / Cloud Monitoring):**
- p50/p95/p99 total_turn_latency_ms — **primary SLO**
- Judge ensemble disagreement rate (alert if > 15%)
- Frustration-ladder firing rate (alert if > 25% of sessions reach step 2+)
- Cost per session (alert if > $0.50)
- STAR gate pass rate per seniority (for question bank calibration)

---

## 11. Ship Plan

### Phase 1 — Text-mode MVP (1 sprint / ~1 week)
**Goal**: prove the data model, rubric, and Elo loop work end-to-end with zero voice risk.

Scope:
- Schemas package: `MockInterviewSession`, `MockInterviewTurn`, `MockInterviewQuestion`, `MockInterviewUserState` with strict request models + permissive data models
- API endpoints: `POST /sessions`, `GET /sessions/:id`, `POST /sessions/:id/end`, `POST /sessions/:id/answer` (text-based for v1)
- Question bank: 40 hand-authored questions covering mid + senior behavioral
- Judge implementation: 5-dim pointwise rubric, logprobs, rubric-versioning, single model (gpt-5-mini)
- Elo update logic
- Frustration ladder stub (detection only, no response)
- Firestore collections + indexes
- Frontend: a `/interview-prep/mock/text` page — text-only chat loop for internal testing

**Exit criteria:**
- 10 dogfood sessions completed end-to-end by the team
- p50 judge latency < 1.2s
- Rubric score stability test: same answer scored within 0.3 of itself across 10 runs
- At least 3 of 10 dogfooders report the feedback was "useful and specific"

**No voice, no LiveKit, no frustration response, no end-of-session report** in Phase 1. Ship boring.

### Phase 2 — Voice-first (2 sprints / ~2 weeks)
- LiveKit integration, new `kitesforu-interview-worker` service
- Silero VAD + turn-detector endpointing
- STT streaming with partial commits
- TTS (ElevenLabs Flash v2.5)
- Barge-in handling with 500ms backchannel grace
- Visual backchannels (mic pulse, nodding avatar)
- Frustration ladder active response
- End-of-session report generation (async)
- Full question bank (200 questions)
- Judge ensemble for first-session users

**Exit criteria:**
- p50 total_turn_latency_ms < 900ms, p95 < 1500ms
- Zero mid-answer interruptions in 20 test sessions
- Internal team rates the voice experience ≥ 4/5 on "feels like a real interview"
- Cost per 15-min session ≤ $0.40

### Phase 3 — Quality & scale
- LLM-assisted question bank expansion (seeded by user JD)
- Company-specific question sets (Amazon LPs, Google, Meta, etc.)
- Adaptive rubric (different weights for behavioral vs system design)
- Learner history memory (judge sees user's weak axes from past sessions)
- B2B recruiter dashboards
- Non-English support

---

## 12. Risks & Open Problems

| Risk | Mitigation |
|------|-----------|
| LLM judge drift across model versions | Pin model + rubric hash in session metadata; replay tests on major upgrades |
| Elo feedback loop (always-win users get stuck at ceiling) | K=24 cap, per-session movement limit, forced category rotation every 4 questions |
| Voice endpoint false cuts on thinkers | 800-900ms VAD threshold + semantic turn-detector; documented open problem for first-time speakers, accept 5-10% error |
| Cost blowout from ensemble judging | Ensemble only on session 1 per user, not every session |
| Question bank exhaustion (regular users see same Qs) | Recency penalty in selection; question bank expansion on schedule |
| Judge ensemble disagreement >15% | Flagged for human calibration; rubric clarification as remediation |
| WebRTC browser compatibility | Safari iOS has known AEC issues; test matrix must include Safari 17+ |
| User anxiety (voice adds stress vs text) | Opt-in mic check + reassurance copy; offer text fallback at any turn |

**Accepted open problems (no good solution yet):**
- Distinguishing "still thinking" from "done answering" for nervous first-time users — accept ~5-10% error, add graceful recovery (if user resumes within 500ms of AI response start, stop and wait)
- LLM-as-judge reliability at the tails (very novice or very expert answers tend to saturate scores) — mitigate with ensemble flagging, not in-pipeline

---

## 13. First Code Slice (PR 1 — Phase 1 MVP starter)

**Branch**: `feat/mock-interview-schemas` in `kitesforu-schemas`

**What ships:**
1. New schema module `kitesforu_schemas/mock_interview.py` with:
   - `MockInterviewSession` (data model, permissive)
   - `MockInterviewTurn` (data model, permissive)
   - `MockInterviewQuestion` (data model, permissive)
   - `MockInterviewUserState` (data model, permissive)
   - `CreateMockSessionRequest` (request model, strict)
   - `SubmitAnswerRequest` (request model, strict)
   - `RubricScores` + `StarGate` nested models
   - All with `ConfigDict(extra='ignore')` on data models per the engineering-wisdom rule
2. Unit tests for schema validation (valid cases, Optional handling, extra-field ignore behavior)
3. Schema version bump in `pyproject.toml`
4. Export the new types from `__init__.py`

**What does NOT ship in PR 1:**
- No API endpoints (that's PR 2)
- No worker service (PR 3)
- No frontend (PR 4)
- No question bank content (PR 5, separate — content, not code)

**Why schemas first:**
- Per the engineering-wisdom file: data models are the load-bearing decision. Get them right before the API, worker, or frontend depend on them.
- Schemas package publishes to Artifact Registry; version bump can happen in parallel to downstream work.
- PR is self-contained, fast to review, fully reversible.
- Gives the rest of the team concrete types to build against immediately.

**Review checklist for PR 1:**
- [ ] All data models have `ConfigDict(extra='ignore')`
- [ ] No `max_length` on any `List` field
- [ ] `SupportedLanguage` used via `cast()`, not direct assignment
- [ ] `datetime` fields are `Optional` where not guaranteed at write time
- [ ] Request vs data model separation enforced
- [ ] Version bump in `pyproject.toml`
- [ ] Tests pass
- [ ] Type exports in `__init__.py`

---

## 14. Out-of-Scope — explicit non-goals

- Video mock interviews (Phase 3+)
- Multi-interviewer panel (Phase 2)
- Whiteboard coding simulation (needs a collaborative editor)
- Real-time sentiment / stress detection (privacy + accuracy concerns)
- Recruiter-side analytics dashboards (B2B, separate product)
- Audio backchannels ("mm-hmm" during user speech) — deferred until prosody-aware TTS is in stack
- Non-English languages in v1 — English only for launch

---

## Appendix: Research sources

**Voice turn-taking & latency** (Agent 1):
- OpenAI Realtime API, Sesame CSM, Hume EVI 2, Pi / Inflection, LiveKit Agents, Pipecat, Retell, Vapi — architecture comparisons
- Stivers et al. cross-linguistic turn-taking baseline (~200ms human gap)
- LiveKit turn-detector open-source model, Pipecat Smart Turn v2

**LLM-as-judge** (Agent 2):
- Big Interview Answer Builder, MIT CAPD STAR worksheet, Predictive Index rubric
- "Judging the Judges: Position Bias in LLM-as-a-Judge" (ACL 2025)
- "Pairwise or Pointwise? Feedback Protocols for Bias" (arxiv 2504.14716)
- "Evaluating Scoring Bias in LLM-as-a-Judge" (arxiv 2506.22316)
- Ryan 2024 (Medical Education): immediate vs delayed feedback
- Metcalfe/Kornell/Finn on delayed feedback retention

**Dynamic difficulty** (Agent 3):
- Duolingo Birdbrain (IRT + half-life regression)
- Khan Academy Elo-based difficulty
- Chess.com tactics trainer, LeetCode rating systems
- Anki SM-2 spaced repetition
- Retention research on flow state precursors

---

**End of architecture proposal.** Next step: ship PR 1 (schemas) against `kitesforu-schemas`.
