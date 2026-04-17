# UX Architecture: Unified Experience Redesign

**Date**: 2026-04-16
**Status**: Proposal — awaiting review
**Source**: 7-agent deep audit of navigation, creation flows, content types, dashboards, capabilities, user journeys, and UX best practices

---

## Part 1: Full Inventory — Every Page, Menu, and Path

### 1.1 Navigation Structure (Current)

**Navbar — Create Mega-Menu (11 items):**

| Section | Item | Route | What It Creates |
|---------|------|-------|-----------------|
| Prep & Simulation | Start with AI | `/create-smart` | Any type (LLM decides) |
| | Interview Mastery Hub | `/interview-prep/hub` | Nothing (dashboard) |
| | Interactive Mock Interview | `/interview-prep/mock/text` | Mock session |
| | Voice-First Simulation | `/interview-prep/mock/voice-preview` | Mock session |
| Curriculum & Knowledge | Custom Interview Series | `/interview-prep/create` | Course (style=Interview) |
| | Study Audio | `/create-smart?template=exam-review` | Course (style=Explainer) |
| | Classroom | `/create-smart?template=k12-elementary` | Class |
| Insights & Distribution | Executive Summaries | `/create-smart?template=blog-post` | Writeup |
| | Audio Series | `/create-smart?template=concept-deep-dive` | Course |
| | Quick Briefings | `/create-smart?template=topic-explainer` | Podcast |

**Navbar — My Library Dropdown (4 items):**

| Item | Route | What It Shows |
|------|-------|---------------|
| My Activity | `/activity` | ALL content (grouped/timeline) |
| Audio Series | `/courses` | Courses + Interview Prep (mixed) |
| Classrooms | `/classes` | Classes only |
| Writing | `/writeups` | Writeups only |

**Home Page (signed-in) — 6 Persona Cards:**

| Persona | First Template Route | Target Type |
|---------|---------------------|-------------|
| Interview Prep | `/create-smart?template=tech-interview` | interview-prep |
| Quick Audio | `/create-smart?template=topic-explainer` | podcast |
| Study Audio | `/create-smart?template=exam-review` | series |
| Creative Stories | `/create-smart?template=horror-series` | series |
| K-12 Classroom | `/create-smart?template=k12-elementary` | class |
| Corporate Training | `/create-smart?template=corp-onboarding` | class |

### 1.2 All Routes (48 total)

**Content Hubs / Dashboards:**

| Route | Purpose | Shows Content From |
|-------|---------|-------------------|
| `/` | Home / Launchpad | Nothing (CTAs only) |
| `/courses` | Audio Series library | `courses` collection (ALL styles) |
| `/interview-prep/hub` | Interview dashboard | `courses` filtered by style=Interview + keywords |
| `/classes` | Classrooms library | `classes` collection |
| `/writeups` | Writing library | `writeups` collection |
| `/activity` | Activity feed | ALL collections (multiple API calls) |

**Creation Flows:**

| Route | Creates | Style Set | content_type |
|-------|---------|-----------|--------------|
| `/create-smart` | Any type | LLM decides | LLM decides |
| `/create-smart?template=tech-interview` | Course | LLM decides (should be Interview) | LLM decides |
| `/interview-prep/create` | Course | Hardcoded `Interview` | Always `interview_prep` |
| `/courses/create` | Course | User selects | `course` |
| `/classes/create` | Class | N/A | `class` |
| `/writeups/create` | Writeup | N/A | `writeup` |

**Detail / Playback:**

| Route | Purpose |
|-------|---------|
| `/courses/[courseId]` | Course detail + audio player |
| `/classes/[classId]` | Class management (teacher) |
| `/classes/learn/[classId]` | Class learning (student) |
| `/writeups/[writeupId]` | Writeup viewer |
| `/studio/[jobId]` | Podcast studio |
| `/progress/[jobId]` | Generation progress |
| `/drive` | Drive mode (all audio types) |
| `/car-mode` | Car mode (voice-first brainstorm) |

**Interview Prep Specific:**

| Route | Purpose |
|-------|---------|
| `/interview-prep` | Landing page (gateway) |
| `/interview-prep/hub` | Executive dashboard |
| `/interview-prep/create` | Multi-step builder |
| `/interview-prep/mock/text` | Text mock interview |
| `/interview-prep/mock/voice-preview` | Voice simulation |

### 1.3 Content Type Storage

| Content Type | Firestore Collection | Distinguishing Field | Frontend Detection |
|-------------|---------------------|---------------------|-------------------|
| Podcast | `podcast_jobs` | N/A | Collection itself |
| Course | `courses` | `style != 'Interview'` | `style` field |
| Interview Prep | `courses` | `style='Interview'` + optional `course_type='interview_prep'` | `isInterviewPrepCourse()` keyword matching (fragile) |
| Class | `classes` | N/A | Collection itself |
| Writeup | `writeups` | N/A | Collection itself |

**Root problem**: Interview prep is NOT its own collection. It's a course with `style='Interview'` and an optional `course_type` field. Detection is inconsistent — some code checks `style`, some checks `course_type`, some checks keywords in the topic.

---

## Part 2: The Problems

### 2.1 The Classification Gap

When a user says "help me prepare for an Amazon interview" via `/create-smart`:
1. LLM detects intent and classifies `content_type`
2. If classified as `"interview-prep"` -> executor calls `_execute_interview_prep()` -> sets `style=Interview` + `course_type=interview_prep` -> appears on hub
3. If classified as `"course"` -> executor calls `_execute_course()` -> sets whatever style the LLM picked -> does NOT appear on hub
4. **The user has no idea which path was taken.** They just said what they wanted.

### 2.2 No Single Source of Truth

- `/courses` shows everything (interview + non-interview mixed)
- `/interview-prep/hub` shows only what passes keyword filter
- `/activity` shows everything but from separate API calls
- There is no unified "My Library" with smart filtering

### 2.3 Capabilities Locked to Content Types

| Capability | Available Where | Should Be Available |
|-----------|----------------|-------------------|
| Quick Q&A | Drive/Car Mode only | Audio player everywhere |
| Voice Personas | Backend only (not in UI) | Creation flow for all types |
| Car Mode | Standalone page | Accessible from any course/episode |
| Mastery Traces | Interview prep only | Could extend to courses |
| Share | Courses + Classes (hidden) | All content types (visible) |
| SCORM Export | Classes only | Could extend to courses |

### 2.4 Dead Ends

| Feature | Where It Exists | Problem |
|---------|----------------|---------|
| Q&A on episodes | Drive mode overlay | No button on `/courses/[id]` player |
| Share content | `/courses/shared/[token]` | No visible Share button anywhere |
| Car Mode for courses | `/drive?contentType=course` | No CTA from course page |
| Voice persona picker | Backend persona system | Not in any creation UI |

---

## Part 3: Proposed Architecture

### 3.1 Design Principles (from UX research on Canva, Spotify, Notion, Figma, Descript)

1. **Hub for discovery, spoke for creation, flat for library** (Canva)
2. **Unified library with filter pills, no workspaces** (Spotify/Gmail)
3. **Uniform listing, specialized detail** (Figma/Descript — polymorphic cards)
4. **All capabilities available to all types, UI varies prominence** (Canva/Descript)
5. **Classification at creation time flows through to library** (the root fix)

### 3.2 The Object Model

```
AudioContent (base interface)
  id, title, thumbnail, created_at, duration, status, content_type
  Common actions: play, share, delete, download

  InterviewPrep extends AudioContent
    + target_role, company, confidence_score, question_coverage
    + Actions: practice_mock, view_feedback, gap_analysis

  Course extends AudioContent
    + modules[], progress_pct, learning_outcomes
    + Actions: continue_learning, view_syllabus

  Podcast extends AudioContent
    + episode_count, show_notes, transcript
    + Actions: publish, view_studio

  ClassroomContent extends AudioContent
    + grade_level, subject, student_count
    + Actions: assign, track_progress, export_scorm

  CreativeStory extends AudioContent
    + genre, characters, series_info
    + Actions: continue_series, remix
```

Every piece of content IS an AudioContent at the base level. The library shows AudioContent cards. The detail view renders the specialized subtype.

### 3.3 Navigation Options

**Option A: Sidebar + Filter Pills (Notion/Spotify hybrid)**

```
Desktop:
+----------+------------------------------------+
| SIDEBAR  |  MAIN CONTENT                      |
|          |                                    |
| [Home]   |  [All|Interview|Course|Pod|Class]  |
| [Library] |                                    |
| [Create] |  Card  Card  Card  Card            |
|          |  Card  Card  Card  Card            |
| -------- |                                    |
| RECENT   |  [Search] [Sort: Newest v]         |
| item 1   |                                    |
| item 2   |                                    |
| -------- |                                    |
| [Settings]|                                   |
+----------+------------------------------------+

Mobile:
+-------------------------------------+
| [All|Interview|Course|Pod|Class]    |  <- Sticky filter pills
|                                     |
|  Card    Card                       |
|  Card    Card                       |
|                                     |
+-------------------------------------+
| [Home] [Library] [+] [Search] [Me]  |  <- Bottom tab bar
+-------------------------------------+
```

**Option B: Enhanced Current Layout (lower effort)**
- Replace "My Library" dropdown with single "Library" link -> unified library with filter pills
- Interview Prep Hub becomes the "Interview Prep" filter view within library
- Keep Create mega-menu as-is

**Option C: Use-Case Landing Pages (highest effort)**
- `/interview-prep` — hub + mock + series + traces (partially exists)
- `/learn` — courses + progress + Q&A
- `/create-audio` — podcasts + studio + distribution
- `/classroom` — classes + students + SCORM
- `/stories` — creative series + genres + voice personas

### 3.4 Recommended Progression: B -> A

**Phase 1 (Option B — 1-2 weeks):**
- Create unified `/library` page with filter pills
- Ensure `content_type` is set reliably at creation time
- Add type badge to all content cards
- Replace "My Library" dropdown with single "Library" link

**Phase 2 (toward Option A — 2-4 weeks):**
- Add sidebar navigation (Home, Library, Create)
- Home becomes personalized dashboard (Continue, Recent, Quick Actions)
- Detail views become type-specialized
- Q&A, Car Mode, Share become universally accessible

**Phase 3 (Option C elements — 4+ weeks):**
- Use-case landing pages for biggest verticals
- Voice persona picker in creation flow
- Mastery system extension beyond interview prep

### 3.5 The Critical Backend Fix (P0)

**Every piece of content must have a reliable `content_type` field.**

```python
# In smart-create executor, ALWAYS set content_type on Firestore document:
db.collection("courses").document(course_id).set({
    "content_type": content_type,  # "interview_prep", "course", etc.
}, merge=True)
```

Frontend detection becomes reliable:
```typescript
// BEFORE (fragile keyword matching):
function isInterviewPrepCourse(course: Course): boolean {
  return course.style === 'Interview' || topic.includes('interview')
}

// AFTER (reliable field check):
function isInterviewPrepCourse(course: Course): boolean {
  return course.content_type === 'interview_prep' || course.style === 'Interview'
}
```

Required steps:
1. API returns `content_type` in course list response
2. Frontend Course type adds `content_type?: string`
3. Library/Hub filter uses `content_type` instead of keyword matching
4. Backfill existing courses based on `style` field

---

## Part 4: Capability Sharing Plan

### Current vs Proposed

| Capability | Current | Proposed |
|-----------|---------|----------|
| Quick Q&A | Drive/Car Mode only | Audio player button (all types) |
| Voice Personas | Backend only | Creation flow picker (all types) |
| Car Mode | Standalone `/drive` | "Listen in Car Mode" button on course pages |
| Mastery Traces | Interview mock only | Extend to course quizzes |
| Share | Hidden | Visible button on all detail pages |
| Text Mock | Interview only | Keep exclusive (IS interview-specific) |
| Voice Sim | Interview only | Keep exclusive (IS interview-specific) |

### Implementation: `getCapabilities(contentType)`

```typescript
type CapabilityLevel = 'prominent' | 'available' | 'hidden'

function getCapabilities(contentType: string): Record<string, CapabilityLevel> {
  const base = { play: 'prominent', share: 'prominent', carMode: 'available', quickQA: 'available' }

  switch (contentType) {
    case 'interview_prep':
      return { ...base, textMock: 'prominent', voiceSim: 'prominent', masteryTrace: 'prominent' }
    case 'course':
      return { ...base, carMode: 'prominent', quickQA: 'prominent' }
    case 'class':
      return { ...base, scormExport: 'available', assignStudents: 'prominent' }
    case 'podcast':
      return { ...base, studio: 'prominent', distribute: 'prominent' }
    default:
      return base
  }
}
```

---

## Part 5: Use Case Agents

### Agent 1: Interview Candidate
**Journey**: Create prep -> Listen -> Practice mock -> Track mastery -> Iterate
**Home**: Library filtered to "Interview Prep"
**Unique**: Text mock, voice sim, mastery traces, gap analysis
**Shared**: Audio player, Q&A, Car Mode, voice personas

### Agent 2: Lifelong Learner
**Journey**: Find topic -> Create course -> Listen -> Ask questions -> Track progress
**Home**: Library filtered to "Courses"
**Unique**: Module progress, learning outcomes
**Shared**: Audio player, Q&A, Car Mode, voice personas

### Agent 3: Creative Storyteller
**Journey**: Pick genre -> Create series -> Listen -> Share -> Continue
**Home**: Library filtered to "Creative"
**Unique**: Genre templates, series continuation
**Shared**: Audio player, voice personas, share, Car Mode

### Agent 4: Podcast Creator
**Journey**: Describe topic -> Generate -> Edit in studio -> Distribute
**Home**: Library filtered to "Podcasts"
**Unique**: Studio editor, RSS, show notes
**Shared**: Audio player, voice personas, share

### Agent 5: K-12 Teacher
**Journey**: Design lesson -> Generate -> Share with students -> Track
**Home**: Library filtered to "Classroom"
**Unique**: Class management, enrollment, quiz grading, SCORM
**Shared**: Audio player, voice personas

### Agent 6: Corporate Trainer
**Journey**: Build training -> Assign -> Track completion -> Export
**Home**: Library filtered to "Classroom" (corporate)
**Unique**: Compliance tracking, team assignment
**Shared**: Audio player, voice personas

### Agent 7: Self-Paced Student
**Journey**: Join class -> Listen -> Take quizzes -> Track mastery
**Home**: Library filtered to "Enrolled"
**Unique**: Quiz interface, progress tracking
**Shared**: Audio player, Q&A

---

## Part 6: Prioritized Action Items

### P0 — Fix the classification gap (1-2 days)
1. API: Always set `content_type` on courses at creation time
2. API: Return `content_type` in list response
3. Frontend: Add `content_type?: string` to Course type
4. Frontend: Hub uses `content_type` for filtering
5. Backfill existing courses based on `style` field

### P1 — Unified Library page (1 week)
1. Create `/library` with filter pills (All | Interview | Course | Podcast | Class | Writing)
2. Polymorphic AudioContentCard with type badge
3. Search + sort + pagination
4. Replace "My Library" dropdown with "Library" link

### P1 — Surface hidden capabilities (1 week)
1. "Ask a Question" button on audio player
2. "Listen in Car Mode" on course detail pages
3. "Share" button on all detail headers

### P2 — Personalized Home dashboard (2 weeks)
1. "Continue" section (in-progress content)
2. "Recent" section (chronological)
3. "Quick Actions" (creation CTAs)

### P3 — Sidebar + type-specialized detail views (4 weeks)
1. Desktop sidebar (Home, Library, Create, Recent)
2. Interview prep detail: practice mode + traces inline
3. Course detail: module progress + Q&A panel
4. `getCapabilities(contentType)` for adaptive UI

---

## Appendix: Research Sources

| Agent | Key Finding |
|-------|-------------|
| Navigation Auditor | 48 routes, 11 Create menu items — scattered |
| Creation Flow Auditor | Smart-create vs interview-prep produce different `style` values |
| Content Type Auditor | Interview prep is NOT its own collection — implicit detection |
| Dashboard Auditor | No unified library; content spread across 5+ pages |
| Capabilities Auditor | Q&A locked to Drive, voice personas not in UI, share hidden |
| Journey Auditor | 2 of 5 journeys have dead ends (Q&A unreachable, share hidden) |
| UX Researcher | Canva/Spotify/Figma pattern: unified library + filter pills + polymorphic cards |
