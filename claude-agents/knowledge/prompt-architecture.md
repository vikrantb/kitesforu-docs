# KitesForU Prompt Architecture
<!-- last_verified: 2026-02-07 -->

## Pipeline Overview

```
API Extraction Layer (gpt-4o-mini, temp 0.1)
  -> candidate_profile.yaml (resume -> structured profile)
  -> target_profile.yaml (JD -> structured position)
  -> jd_skills.yaml (legacy JD skill extraction)
  -> prep_consolidation.yaml (combine all -> enriched context)
  |
  v
Course Workers (gpt-4o, temp 0.7)
  -> curriculum_generation.yaml (standard courses)
  -> interview_prep_curriculum.yaml (interview prep with gap analysis)
  |
  v
Orchestrator (template injection, no direct LLM)
  -> episode type templates -> filled with candidate data -> passed as custom_instructions
  |
  v
Podcast Workers (YAML-based prompts with includes system)
  -> outline_generation.yaml (outline prompt)
  -> research: task_planning.yaml -> agentic_research.yaml -> research_synthesis.yaml (planner)
  -> assimilator: research_synthesis.yaml (curate/dedupe research)
  -> script: dialogue_generation.yaml | narration_generation.yaml | monologue_generation.yaml
  -> Uses custom_instructions as additional context in {custom_instructions} variable
  |
  v
Audio Stage (TTS, no LLM)
  -> Voice synthesis from script JSON with emotion/intensity/rate metadata
```

---

## Prompt Loading System

Both repos use a `PromptLoader` class in `prompts/loader.py` with the same core architecture:

- **YAML format**: All prompts are externalized YAML with `system:`, `user_template:`, `metadata:`, `includes:`, `style_overrides:`
- **Variable substitution**: `{variable_name}` syntax, replaced at runtime
- **Include directives**: `shared/speaker_rules.yaml` or `shared/output_formats.yaml#section_name` (section-level reference with `#`)
- **Style overrides**: Tone-specific YAML files injected via `{style_guidance}` variable (comic, professional, casual, storytelling)
- **Content type rules**: Structure-specific YAML injected via `{content_type_guidance}` variable (podcast workers only; convention-based discovery at `content_types/{type}/{stage}_rules.yaml`)
- **Caching**: In-memory dict cache per loader instance

### Course Workers Loader
- **File**: `kitesforu-course-workers/src/workers/prompts/loader.py`
- Simpler; no content_type support (not needed for syllabus/orchestrator)

### Podcast Workers Loader
- **File**: `kitesforu-workers/src/workers/prompts/loader.py`
- Full content_type support with convention-based discovery
- Derives stage from prompt path (`stages/planner/outline_generation` -> stage `outline`)
- Loads `content_types/{type}/outline_rules.yaml` or `content_types/{type}/dialogue_rules.yaml` automatically

---

## API Extraction Prompts (NOT in scope for worker changes)

These run at course creation time, before any workers fire. They extract structured data from raw resume/JD text and store the result as `interview_prep_context` in Firestore.

### candidate_profile.yaml
- **File**: `kitesforu-api/src/api/services/extraction/prompts/candidate_profile.yaml`
- **Model**: gpt-4o-mini, temperature 0.1, max_tokens 1000
- **System**: "You extract condensed professional profiles from documents. Return valid JSON only. NEVER include personal information."
- **User template variable**: `{text}` (raw resume text)
- **Output JSON fields**: headline, years_of_experience, domain, experience_summary, education, technical_skills (up to 15), functional_skills (up to 10), highlights (top 3)
- **Privacy**: Explicitly strips name, email, phone, address, DOB

### target_profile.yaml
- **File**: `kitesforu-api/src/api/services/extraction/prompts/target_profile.yaml`
- **Model**: gpt-4o-mini, temperature 0.1, max_tokens 1000
- **System**: "You extract structured job/role profiles from job descriptions."
- **User template variable**: `{text}` (raw JD text)
- **Output JSON fields**: job_title, company_name, seniority_level, role_summary, technical_skills (up to 15), functional_skills (up to 10), key_responsibilities (up to 4), required_qualifications (up to 3), preferred_qualifications (up to 3)

### jd_skills.yaml (legacy)
- **File**: `kitesforu-api/src/api/services/extraction/prompts/jd_skills.yaml`
- **Model**: gpt-4o-mini, temperature 0.1, max_tokens 1000
- **System**: "You extract structured information from job descriptions."
- **Output JSON fields**: job_title, company_name, technical_skills, functional_skills
- **Note**: Legacy prompt, `target_profile.yaml` is the more comprehensive replacement

### prep_consolidation.yaml
- **File**: `kitesforu-api/src/api/services/extraction/prompts/prep_consolidation.yaml`
- **Model**: gpt-4o-mini, temperature 0.2, max_tokens 1500
- **System**: "You synthesize interview preparation context from available inputs."
- **User template variables**: `{candidate_json}`, `{target_json}`, `{help_text}`
- **Output JSON fields**: candidate_snapshot (2-3 lines), target_snapshot (2-3 lines), focus_areas (3-5), expected_outcomes (3-5)
- **Graceful degradation**: Handles missing inputs ("Not provided" fallback)

---

## Course Worker Prompts

### Syllabus Worker: curriculum_generation.yaml (Standard courses)
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/syllabus/curriculum_generation.yaml`
- **Worker**: `kitesforu-course-workers/src/workers/stages/syllabus/worker.py`
- **Version**: 1.0
- **Model**: gpt-4o, temp 0.7, max_tokens 4000, response_format json_object
- **System persona**: "You are an expert curriculum designer and instructional designer."
- **Date awareness**: `{current_date}` injected to prevent outdated content
- **User template variables**: `{topic}`, `{num_episodes}`, `{episode_duration_display}`, `{style}`, `{language}`, `{audience}`, `{context_section}`
- **Output JSON structure**:
  ```json
  {
    "summary": "...",
    "course_objectives": ["..."],
    "target_audience": "...",
    "prerequisites": ["..."],
    "episodes": [
      {
        "episode_number": 1,
        "title": "...",
        "description": "...",
        "objectives": ["..."],
        "key_topics": ["..."]
      }
    ]
  }
  ```
- **Key guidelines**: Progressive episodes, no overlap, 2-4 objectives and 3-5 key topics per episode

### Syllabus Worker: interview_prep_curriculum.yaml (Interview prep)
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/syllabus/interview_prep_curriculum.yaml`
- **Version**: 2.0
- **Model**: gpt-4o, temp 0.7, max_tokens 6000, response_format json_object
- **System persona**: "You are an expert career coach, interview preparation specialist, and curriculum designer."
- **Date awareness**: Same `{current_date}` injection
- **User template variables**: `{num_episodes}`, `{episode_duration_display}`, `{language}`, `{context_section}` (contains enriched resume/JD context)

#### Key Prompt Sections

**ANTI-GENERIC GUARDRAILS (critical quality gate)**:
- Each title MUST be candidate-specific (could NOT apply to a different candidate)
- Must reference resume items in descriptions
- At least 2 concrete practice questions per episode
- Examples of BAD vs GOOD titles:
  - BAD: "Introduction to System Design" -> GOOD: "Bridging Your Python Monolith Experience to Google's Distributed Systems Bar"

**Episode Type Classification** (required for every episode):
- `technical_deep_dive`: Technical concepts from JD needing strengthening
- `behavioral_practice`: STAR method with candidate's actual experiences
- `system_design`: Design process, decomposition, evaluation criteria
- `company_culture`: Company research, culture fit, values alignment
- `role_specific`: Role-level expectations, match/gap analysis
- `mock_interview`: Simulated interview with coaching commentary

**Episode Progression Strategy (5-step ordering)**:
1. Start with company+role context (so candidate understands target)
2. Address biggest skill gaps early (highest impact)
3. Build technical depth in middle episodes
4. Behavioral practice after candidate understands role expectations
5. End with comprehensive mock interviews

**Output JSON structure** (extends standard with):
```json
{
  "gap_analysis": {
    "strengths": ["..."],
    "gaps": ["..."],
    "unique_advantages": ["..."]
  },
  "episodes": [
    {
      "episode_number": 1,
      "title": "...",
      "description": "...",
      "objectives": ["..."],
      "key_topics": ["..."],
      "episode_type": "technical_deep_dive",
      "gaps_addressed": ["..."],
      "practice_questions": ["..."]
    }
  ]
}
```

### Planner Worker: workflow_planning.yaml (Phase 2, not active)
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/planner/workflow_planning.yaml`
- **Version**: 1, Phase 2 (stub)
- **Model config**: temp 0.1, max_tokens 2000, response_format json
- **System persona**: "You are a workflow planner for an audio course generation system."
- **User template variables**: `{worker_registry_yaml}`, `{course_type}`, `{topic}`, `{num_modules}`, `{module_duration_min}`, `{style}`, `{language}`, `{audience}`, `{context}`
- **Status**: Currently uses DeterministicRouter. This prompt is a stub for future LLM-based planning.

---

## Episode Type Templates (Orchestrator)

These are NOT direct LLM prompts. They are templates filled with candidate data by `orchestrator/worker.py` and passed as `custom_instructions` to the podcast API. The podcast pipeline then uses these instructions when generating outlines and scripts.

### Loading mechanism
- **Worker**: `kitesforu-course-workers/src/workers/stages/orchestrator/worker.py`
- **Template directory**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/`
- **Selection logic**: If episode has `episode_type` AND course context contains `=== INTERVIEW PREP CONTEXT ===`, loads the matching YAML template. Otherwise falls back to standard instructions.
- **Context parsing**: `_parse_interview_context()` extracts company, target_role, resume, JD, technical_skills, functional_skills from the enriched context string.

### Common template variables
All 6 templates share these variables:
- `{company}` - target company name
- `{target_role}` - target role title
- `{candidate_summary}` - episode position context
- `{gaps_addressed}` - specific gaps this episode targets
- `{key_topics}` - comma-separated topics
- `{practice_questions}` - bulleted practice questions
- `{objectives}` - bulleted learning objectives
- `{resume_context}` - full resume text
- `{jd_context}` - full JD text

### 1. technical_deep_dive.yaml
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/technical_deep_dive.yaml`
- **Purpose**: Technical concepts from JD that candidate needs to strengthen
- **Host personas**: Host 1 (Coach) = technical interview coach; Host 2 (Interviewer) = realistic follow-up questioner
- **Content approach** (6 steps): Bridge from experience -> explain concept -> example question walkthrough -> what interviewers evaluate -> practice variation -> common mistakes
- **Tone**: Encouraging but rigorous

### 2. behavioral_practice.yaml
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/behavioral_practice.yaml`
- **Purpose**: STAR method with candidate's actual experiences
- **Host personas**: Host 1 (Coach) = behavioral interview specialist; Host 2 (Interviewer) = asks realistic behavioral questions
- **Content approach** (7 steps): Introduce competency -> explain company-specific expectations -> STAR model answer from resume -> adapt same experience -> pause-and-think moments -> red flags -> practice variations
- **Tone**: Warm, conversational, specific actionable coaching
- **Key constraint**: "Reference specific experiences from the candidate's resume. Don't give generic STAR examples."

### 3. mock_interview.yaml
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/mock_interview.yaml`
- **Purpose**: Simulated interview with coaching commentary
- **Host personas**: Host 1 (Coach) = real-time commentary and scoring; Host 2 (Interviewer) = realistic interviewer
- **Content approach** (6 steps): Simulate realistic flow -> structured Q&A -> coaching pauses before model answers -> meta-commentary -> curveball question -> self-assessment framework + practice assignment
- **Structure**: Intro -> Questions -> Follow-ups -> Candidate's Turn -> Closing
- **Key features**: Pauses for candidate formulation, at least one surprise/curveball, actionable next steps

### 4. company_culture.yaml
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/company_culture.yaml`
- **Purpose**: Company-specific research, culture fit, values alignment
- **Host personas**: Host 1 (Coach) = career strategist; Host 2 (Insider) = discusses what working at the company is like
- **Content approach** (7 steps): Deep dive into company mission/values -> map candidate experiences to values -> differentiate from past employers -> prepare "Why this company?" answers -> cover recent news/initiatives -> address culture-fit questions -> prepare questions for interviewers
- **Tone**: Authentic and strategic

### 5. role_specific.yaml
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/role_specific.yaml`
- **Purpose**: Role-level expectations, match/gap analysis
- **Host personas**: Host 1 (Coach) = leveling expert; Host 2 (Advisor) = career trajectory advisor
- **Content approach** (7 steps): Explain target role scope -> compare current vs target level -> identify level-up areas -> prepare target-level thinking -> talk about gaps honestly -> address "why hire at this level" -> compensation/negotiation context
- **Tone**: Strategic and candid

### 6. system_design.yaml
- **File**: `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/system_design.yaml`
- **Purpose**: System design interview preparation
- **Host personas**: Host 1 (Coach) = system design expert; Host 2 (Interviewer) = poses follow-ups and pushes on trade-offs
- **Content approach** (7 steps): Present relevant design problem -> structured walkthrough (Requirements -> High-Level Design -> Deep Dive -> Trade-offs) -> drive conversation skills -> level-appropriate evaluation criteria -> bridge from existing experience -> common pitfalls -> practice prompt with pause
- **Tone**: Structured and methodical, but conversational

---

## Podcast Worker Prompts

All podcast prompts live in `kitesforu-workers/src/workers/prompts/`. They use the full loader with includes, style overrides, and content type discovery.

### Shared Components (included via `includes:` directive)

#### speaker_rules.yaml
- **File**: `kitesforu-workers/src/workers/prompts/shared/speaker_rules.yaml`
- **Purpose**: Mandatory speaker naming for TTS voice assignment
- **Rules**: ONLY "Host1" and "Host2" allowed. Never localized names, character names, or variations.
- **Why**: Exact names map to distinct TTS voices. Wrong names = both hosts get same voice.
- **Personalities**: Host1 = main driver, introduces topics; Host2 = curious one, asks follow-ups

#### quality_guidelines.yaml
- **File**: `kitesforu-workers/src/workers/prompts/shared/quality_guidelines.yaml`
- **Two sections** (referenced via `#content_quality` and `#research_quality`):
  - **content_quality**: Specific facts not vague statements, each turn builds on previous, content-adaptive engagement, anti-repetition
  - **research_quality**: Prioritize official sources, go beyond surface-level, cross-reference, anti-repetition

#### density_guidelines.yaml
- **File**: `kitesforu-workers/src/workers/prompts/shared/density_guidelines.yaml`
- **Four sections**:
  - **density_rules**: THE 5-SECOND RULE - every 5 seconds must deliver a fact, insight, example, question, or story beat
  - **banned_patterns**: Table of banned phrases with required alternatives (e.g., "That's fascinating" -> follow-up question; "A lot of people" -> "Over 2 million...")
  - **host2_rules**: Every Host2 response must Add Info, Deepen, Connect, or offer Perspective. Never pure reactions.
  - **uniqueness_rules**: Prioritize expert insights over Wikipedia summaries. "Would someone learn this from a quick Google search?" If YES -> dig deeper.

#### emotion_guidance.yaml
- **File**: `kitesforu-workers/src/workers/prompts/shared/emotion_guidance.yaml`
- **Purpose**: TTS emotion metadata instructions for expressive audio
- **Emotion options**: warm, excited, curious, thoughtful, amazed, concerned, playful
- **Intensity range**: 0.3 (subtle) to 1.0 (strong)
- **Rate range**: 0.8 (slower) to 1.3 (faster)
- **Narrative arc mapping**: Opening=warm, facts=excited/amazed, questions=curious, serious=thoughtful, jokes=playful, closing=warm

#### output_formats.yaml
- **File**: `kitesforu-workers/src/workers/prompts/shared/output_formats.yaml`
- **Sections** (referenced via `#section_name`):
  - **script_dialogue**: JSON with `dialogue[]` array (speaker, text, emotion, intensity, rate) + metadata with timestamps
  - **outline_structure**: JSON with title, content_type, introduction, segments[], conclusion + content type classification table
  - **narration_format**: Single "Narrator" speaker JSON for dramatic content
  - **monologue_format**: Single "Narrator" speaker JSON for meditation (low intensity, slow rate)
  - **research_tasks**: JSON with tasks[] array (task_id, task_type, query, search_options, priority, rationale, enabled)

### Stage Prompts

#### Outline Generation (Planner Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/planner/outline_generation.yaml`
- **Version**: 2.0
- **System persona**: "You are an expert podcast producer who creates compelling episode outlines."
- **Includes**: `output_formats.yaml#outline_structure`, `quality_guidelines.yaml#content_quality`
- **Content type rules**: Auto-discovered from `content_types/{type}/outline_rules.yaml`
- **User template variables**: `{topic}`, `{custom_instructions}`, `{tone}`, `{target_duration}`, `{content_type_guidance}`, `{content_quality}`, `{outline_structure}`
- **Content type classification**: Prompt instructs LLM to classify topic as course|story|news|educational|general
- **Structure formula**: Hook/Introduction (10-15%) -> Main Segments (70-80%, 2-4 segments) -> Conclusion (10-15%)
- **Output**: JSON with title, content_type, segments[], conclusion

#### Research Task Planning
- **File**: `kitesforu-workers/src/workers/prompts/stages/research/task_planning.yaml`
- **Version**: 2.0
- **System persona**: "You are a research strategist who plans efficient, high-quality research for podcast content."
- **Includes**: `quality_guidelines.yaml#research_quality`, `output_formats.yaml#research_tasks`
- **User template variables**: `{topic}`, `{duration_min}`, `{tone}`, `{credits_budget}`, `{current_date}`, `{research_quality}`, `{research_tasks}`
- **Duration-based scaling**: <1min=2 tasks, 1-2min=2-3 tasks, 2-5min=3-4 tasks, 5-10min=4-5 tasks, 10+min=5 tasks
- **Query design rules**: Be specific not broad, target different info types, include source qualifiers, make non-overlapping
- **Freshness bias**: CRITICAL rules for when to use topic="news" vs topic="general". Default to "news" when uncertain.
- **Output**: JSON with tasks[] array containing search_options (topic, time_range, search_depth, max_results)

#### Research Planning (Agentic, Planner Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/planner/research_planning.yaml`
- **Version**: 1.0
- **System persona**: "You are an intelligent research agent gathering information for a podcast."
- **Purpose**: Guide LLM in planning which tools to call during agentic research
- **User template variables**: `{topic}`, `{outline}`, `{available_tools}`, `{iteration}`, `{budget_remaining}`, `{previous_results}`
- **Tool selection strategy**: news topic for current events, general topic for background/evergreen

#### Research Synthesis (Planner Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/planner/research_synthesis.yaml`
- **Version**: 1.0
- **System persona**: "You are an intelligent research agent reviewing gathered information for a podcast."
- **Purpose**: Synthesize results and decide if more research needed
- **Decision framework**: COMPLETE (good coverage, low budget) vs NEED_MORE_RESEARCH (gaps exist, budget allows)
- **User template variables**: `{topic}`, `{results}`, `{iteration}`, `{budget_remaining}`

#### Agentic Research Executor
- **File**: `kitesforu-workers/src/workers/prompts/stages/tools/agentic_research.yaml`
- **Version**: 1.0
- **System persona**: "You are an intelligent research assistant for podcast content creation."
- **Purpose**: Guide LLM in orchestrating research tools (web, news, academic search)
- **Includes**: `quality_guidelines.yaml#research_quality`
- **User template variables**: `{research_needs}`, `{outline_context}`, `{research_quality}`

#### Research Synthesis (Assimilator Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/assimilator/research_synthesis.yaml`
- **Version**: 1.0
- **System persona**: "You are a research curator and synthesis expert for podcast content creation."
- **Purpose**: Transform raw research into curated, deduplicated, token-limited content for script generation
- **5 responsibilities**: Deduplicate, Assess freshness, Organize into 3-5 themes, Filter for relevance, Optimize for token limit
- **User template variables**: `{duration_min}`, `{topic}`, `{target_token_limit}`, `{current_year}`, `{research_results}`
- **Output JSON**: synthesis (summary, themes[], best_sources[], surprising_facts[]), freshness_warnings[], deduplication_count, stale_content_count, sources_used_count

#### Dialogue Generation (Script Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/script/dialogue_generation.yaml`
- **Version**: 2.0
- **System persona**: "You are an expert podcast scriptwriter who creates natural, engaging dialogue."
- **Includes**: speaker_rules.yaml, output_formats.yaml#script_dialogue, quality_guidelines.yaml#content_quality, density_guidelines.yaml#density_rules, density_guidelines.yaml#banned_patterns, density_guidelines.yaml#host2_rules, density_guidelines.yaml#uniqueness_rules
- **Style overrides**: comic, professional, casual, storytelling (loaded from `stages/script/styles/`)
- **Content type rules**: Auto-discovered from `content_types/{type}/dialogue_rules.yaml`
- **User template variables**: `{language_instructions}`, `{topic}`, `{custom_instructions}`, `{outline}`, `{research_summary}`, `{tone}`, `{language_name}`, `{speaker_rules}`, `{style_guidance}`, `{emotional_arc}`, `{expression_guidance}`, `{expression_examples}`, `{density_rules}`, `{banned_patterns}`, `{host2_rules}`, `{uniqueness_rules}`, `{content_type_guidance}`, `{content_quality}`, `{script_dialogue}`
- **Content-adaptive approach**: Explicit instruction NOT to force fixed narrative template. News=facts first, Comedy=setup->punchline, Educational=step-by-step
- **Voice direction**: EVERY line requires emotion/intensity/rate metadata for expressive TTS
- **Final checklist**: Language, speaker names, new information per exchange, research facts, no filler, emotion metadata

#### Narration Generation (Script Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/script/narration_generation.yaml`
- **Version**: 1.0
- **System persona**: "You are an expert podcast scriptwriter specializing in SINGLE-NARRATOR dramatic content."
- **Includes**: output_formats.yaml#narration_format, quality_guidelines.yaml#content_quality, emotion_guidance.yaml
- **Speaker**: ONLY "Narrator" (never Host1/Host2). Character dialogue embedded in narration text with quotes.
- **Emotion options**: mysterious, intense, warm, suspenseful, contemplative, excited, somber, urgent
- **Rate range**: 0.7-1.2 (slower for drama, faster for urgency)
- **Special notations**: `[pause]`, `[whisper]`, `[building]`
- **Style**: Cinematic, like a master storyteller

#### Monologue Generation (Script Stage)
- **File**: `kitesforu-workers/src/workers/prompts/stages/script/monologue_generation.yaml`
- **Version**: 1.0
- **System persona**: "You are an expert scriptwriter specializing in CALMING, MEDITATIVE audio content."
- **Includes**: output_formats.yaml#monologue_format, quality_guidelines.yaml#content_quality
- **Speaker**: ONLY "Narrator"
- **Emotion options**: soothing, gentle, peaceful, warm, calm, reassuring
- **Intensity**: 0.2-0.5 (NEVER above 0.6)
- **Rate**: 0.75-0.9 (NEVER above 1.0)
- **Special notations**: `[breathe]`, `[long pause]` (5+ seconds), `[pause]` (2-3 seconds)
- **Sub-styles**: Sleep/relaxation (progressive energy decrease, body scan, visualization), Guided meditation (clear non-commanding instructions, acceptance language), Mindfulness (anchor to breath, non-judgmental observation)

### Routing Prompts

#### Complexity Analysis
- **File**: `kitesforu-workers/src/workers/prompts/routing/complexity/complexity_analysis.yaml`
- **Version**: 1.0
- **System persona**: "You are a content complexity analyzer."
- **Purpose**: Classify content as SIMPLE/MODERATE/COMPLEX for model routing
- **Model config**: max_tokens 10, temperature 0.0
- **Output**: Single word: SIMPLE, MODERATE, or COMPLEX
- **User template variable**: `{content}`

#### LLM Fallback Research
- **File**: `kitesforu-workers/src/workers/prompts/routing/research/llm_fallback_research.yaml`
- **Version**: 1.0
- **System persona**: "You are a research assistant helping gather information for podcast creation."
- **Purpose**: Generate research when primary Tavily API is unavailable
- **Confidence score**: 0.6 (lower than web search)
- **User template variable**: `{query}`
- **Output JSON**: summary, facts[], limitations

---

## Content Type System (Podcast Workers)

Content types control STRUCTURE (what sections to include, how to organize). They are independent of styles which control TONE (how it sounds). The system uses convention-based auto-discovery.

### Profile files (detection + metadata)

Each content type has a `profile.yaml` defining:
- **Detection**: Regex patterns and keywords for auto-classification
- **Priority**: Higher = checked first (course=100, interview=95, story=90, news=80)
- **Context extraction**: Patterns to extract episode/chapter numbers from topic strings

| Content Type | File | Priority | Key Detection Patterns |
|-------------|------|----------|----------------------|
| course | `content_types/course/profile.yaml` | 100 | "Episode N of M", "Lesson N of M", "module" |
| interview | `content_types/interview/profile.yaml` | 95 | "interview prep", "mock interview", "STAR method" |
| story | `content_types/story/profile.yaml` | 90 | "Chapter N", "Part N:", fiction keywords |
| news | `content_types/news/profile.yaml` | 80 | "breaking", "latest", date patterns |
| educational | `content_types/educational/profile.yaml` | - | How-to, tutorial, single-topic teaching |
| general | `content_types/general/profile.yaml` | - | Default fallback |

### Rules files (injected into prompts)

#### Course Outline Rules
- **File**: `kitesforu-workers/src/workers/prompts/content_types/course/outline_rules.yaml`
- **Sections**: roadmap_rules, episode_identity_rules, progression_rules, builds_on_structure
- **Key features**: Episode 1 must have full course roadmap (15-20% of intro), middle episodes need mini-roadmap with callback, final episode needs journey recap. Episode-specific hooks required (test: "Could this opening apply to ANY other episode?"). Progressive complexity by position (0-25% = foundation, 25-75% = development, 75-100% = mastery).

#### Course Dialogue Rules
- **File**: `kitesforu-workers/src/workers/prompts/content_types/course/dialogue_rules.yaml`
- **Sections**: opening_rules, episode_content_rules, progression_dialogue_rules, closing_rules, continuity_markers
- **Key features**: Banned generic openings list, position-aware opening/closing formulas, vocabulary progression by episode position, Host2 role evolution (early=beginner questions, middle=practical, late=challenges Host1). Explicit continuity markers: Callback + Bridge + Tease for every non-first episode.

#### Interview Prep Outline Rules
- **File**: `kitesforu-workers/src/workers/prompts/content_types/interview/outline_rules.yaml`
- **Sections**: practice_first_rules, gap_targeting_rules, episode_type_rules, company_specific_rules, actionable_takeaways_rules
- **Key features**: Every segment must include "pause and practice" moment. Gap targeting when resume/JD context available. Episode type specialization (different approach per type). Banned closing patterns ("Practice makes perfect", "Remember to be yourself").

#### Interview Prep Dialogue Rules
- **File**: `kitesforu-workers/src/workers/prompts/content_types/interview/dialogue_rules.yaml`
- **Sections**: persona_rules, opening_rules, content_rules, progression_rules, closing_rules
- **Key features**: Host1=Interview Coach (not generic host), Host2=Practice Interviewer (pushes back on weak answers). Resume/JD-specific opening required. Specificity test: "Could this advice apply to ANY candidate for ANY role?" If YES -> too generic. Episode progression: early=gentle, middle=rigorous, final=mock with realistic pressure.

### Story and News types
- Story has `outline_rules.yaml` and `dialogue_rules.yaml` (narrative arc, chapter structure)
- News has `outline_rules.yaml` and `dialogue_rules.yaml` (facts-first, recency emphasis)
- Not documented in detail here; patterns match the course/interview structure

---

## Style System (Podcast Workers)

Styles control TONE (how content sounds). They are loaded via `style_overrides:` in the dialogue generation prompt.

| Style | File | Tone Characteristics |
|-------|------|---------------------|
| comic | `stages/script/styles/comic.yaml` | Humor, wit, comedic timing |
| professional | `stages/script/styles/professional.yaml` | Measured, authoritative |
| casual | `stages/script/styles/casual.yaml` | Relaxed, conversational |
| storytelling | `stages/script/styles/storytelling.yaml` | Narrative arc, dramatic |

---

## Podcast Workers Syllabus Prompt (separate from course workers)

There is ALSO a `curriculum_generation.yaml` in the podcast workers:
- **File**: `kitesforu-workers/src/workers/prompts/stages/syllabus/curriculum_generation.yaml`
- Same structure as course workers version but uses `{episode_duration_min}` (not `{episode_duration_display}`)
- This is used when the podcast workers directly generate syllabi (e.g., for standard podcast courses without the course-workers pipeline)

---

## How to Modify Prompts

### To change curriculum quality (what episodes get created):
1. Edit the relevant YAML in `kitesforu-course-workers/src/workers/prompts/stages/syllabus/`
2. Variables use `{variable_name}` syntax
3. Test by creating a course and checking `/debug/course/{id}`
4. Deploy course-workers

### To change episode content approach (how the podcast pipeline treats each episode):
1. Edit the episode type YAML in `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/`
2. These become `custom_instructions` passed to podcast API
3. Test by creating a course and checking individual episode debug pages
4. Deploy course-workers

### To change outline structure (how outlines are built):
1. Edit `kitesforu-workers/src/workers/prompts/stages/planner/outline_generation.yaml` for the base prompt
2. Edit `kitesforu-workers/src/workers/prompts/content_types/{type}/outline_rules.yaml` for type-specific rules
3. Deploy podcast workers (GH Actions, auto on main push)

### To change script/dialogue quality (how the final audio script reads):
1. Edit `kitesforu-workers/src/workers/prompts/stages/script/dialogue_generation.yaml` for the base prompt
2. Edit `kitesforu-workers/src/workers/prompts/content_types/{type}/dialogue_rules.yaml` for type-specific rules
3. Edit `kitesforu-workers/src/workers/prompts/shared/density_guidelines.yaml` for quality bars
4. Deploy podcast workers (GH Actions, auto on main push)

### To change research depth/quality:
1. Edit `kitesforu-workers/src/workers/prompts/stages/research/task_planning.yaml` for search strategy
2. Edit `kitesforu-workers/src/workers/prompts/stages/assimilator/research_synthesis.yaml` for curation
3. Deploy podcast workers

### To add a new content type:
1. Create `kitesforu-workers/src/workers/prompts/content_types/{new_type}/profile.yaml` (detection rules)
2. Create `kitesforu-workers/src/workers/prompts/content_types/{new_type}/outline_rules.yaml`
3. Create `kitesforu-workers/src/workers/prompts/content_types/{new_type}/dialogue_rules.yaml`
4. The loader auto-discovers them via convention

### To add a new episode type (interview prep):
1. Create `kitesforu-course-workers/src/workers/prompts/stages/orchestrator/episode_types/{new_type}.yaml`
2. Add the type to the Episode Type Classification list in `interview_prep_curriculum.yaml`
3. The orchestrator worker auto-loads it by `episode_type` field

---

## Diagnostic: "What's Wrong With My Output?"

| Symptom | Likely Cause | File to Check |
|---------|-------------|---------------|
| Generic episode titles | Curriculum prompt lacks specificity or anti-generic guardrails not strong enough | `kitesforu-course-workers/.../syllabus/interview_prep_curriculum.yaml` |
| Wrong episode ordering | Progression strategy too weak | `kitesforu-course-workers/.../syllabus/interview_prep_curriculum.yaml` |
| Episodes don't reference resume | Context not passed in template or parsing failed | `kitesforu-course-workers/.../orchestrator/worker.py` + `episode_types/*.yaml` |
| Script sounds robotic | Missing emotion metadata or flat intensity/rate values | `kitesforu-workers/.../script/dialogue_generation.yaml` + `shared/emotion_guidance.yaml` |
| Content is factually thin | Research prompt too shallow or budget too low | `kitesforu-workers/.../research/task_planning.yaml` |
| Wrong tone/persona | Episode type template personas wrong | `kitesforu-course-workers/.../episode_types/*.yaml` |
| Outdated information | Missing `{current_date}` in prompt or freshness bias rules violated | `curriculum_generation.yaml` or `task_planning.yaml` (freshness rules) |
| Both hosts sound the same | Speaker names not "Host1"/"Host2" | `kitesforu-workers/.../shared/speaker_rules.yaml` |
| Repetitive openings across course episodes | Content type rules not loaded or course dialogue rules too weak | `kitesforu-workers/.../content_types/course/dialogue_rules.yaml` |
| Host2 just agrees with Host1 | density_guidelines host2_rules not enforced | `kitesforu-workers/.../shared/density_guidelines.yaml` (host2_rules section) |
| Filler phrases in script | banned_patterns not strong enough | `kitesforu-workers/.../shared/density_guidelines.yaml` (banned_patterns section) |
| Generic interview advice | Interview dialogue rules specificity test not passing | `kitesforu-workers/.../content_types/interview/dialogue_rules.yaml` |
| Episode content overlaps | Course outline rules episode identity not enforced | `kitesforu-workers/.../content_types/course/outline_rules.yaml` |
| Wrong script format (narration vs dialogue) | Wrong script stage prompt loaded | Check if monologue/narration/dialogue generation prompt matched correctly |
| Research returns stale data | Search using topic="general" instead of "news" for time-sensitive content | `kitesforu-workers/.../research/task_planning.yaml` (freshness bias rules) |
| Meditation content too energetic | Intensity/rate bounds exceeded | `kitesforu-workers/.../script/monologue_generation.yaml` |

---

## Complete File Inventory

### API Extraction Prompts (4 files)
```
kitesforu-api/src/api/services/extraction/prompts/
  candidate_profile.yaml
  target_profile.yaml
  jd_skills.yaml             (legacy)
  prep_consolidation.yaml
```

### Course Worker Prompts (9 files)
```
kitesforu-course-workers/src/workers/prompts/
  loader.py                                           (prompt loading engine)
  stages/syllabus/
    curriculum_generation.yaml                         (standard courses)
    interview_prep_curriculum.yaml                     (interview prep)
  stages/planner/
    workflow_planning.yaml                             (Phase 2 stub)
  stages/orchestrator/episode_types/
    technical_deep_dive.yaml
    behavioral_practice.yaml
    mock_interview.yaml
    company_culture.yaml
    role_specific.yaml
    system_design.yaml
```

### Podcast Worker Prompts (32+ files)
```
kitesforu-workers/src/workers/prompts/
  loader.py                                            (prompt loading engine with content type support)
  shared/
    speaker_rules.yaml
    quality_guidelines.yaml
    density_guidelines.yaml
    emotion_guidance.yaml
    output_formats.yaml
  stages/
    planner/
      outline_generation.yaml
      research_planning.yaml                           (agentic research planning)
      research_synthesis.yaml                          (agentic synthesis/decision)
    research/
      task_planning.yaml                               (research task generation)
    tools/
      agentic_research.yaml                            (tool orchestration)
    assimilator/
      research_synthesis.yaml                          (research curation/dedup)
    syllabus/
      curriculum_generation.yaml                       (podcast-direct syllabus)
    script/
      dialogue_generation.yaml                         (2-host conversation)
      narration_generation.yaml                        (single dramatic narrator)
      monologue_generation.yaml                        (meditation/calm)
      styles/
        comic.yaml
        professional.yaml
        casual.yaml
        storytelling.yaml
  routing/
    complexity/
      complexity_analysis.yaml                         (model routing)
    research/
      llm_fallback_research.yaml                       (Tavily fallback)
  content_types/
    course/
      profile.yaml
      outline_rules.yaml
      dialogue_rules.yaml
    interview/
      profile.yaml
      outline_rules.yaml
      dialogue_rules.yaml
    story/
      profile.yaml
      outline_rules.yaml
      dialogue_rules.yaml
    news/
      profile.yaml
    educational/
      profile.yaml
    general/
      profile.yaml
```
