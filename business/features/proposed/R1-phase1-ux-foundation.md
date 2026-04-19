# R1 Phase 1 — UX Foundation (Navigation + Library + Classification)

**Status**: PROPOSED
**Priority**: P0
**Effort**: 2 weeks
**Affected repos**: kitesforu-frontend, kitesforu-api
**Depends on**: Nothing — ships first
**Blocks**: All subsequent phases

---

## Why This Is Phase 1

This fixes the two most reported user problems: "I can't find what I created" and "the menus are confusing." Every subsequent phase builds on the unified library and simplified navigation. Nothing else matters if users can't find their content.

---

## Deliverable 1: Content Classification Fix (Backend)

**Problem**: Content created via smart-create may not get `style=Interview`, causing it to not appear on the interview hub. The `isInterviewPrepCourse()` function uses fragile keyword matching.

**Fix**:

### API Changes (kitesforu-api)

1. Add `content_purpose` field to all course documents at creation time:
   ```python
   # In smart-create executor, _execute_interview_prep():
   update_data["content_purpose"] = "interview_prep"
   
   # In _execute_course():
   update_data["content_purpose"] = "course"
   
   # In _execute_podcast():
   # Set on podcast_jobs document
   update_data["content_purpose"] = "podcast"
   ```

2. Return `content_purpose` in the `/v1/courses` list response. Add to the Course Pydantic model.

3. One-time backfill script: set `content_purpose` on existing courses based on `style` field and `course_type` field.

### Frontend Changes (kitesforu-frontend)

1. Add `content_purpose?: string` to the `Course` TypeScript interface in `shared/schemas.ts`
2. Replace `isInterviewPrepCourse()` with `course.content_purpose === 'interview_prep' || course.style === 'Interview'`

### Acceptance Criteria
- [x] Every new course/podcast/class/writeup gets `content_purpose` set at creation time — `kitesforu-api/src/api/services/smart_create/executor.py` sets it for all 5 content types (course/class/writeup/podcast/interview_prep); commits f63bdfd (PR #237), 83aebef (PR #239)
- [x] API returns `content_purpose` in list responses — field on Course/Class/Writeup/PodcastJob models in `kitesforu-schemas`
- [x] Existing courses backfilled — `kitesforu-api/scripts/backfill_content_purpose.py` exists; memory notes backfill EXECUTED (2824 docs in kitesforu-dev on 2026-04-17)
- [x] Hub uses `content_purpose` for filtering, not keyword matching
- [x] Content created via smart-create with interview intent appears on interview hub

---

## Deliverable 2: Unified Library Page

**Route**: `/library` (new page)

**Replaces**: `/courses`, `/classes`, `/writeups`, `/activity`, `/interview-prep/hub`

### Data Fetching

```typescript
// hooks/useUnifiedLibrary.ts
// Fetch all content types in parallel, normalize into UnifiedItem[]
const [courses, classes, writeups] = await Promise.all([
  fetch('/v1/courses'),
  fetch('/v1/classes'),
  fetch('/v1/writeups'),
])
// Normalize into common shape, merge, sort
```

### UI Structure

```
[Page Header: "My Library" + create button]
[Sticky Filter Bar]
  [Filter pills: All | Interview Prep | Courses | Classes | Writing]
  [Search input with debounce]
  [Sort dropdown: Newest | A-Z | Status]
[Content Grid: AudioContentCard × N]
[Pagination: infinite scroll with sentinel]
```

### Filter Pills

| Pill | Filter Logic |
|------|-------------|
| All | No filter |
| Interview Prep | `content_purpose === 'interview_prep' \|\| style === 'Interview'` |
| Courses | `type === 'course' && content_purpose !== 'interview_prep'` |
| Classes | `type === 'class'` |
| Writing | `type === 'writeup'` |

When "Interview Prep" is active, show the interview hub metrics as a header card (active programs, mastery traces, average score) — preserving the hub's value without a separate page.

### AudioContentCard (Polymorphic Component)

One card component, different metadata per type:
- 3px top accent bar (color per type)
- Type badge + status indicator
- Title (2-line clamp)
- Type-specific subtitle (episode count, lesson count, word count)
- Progress bar (if in-progress)
- Footer: relative date + action button (play/view/retry)

### Route Redirects

| Old Route | Redirect To |
|-----------|------------|
| `/courses` | `/library?type=course` |
| `/classes` | `/library?type=class` |
| `/writeups` | `/library?type=writeup` |
| `/activity` | `/library` |
| `/interview-prep/hub` | `/library?type=interview_prep` |

### Mobile

- Filter pills scroll horizontally
- Cards stack single-column
- Search input full-width above pills

### Acceptance Criteria
- [x] `/library` shows ALL content types in one view — `app/library/page.tsx` with `useUnifiedLibrary` hook
- [x] Filter pills work correctly — 5 pills: All, Interview Prep, Audio Series, Classes, Writing
- [x] Search filters in real-time (300ms debounce) — `useDebounce` imported
- [x] Sort works (Newest, A-Z, Status) — SortKey type has `recent | alpha | status`
- [x] Old routes redirect correctly with type pre-selected — shipped in frontend PR #419: `/activity` → `/library`, `/courses` → `/library?type=course`, `/classes` → `/library?type=class`, `/writeups` → `/library?type=writeup`. Nested routes continue to render independently.
- [x] Interview hub metrics appear when Interview Prep filter is active — shipped in frontend PR #421: `components/library/InterviewHubMetricsCard.tsx` renders 3 KPIs (active programs / mastery traces / average score) as a header card.
- [x] Mobile responsive (single column, horizontal pill scroll)
- [ ] Loads in < 2s on 3G (parallel API calls) — needs perf measurement
- [x] Empty states for no content and no search results

---

## Deliverable 3: Simplified Navigation

### Desktop Navbar

Replace the current Create mega-menu + My Library dropdown with direct links:

```
[Logo]  Home  Library  [Search ⌘K]  [+ Create]  [Credits]  [Avatar]
```

- **Home** → `/`
- **Library** → `/library`
- **Search** → Cmd+K modal (existing)
- **+ Create** → `/create-smart` (direct link, not dropdown)
- **Credits** → shows credit count, links to `/pricing`
- **Avatar** → Clerk user button with Settings, Pricing, Sign Out

### Mobile Bottom Tab Bar

```
[Home]  [Library]  [+ Create]  [Search]  [Profile]
```

- 5 tabs, fixed bottom, `h-20` with safe area padding
- Create button visually elevated: `w-12 h-12 rounded-full bg-brand-500` centered, raised `-mt-4`
- Icons: Lucide (Home, Library, Plus, Search, User)
- Labels: `text-[10px]` below icons

### What Gets Removed
- NavMegaMenu component (the 11-item Create dropdown)
- NavDropdown for "My Library"
- Mobile hamburger menu (replaced by bottom tabs)

### Files Changed
- `components/Navbar.tsx` — simplify to direct links
- New: `components/BottomNav.tsx` — mobile bottom tab bar
- `app/layout.tsx` — add BottomNav for mobile, add `pb-20` on mobile for bottom bar space

### Acceptance Criteria
- [x] Desktop: 3 nav items + 2 actions (no dropdowns) — Navbar has Home | Library | Search | + Create | ThemeToggle | Avatar
- [x] Mobile: 5 bottom tabs, Create button elevated — `components/BottomNav.tsx` exists
- [x] No hamburger menu on mobile — replaced by BottomNav
- [x] Active state on current nav item — NavLink component handles active pathname
- [x] Create links to `/create-smart` directly
- [x] Library links to `/library`

---

## Implementation Plan

### Week 1: Backend + Library

| Day | Task | Owner |
|-----|------|-------|
| Mon | Add `content_purpose` to API models + executor | API |
| Mon | Write backfill script for existing courses | API |
| Tue | Return `content_purpose` in list endpoints | API |
| Tue | Add `content_purpose` to frontend Course type | Frontend |
| Wed-Thu | Build `/library` page with `useUnifiedLibrary` hook | Frontend |
| Fri | Build `AudioContentCard` component | Frontend |

### Week 2: Navigation + Polish

| Day | Task | Owner |
|-----|------|-------|
| Mon | Build `BottomNav` component | Frontend |
| Tue | Simplify `Navbar.tsx` (remove mega-menu) | Frontend |
| Wed | Add route redirects for old pages | Frontend |
| Thu | Integration testing + mobile testing | Frontend |
| Fri | Deploy to beta, verify all redirects + library + nav | Both |

### Testing Checklist
- [x] All old routes redirect correctly — frontend PR #419
- [ ] Library shows content from all types
- [ ] Content created via smart-create appears under correct filter
- [ ] Interview prep content appears under Interview Prep filter
- [ ] Mobile bottom tabs work on iOS Safari and Chrome Android
- [ ] Playwright E2E: navigate to /courses → verify redirect to /library?type=course
- [ ] Playwright E2E: create content via smart-create → verify appears in library

---

## Risks

1. **Multiple API calls may be slow** — Mitigate with Promise.all() + skeleton loading. Consider a unified `/v1/library` endpoint in Phase 2 if performance is an issue.
2. **Old bookmarked URLs break** — All redirects preserve query params. 301 permanent redirects for SEO.
3. **Interview hub metrics loss** — Metrics are preserved as a header card when Interview Prep filter is active.
4. **Mobile bottom tabs + existing MiniPlayer conflict** — MiniPlayer sits above bottom tabs. Content area gets `pb-20` (tabs) + `pb-16` (player when active) = `pb-36` max.
