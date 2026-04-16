# P0 Phase 3 — Interview-Specific Episode Generation

**Status**: PROPOSED (implement after Phase 2)
**Priority**: P0
**Affected repos**: kitesforu-workers, kitesforu-course-workers
**Depends on**: Phase 2 (curriculum builder produces the episode specs)
**Estimate**: 12-16 hours

---

## What This Phase Does

Replaces the generic episode prompt templates with **interview-specific drill templates** that produce audio structured as practice sessions, not lectures.

## Episode Structure (what the audio sounds like)

```
[Interviewer voice — professional, direct]
"Let's talk about your leadership experience. Tell me about a time
you had to coordinate a team through a high-pressure situation."

[Brief pause — 3 seconds for listener to think]

[Coach voice — warm, precise]
"Before you answer, remember: this is a STAR question. Lead with
the Situation — where were you, what was the context? Then the
Task — what specifically was your responsibility?"

[Longer pause — 15 seconds, explicit prompt]
"Pause the audio now and answer out loud. Take your time."

[Coach voice returns]
"Here's what a strong answer sounds like for someone with your
background at Community Hospital: 'During a flu surge in December
2024, I was the senior RN on a 12-bed ICU when we went from 8
patients to 12 in one shift...' Notice how specific that is. The
interviewer at Mayo Clinic wants to hear exact numbers, exact
situations, exact outcomes."

[Interviewer voice]
"Good. Now let's go deeper — what was the result of your actions?
How did patient outcomes change?"
```

## Prompt Template Structure

```
prompts/interview_prep/
  behavioral_drill.py       # STAR-structured practice
  technical_assessment.py   # Problem-solving walkthrough
  culture_fit.py           # Company values alignment
  case_study.py            # System design / case interview
  mock_final.py            # Full simulation combining all types
  _shared_context.py       # Resume, JD, gap injection helpers
```

Each template produces a script with:
- **Interviewer segments**: questions calibrated to the specific role + seniority
- **Coach segments**: guidance that references the candidate's actual resume
- **Practice pauses**: explicit "pause and answer" prompts with timing cues
- **Company-specific references**: uses company name, JD quotes, industry context

## Implementation Steps

1. Create `prompts/interview_prep/` directory with 5 templates + shared context helper (8-10h)
2. Update course orchestrator to detect interview-prep content type and route to specialized templates (2-3h)
3. Thread resume + JD + gap into each episode's generation context (2-3h)
4. Test with a real resume + JD pair, verify episodes are drill-structured (2h)

## Voice Persona Selection

For interview prep episodes, the system auto-selects:
- **Interviewer**: professional persona, matched to the industry (tech = direct, healthcare = warm-professional, finance = formal)
- **Coach**: warm mentor persona with high scaffolding level

No user input needed — the system reads the target industry from the JD and picks appropriate voices. This is already supported by the existing persona selector; it just needs interview-prep-specific selection rules.
