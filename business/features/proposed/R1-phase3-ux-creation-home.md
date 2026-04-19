# R1 Phase 3 — UX Creation Flow & Home Dashboard

**Status**: PROPOSED
**Priority**: P1
**Effort**: 2 weeks
**Affected repos**: kitesforu-frontend
**Depends on**: R1 Phase 1 (navigation), R1 Phase 2 (player)

---

## Why This Is Phase 3

Phases 1-2 fix finding and consuming content. Phase 3 fixes how users CREATE content and what they see when they land. The current Create mega-menu (11 items) overwhelms new users. The home page is a static persona grid that doesn't adapt to returning users.

---

## Deliverable 1: Create Page

**Route**: `/create` (replaces the current redirect to `/create-smart`)

### Layout

```
[Hero Input: "What will you create?"]
[Smart Suggestions — appears as user types]
[Template Grid — organized by use-case]
[Blank Slate: "Start from scratch"]
```

### Hero Input

- Prominent text input, centered, `max-w-2xl`
- Placeholder cycles through examples: "Interview prep for Google PM...", "A bedtime story about a brave dragon...", "Explain quantum computing..."
- Voice input button (reuse existing VoiceInput component)
- Submit button with arrow icon

### Smart Suggestions

As user types (300ms debounce), classify intent and show a suggestion:

| User Types | Detected Intent | Suggestion |
|------------|----------------|------------|
| "interview at amazon" | interview_prep | "Start an interview prep series for this role →" |
| "bedtime story for maya" | creative_story | "Create a personalized bedtime story →" |
| "explain kubernetes" | course | "Create a learning series on this topic →" |
| "team update about Q3" | podcast | "Create a quick audio briefing →" |

Clicking the suggestion navigates to `/create-smart` with the query + detected template pre-filled.

### Template Grid

Organized by use-case sections (not technical content type):

**Section 1: Career & Interview** (emerald accent)
- Tech Interview, Behavioral Prep, Case Interview, Quick Mock

**Section 2: Learning & Study** (sky accent)
- Exam Review, Concept Deep Dive, Language Practice, Topic Explainer

**Section 3: Teaching** (blue accent)
- K-12 Elementary, K-12 Middle/High, University, Corporate Training

**Section 4: Stories & Entertainment** (violet accent)
- Horror Series, Bedtime Story, Sci-Fi, Comedy, Romance, Drama

**Section 5: Professional** (amber accent)
- Quick Briefing, Executive Summary, Team Update, Newsletter

### Template Card

```
[Icon/Emoji]
[Template Name — text-base font-semibold]
[Tagline — text-sm text-gray-600]
[Duration + Episode count chips]
```

Click → `/create-smart?template={id}`

### Mobile

Template sections become horizontal scroll rows. Hero input is full-width.

### Acceptance Criteria
- [x] `/create` shows hero input + template grid — `app/create/page.tsx` ships hero input + 5 template sections
- [ ] Smart suggestions appear based on typing — **deferred**; needs an intent classifier not yet exposed
- [x] Each template links to `/create-smart?template={id}`
- [x] "Start from scratch" links to `/create-smart` (no template)
- [x] Mobile: sections are horizontal scrollable rows
- [x] Page loads fast (no API calls — templates are client-side data) — `TemplateCard` data is static

---

## Deliverable 2: Personalized Home Dashboard

**Route**: `/` (refactor existing signed-in home page)

### Layout (4 sections, adaptive)

```
[Greeting Bar]         ← "Good morning, Vikrant" + contextual message
[Continue Section]     ← in-progress items, horizontal scroll (if any)
[Quick Actions]        ← 3-4 cards, persona-adaptive
[Recent]               ← last 8 items, grid
[Recommended]          ← persona-based template suggestions
```

### Section 1: Greeting

Time-aware greeting + contextual subline:
- Has generating items: "You have 2 items generating right now."
- Has ready-unplayed items: "3 new episodes ready to play."
- Default: "What will you create today?"

### Section 2: Continue (conditional)

Shows items with status in [generating, partial, curriculum_ready, syllabus_generation]. Horizontal scroll, max 3 visible. Each card shows: type badge, title, status, progress bar, relative time.

**Hidden entirely if no in-progress items** (no empty state shown).

### Section 3: Quick Actions

3-4 action cards in a grid. Ordering adapts based on `primary_persona`:

| Persona | Card 1 | Card 2 | Card 3 | Card 4 |
|---------|--------|--------|--------|--------|
| interview-candidate | Mock Interview | Interview Series | Study Audio | Quick Briefing |
| student | Study Audio | Exam Review | Quick Briefing | Mock Interview |
| creative-storyteller | Tell a Story | Audio Series | Quick Briefing | Study Audio |
| k12-teacher | Create Lesson | Classroom | Quick Briefing | Audio Series |
| New user | Mock Interview | Study Audio | Audio Series | Tell a Story |

### Section 4: Recent

Last 8 items from activity feed. Uses AudioContentCard in a 2x4 grid (desktop) or vertical list (mobile).

### Section 5: Recommended

Persona-based template suggestions from the API. 3 cards showing templates the user hasn't tried yet.

### New User State

No Continue section. Quick Actions show defaults. Recent replaced with "Get Started" cards. Recommended still shows.

### Acceptance Criteria
- [x] Greeting is time-aware and contextual — `greetingForHour()` in HomeDashboard.tsx
- [x] Continue section shows in-progress items (hidden if none) — conditional "Continue" SectionHeader
- [x] Quick Actions adapt to user's primary_persona — `QuickActionsSection({ persona })`
- [x] Recent shows last 8 items with correct types and statuses
- [x] Recommended suggests persona-relevant templates — `getRecommendedTemplates(persona)` via `RecommendedSection`
- [x] New user sees appropriate getting-started experience — Hero + PersonaSection fall-through for empty state
- [x] Mobile responsive (single column)

---

## Deliverable 3: Smart-Create Classification Improvement

**Problem**: When smart-create can't confidently classify content type, it may route interview-prep content as a regular course.

**Fix**: When classification confidence is low (< 0.7), ask ONE clarifying question in the chat:

> "Got it — I'll create audio about {topic}. Quick question: is this for **studying/prep**, **work**, or **entertainment**?"

Three tappable options. Maps to `content_purpose`. Only asked when ambiguous — 80%+ of cases auto-classify from template + topic.

### API Change

In `services/smart_create/intake.py`, when `classification_confidence < 0.7` and no template is provided, emit a `clarify_content_purpose` SSE event before generating the plan.

### Frontend Change

In `useSmartCreateChat`, handle the `clarify_content_purpose` event by showing 3 tappable option buttons in the chat. User's selection is sent as a follow-up message that includes `content_purpose` metadata.

### Acceptance Criteria
- [x] Ambiguous prompts trigger a one-question clarification — PR #239 via `<<QUICK:>>` pattern in chat prompts
- [x] Three options: studying/prep, work, entertainment — rendered as QuickActionButtons in chat UI
- [ ] Selected purpose flows through to `content_purpose` on the resulting document — **naming collision**: the code's `content_purpose` is already used for content TYPE (course/class/writeup/podcast/interview_prep). The clarify answer currently drives `content_category`/`content_domain`, not `content_purpose`. Needs a docs/code semantic decision before this can be checked off.
- [x] Clear prompts (with template or strong keywords) skip the question — PR #239 skip triggers: template selected, topic implies purpose ('horror story', 'interview prep', 'study guide'), pasted JD or syllabus
- [x] The question feels natural in the chat flow, not like a form — piggybacks on existing quick-reply infrastructure (no new SSE event/FE code)

---

## Testing Plan

- [ ] Navigate to `/create` — verify template grid and hero input
- [ ] Type "interview prep" — verify smart suggestion appears
- [ ] Click template → verify `/create-smart?template=X` opens correctly
- [ ] Visit home as returning user — verify Continue, Recent, Quick Actions
- [ ] Visit home as new user — verify getting-started experience
- [ ] Create content with ambiguous prompt — verify clarification question
- [ ] Verify Quick Actions adapt when user changes persona in settings
