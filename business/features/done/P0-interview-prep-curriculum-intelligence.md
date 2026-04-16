# P0 — Interview Prep Curriculum Intelligence

**Status**: PROPOSED — needs architecture design before any code
**Priority**: P0 (this IS the product for interview prep)
**Affected repos**: kitesforu-api, kitesforu-workers, kitesforu-course-workers, kitesforu-frontend
**Specialized code**: YES — this justifies its own code path, not reuse of generic podcast pipeline

---

## The Problem (what's broken today)

The Custom Interview Series currently produces **generic explainer episodes** that read like Wikipedia audio articles about a job field. Example from a real Registered Nurse course:

```
#1  Understanding the Role of a Registered Nurse
#2  Common Interview Questions for Nurses  
#3  Clinical Skills Assessment Preparation
```

These are useless for someone who has an actual interview. They don't reference the user's resume, the specific JD they uploaded, or the skills gap between them. A user with 3 years of clinical experience at a community hospital interviewing for a charge nurse role at Mayo Clinic gets the same generic "Understanding the Role" episode that a new grad would get.

**Root cause**: the course orchestrator sends each episode to the podcast API as an independent topic string with no resume context, no JD context, no skills gap analysis, and no interview-specific prompt structure. The podcast pipeline generates good audio from whatever topic it's given — but it's given generic topics.

---

## What It Should Be

For interview prep specifically, the course should be a **targeted drill sequence** that:

1. **Knows the gap** between the user's resume and the target JD
2. **Focuses on weak spots** — not what the user already knows, but what they need to practice
3. **Generates drill episodes, not explainers** — each episode is a simulated interview segment, not a lecture
4. **Adapts to the interview type** — behavioral episodes use STAR prompting, technical episodes use problem-solving structure, system design episodes use the design interview framework
5. **References the actual company/role** — "At Mayo Clinic, charge nurses are expected to..." not "In various healthcare settings..."

---

## Architecture: Two-Phase Design

### Phase 1: Deep Intake Wizard (frontend + API)

**Before any curriculum is generated**, the user goes through a conversational intake that narrows their prep needs. This replaces the current "resume → JD → go" flow with a chat-based wizard.

**Flow:**

```
User uploads resume + JD
    ↓
AI analyzes both and produces a Gap Analysis:
  - Skills the JD requires that the resume doesn't demonstrate
  - Experience the JD expects at a level the user hasn't shown
  - Company-specific culture/values signals from the JD
  - Interview type signals (behavioral-heavy, technical-heavy, case-heavy)
    ↓
System presents the gap analysis to the user:
  "Based on your resume and this JD, here are the areas where
   you'll face the toughest questions:"
  [ ] Leadership experience — your resume shows IC work, but the 
      JD requires team leadership examples
  [ ] System integration — the JD mentions Epic/Cerner integration,
      your resume doesn't mention either
  [ ] Behavioral: conflict resolution — charge nurse JDs always
      probe this, and your resume has no conflict examples
    ↓
User confirms, adjusts, adds their own focus areas:
  "I also want to practice salary negotiation"
  "Skip the system integration part, I have Epic experience 
   but it's not on my resume"
    ↓
System generates a Prep Plan (visible, editable curriculum):
  Episode 1: Leadership transition drill — "Tell me about a time 
    you led a team through a difficult shift change"
  Episode 2: Behavioral deep-dive — conflict resolution scenarios
    specific to charge nurse dynamics
  Episode 3: Technical assessment — clinical skills evaluation
    calibrated to the specific hospital's assessment format
  Episode 4: Culture fit — Mayo Clinic values + your experience
    mapped to their behavioral framework  
  Episode 5: Mock final round — full 30-minute simulation combining
    all focus areas
    ↓
User reviews, reorders, edits episode focus → THEN generates
```

**Key difference from today**: the user sees and shapes the curriculum BEFORE generation. Today they get 5 generic episodes after the fact.

**Implementation**: this is a **specialized chat flow** in the frontend, not the generic Smart Create chat. It deserves its own route (`/interview-prep/create` already exists), its own API endpoint for gap analysis, and its own curriculum builder that's aware of resume + JD + focus areas.

### Phase 2: Interview-Specific Episode Generation (workers)

Each episode is generated with a **specialized prompt template** that includes:

```
CONTEXT:
- Candidate resume summary: {resume_summary}
- Target role: {target_role} at {company}
- Gap being addressed: {gap_description}
- Interview type for this episode: {behavioral|technical|system_design|culture}
- Candidate's stated weak areas: {user_focus_areas}

EPISODE STRUCTURE:
- This is NOT an explainer. This is a DRILL.
- Format: simulated interview segment with interviewer + candidate coaching
- The interviewer asks questions calibrated to {seniority_level}
- After each question block, the coach provides:
  - What a strong answer sounds like (with specifics from the candidate's resume)
  - What to avoid (common mistakes for this gap area)
  - A practice prompt the user should answer out loud before continuing

VOICE:
- Interviewer: professional, direct, {company}-appropriate tone
- Coach: warm but precise, references the candidate's actual experience
```

**Key difference from today**: the episode prompt has the resume, the JD gap, and the drill structure baked in. Today it gets "Clinical Skills Assessment Preparation" and produces a Wikipedia-style overview.

---

## What Stays the Same

- The audio pipeline (TTS, voice persona, expression, mixing) stays exactly as-is — it already produces great audio from whatever prompt it gets
- The course orchestrator structure stays — it already handles parallel episode generation, progress tracking, and Firestore updates
- The audio player, episode list, and course detail page stay — they render whatever the backend produces
- Persona/voice/expression selection stays automatic and invisible — the system picks the right voice for "professional interviewer" and "warm coach" without user input

## What's New / Specialized

| Component | Generic path today | Interview-specific path needed |
|---|---|---|
| **Intake** | Resume + JD → submit | Resume + JD → gap analysis → user review → focus selection → curriculum preview → submit |
| **Curriculum generation** | Generic topic list from syllabus LLM | Gap-aware drill sequence with episode-level interview type |
| **Episode prompts** | Topic string + "episode 2 of 5" | Full context: resume, JD, gap, drill structure, interview type, company culture |
| **Episode titles** | Generic ("Understanding the Role") | Specific ("Leadership Transition Drill: Team Shift Management at Mayo Clinic") |
| **API endpoints** | POST /v1/courses | Needs: POST /v1/interview-prep/gap-analysis, POST /v1/interview-prep/curriculum |

---

## Acceptance Criteria

### Phase 1 (intake wizard)
- [ ] Gap analysis endpoint exists and returns structured focus areas from resume + JD
- [ ] Chat wizard presents gaps and lets user confirm/adjust/add
- [ ] Curriculum preview shows drill-structured episodes, not generic topics
- [ ] User can reorder, edit focus, and remove episodes before generation
- [ ] Each episode has a visible interview type tag (behavioral, technical, etc.)

### Phase 2 (episode generation)
- [ ] Episode prompts include resume context, JD gap, and drill structure
- [ ] Episodes are structured as simulated interview segments, not lectures
- [ ] Episode titles reference the specific role/company
- [ ] Coach segments reference the candidate's actual experience
- [ ] At least 3 practice prompts per episode where the user is told to pause and answer out loud

### Quality bar
- [ ] A user with a specific resume and JD gets episodes that could NOT have been generated without those inputs
- [ ] Two users targeting different roles at the same company get different curricula
- [ ] Episode content is specific enough that the user hears their own experience reflected back

---

## Dependencies

- Resume extraction already works (candidateProfile in the frontend form hook)
- JD extraction already works (targetProfile)
- Skills gap could be computed from the delta between candidateProfile.skills and targetProfile.required_qualifications
- The chat-based wizard pattern exists in Smart Create (useSmartCreateChat) — can be adapted
- The course orchestrator already handles per-episode Firestore updates

## What This Does NOT Need

- No new TTS provider integration
- No new voice persona (use existing "professional interviewer" + "warm coach" archetypes)
- No new audio pipeline changes
- No new frontend component library — reuse existing Card, CollapsibleSection, ChatSection patterns
- No changes to the course detail page (it already renders episodes; better episodes = better page automatically)

---

## Effort Estimate

- **Phase 1 (intake wizard + gap analysis API)**: 20-30 hours
  - Gap analysis LLM endpoint: 6-8h (API)
  - Chat wizard frontend: 8-12h (frontend, specialized route)
  - Curriculum preview + editing: 6-8h (frontend)
- **Phase 2 (episode generation)**: 12-16 hours
  - Interview-specific prompt templates: 6-8h (workers)
  - Episode context threading (resume + JD + gap into each episode): 4-6h (workers)
  - Curriculum builder (gap → episode sequence): 4-6h (API or course-workers)
- **Total**: 32-46 hours across 3-4 repos

This is 2-3 weeks of focused engineering. It is the single highest-leverage product investment for the Interview Mastery positioning.

---

## Why This Justifies Specialized Code

You said: *"I don't mind specialized code/duplication/whatever for this because this specific area is so important."*

Agreed. The generic podcast pipeline is optimized for "turn any topic into audio." Interview prep needs a fundamentally different intake (gap analysis), a fundamentally different curriculum structure (drills, not lectures), and fundamentally different episode prompts (resume-aware, company-specific, interview-type-aware). Trying to force this through the generic pipeline produces the generic episodes you saw in the screenshot. A specialized path produces the product that actually wins.

The specialization points:
1. **API**: new `/v1/interview-prep/gap-analysis` + `/v1/interview-prep/curriculum` endpoints
2. **Frontend**: specialized chat wizard at `/interview-prep/create` (replace the current form)
3. **Workers**: interview-specific prompt templates in a new `prompts/interview_prep/` directory
4. **Course workers**: interview-aware syllabus generator that takes gaps as input, not just a topic string

Everything else (TTS, audio mixing, player, Firestore, Pub/Sub, Cloud Run) stays shared.
