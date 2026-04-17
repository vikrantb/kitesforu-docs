# KitesForU UX Redesign — Comprehensive Plan

**Date**: 2026-04-16
**Status**: PLAN — ready for review
**Source**: 7 specialized agents (PM, 2 UX designers, engineer, visual designer, user advocate, product strategist) informed by 27-persona research
**Companion docs**: [UX Architecture](./ux-architecture-unified-experience.md) | [Persona Bible](./user-persona-bible-index.md)

---

## Executive Summary

The platform has 48 routes, 11 Create menu items, and 5 separate content list pages. Users can't find what they created. The redesign collapses this into **4 top-level pages** (Home, Create, Library, Drive), introduces a **unified library with filter pills**, and makes the player **persistent across navigation** like Spotify.

**Positioning**: "Audio content that knows you — created in seconds."

**Timeline**: 7 weeks across 3 phases. 155 engineering hours. HIGH feasibility — existing codebase has all foundational patterns.

---

## Part 1: Information Architecture

### The 4 Pages

| Page | Route | Purpose | Replaces |
|------|-------|---------|----------|
| **Home** | `/` | Personalized dashboard: Continue, Quick Actions, Recent, Recommended | Current static persona grid |
| **Create** | `/create` | "What will you create?" — input + templates | 11-item Create mega-menu |
| **Library** | `/library` | ALL content, unified with filter pills | /courses, /classes, /writeups, /activity, /interview-prep/hub |
| **Drive** | `/drive` | Hands-free listening mode | Currently hidden, promoted to top-level |

### What Gets Killed

- The Create mega-menu (11 items in 3 sections)
- The "My Library" dropdown (4 items leading to 4 separate pages)
- `/courses` as a standalone page → redirect to `/library?type=course`
- `/classes` as a standalone page → redirect to `/library?type=class`
- `/writeups` as a standalone page → redirect to `/library?type=writeup`
- `/interview-prep/hub` as a standalone page → redirect to `/library?type=interview_prep`
- `/activity` → redirect to `/library`

### What Stays

- `/courses/[courseId]` — detail pages remain
- `/classes/[classId]` — detail pages remain
- `/interview-prep/create` — dedicated builder stays
- `/interview-prep/mock/*` — mock interview flows stay
- `/create-smart` — AI chat creation flow stays (Create page routes into it)
- `/drive`, `/car-mode` — stay as-is

### The Three Modes (Implicit)

Modes (Learn, Create, Experience) are **metadata on content, not UI states**. The system detects mode from user behavior and adapts the home page. Users never see mode labels or toggle between modes.

```typescript
interface TemplatePreset {
  // ... existing fields ...
  contentMode: 'learn' | 'create' | 'experience'  // NEW — drives home personalization
}
```

### Cross-Linking Fix

**The root cause**: `isInterviewPrepCourse()` uses keyword matching. Content created via smart-create may not get `style=Interview`.

**The fix**: Add `content_purpose` field set at creation time:
```python
# On every Firestore document at creation time:
content_purpose: str  # "interview_prep" | "study" | "creative" | "professional" | "classroom"
content_mode: str     # "learn" | "create" | "experience"
```

For ambiguous cases (confidence < 0.7), smart-create asks ONE question:
> "Is this for **studying/prep**, **work**, or **entertainment**?"

Library filters then query `content_purpose`, not keyword heuristics. One-time backfill for existing documents.

---

## Part 2: Navigation

### Desktop — Top Navbar

```
[Logo]   Home   Library   Discover    [Search ⌘K]   [+ Create]   [Avatar]
```

4 nav items + 1 action button. No dropdowns. No mega-menu. Each is a direct link.

### Mobile — Bottom Tab Bar

```
[Home]   [Library]   [+ Create]   [Search]   [Profile]
```

5 tabs. The Create button is visually elevated — `w-12 h-12 rounded-full bg-indigo-600` raised above the bar. Tapping opens a bottom sheet with 4 format options, not a page transition.

### Sidebar (Contextual)

Only appears on the Library page (desktop). 256px wide, collapsible to 64px icon rail. Contains: Quick filters (Favorites, Recent, Downloads), user-created Collections, and auto-generated Tags.

### Global Search (Cmd+K)

Searches 4 categories in one overlay: Your Content, Templates, Help, Quick Actions. Keyboard-navigable with arrow keys, Enter to open, Tab to cycle categories.

---

## Part 3: Key Page Designs

### Home Dashboard

4 sections, adaptive to user behavior:

1. **Continue** — in-progress items, horizontal scroll. Max 3. Hidden if empty.
2. **Quick Actions** — 3-4 action cards, persona-adaptive ordering. Interview prepper sees Mock Interview first. Bedtime parent sees "Tonight's Story" first.
3. **Recent** — last 8 items in a 2x4 grid (desktop), vertical list (mobile).
4. **Recommended** — persona-based template suggestions from the API.

New user state: Quick Actions (default set) + Persona Grid (existing, kept as-is).

### Unified Library

- **Filter pills**: All | Interview Prep | Audio Series | Classrooms | Writing | Quick Audio
- **Search**: 300ms debounce, searches title + summary + language
- **Sort**: Recently Updated (default) | Recently Created | A-Z | Duration | Progress
- **View**: Grid (default) or List toggle
- **Cards**: Polymorphic AudioContentCard — same layout, different accent colors and metadata per type
- **Pagination**: Infinite scroll with IntersectionObserver

When "Interview Prep" filter is active, the interview hub metrics (active programs, mastery traces, average score) render as a header card above the list.

### Create Page

Replaces the mega-menu with a beautiful, focused page:

1. **Hero input**: "What will you create?" — prominent text field with smart suggestions as user types
2. **Smart detection**: Typing triggers intent classification. "Interview at Google" suggests Interview Prep flow. "Horror story" suggests Creative.
3. **Template grid**: Organized by use-case sections (Career & Interview, Learning & Study, Teaching, Stories & Entertainment, Professional & Business)
4. **Blank slate card**: "Start from scratch" at bottom, links to `/create-smart`

### Content Detail Page (Polymorphic)

One page shell, type-specific content panels:
- **Header**: Title, status badge, type badge, action buttons (Play All, Share, Car Mode, Download)
- **Tabs vary by type**:
  - Interview Prep: Episodes | Mock Practice | Mastery | Settings
  - Course: Episodes | Progress | Q&A | Settings
  - Story: Episodes | Series | Voice Cast | Settings
  - Classroom: Lessons | Students | Quizzes | Settings

### Player (Persistent)

Lives in `layout.tsx`, persists across all navigation. Two states:

**MiniPlayer** (bottom bar, 76px):
- 3px seekable progress rail at top
- Episode info (clickable to expand) + time display + transport controls
- Appears above mobile bottom tabs

**FullPlayer** (bottom sheet, 90vh):
- Drag handle + artwork + episode info + full progress bar
- Primary controls: prev, skip-15, play/pause, skip+15, next
- Secondary controls: speed, Car Mode, Q&A, volume, share, queue, sleep timer
- Voice persona info (for story/creative content)
- Episode queue with drag-to-reorder

---

## Part 4: Visual Design System

### Color

- **Brand**: Keep orange-500 (#F97316)
- **Neutrals**: Shift from slate to zinc (warmer, pairs better with orange)
- **Content accents**: CSS custom properties per type — `[data-content-type="horror"] { --accent: 220 38 38 }`. Usage: `bg-accent/10 text-accent border-accent/20`
- **Dark mode**: Yes, mandatory. `darkMode: 'class'`. Auto-enable between 8pm-7am.

### Cards

- Top accent line (3px, type-colored)
- Title (2-line clamp, `text-sm font-semibold`)
- Type badge (`bg-accent/10 text-accent`)
- Status indicator (dot + label)
- Progress bar at card bottom (for in-progress items)
- Hover: `shadow-md, -translate-y-0.5`, thumbnail scales 1.03

### Typography

- Inter for body, Inter Display for headings 24px+
- 14px body base, 1.25 ratio scale
- Text colors: zinc-900 (primary), zinc-600 (secondary), zinc-500 (tertiary)

### Micro-interactions

- Page transitions: crossfade with 8px vertical motion (200ms)
- Card hover: lift + shadow + thumbnail scale
- Player expand: spring animation (damping 30, stiffness 300)
- Filter pill selection: layout animation with checkmark appear
- Loading: skeleton with shimmer
- Generation complete: brief green checkmark pulse (1.5s)

### Accessibility

- WCAG AA contrast on all text
- Focus-visible ring on all interactives
- 44px minimum touch targets
- `prefers-reduced-motion` respected
- aria-labels on all icon buttons
- aria-live regions for player state changes
- Color never sole information carrier (always icon + text + color)

---

## Part 5: Priority & Trade-offs

### Top 10 Personas by Business Value

1. Bedtime Parent (daily 365/year, multi-year LTV)
2. Non-Native Speaker (12-36 month subscription)
3. Exam Crammer (high WTP during study, strong referral)
4. Commuter (daily habit, broad TAM)
5. Solo Podcaster (growth channel — every episode is marketing)
6. Career Pivoter (3-6 month subscription, specialized content)
7. Meeting Prepper (high per-use value, B2B potential)
8. Comedy Listener (highest virality, daily usage)
9. Horror Listener (passionate niche, voice persona showcase)
10. Grief Listener (powerful testimonials, ethical brand value)

### Day-1 Personas (optimize first)

1. **Bedtime Parent** — highest frequency, fix content safety + queue + sleep timer
2. **Exam Crammer** — fix organization by topic, progress tracking
3. **Commuter** — fix Car Mode discovery, queue management, offline
4. **Solo Podcaster** — fix creation flow for creators, distribution

### MVP: 3 Changes That Matter Most

1. **Unified Library** — one page with filter pills, replaces 5 scattered pages
2. **Simplified Navigation** — 4 top-level items, kill the mega-menu
3. **Persistent Player** — audio continues across page navigation

### Trade-off Decisions (DECIDED)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Library structure | Unified with filter pills | Cheaper, avoids siloing, serves all personas |
| Navigation | 4 items + contextual overflow | 90/10 rule |
| Home personalization | Declared-intent, not algorithmic | Transparent, no cold-start problem |
| Voice picker in creation | Collapsed by default, smart defaults | Zero cost to skippers, full power for enthusiasts |
| Dark mode | Ship for everyone, auto at night | Platform expectation |
| Persona selection at onboarding | Kill it — go straight to creation | Detect, don't ask |

---

## Part 6: Product Strategy

### Positioning

**"Audio content that knows you — created in seconds."**

### First-Run Experience

Kill persona selection. After signup, land directly on creation:
1. "What do you want to listen to?" with smart suggestions
2. System infers persona from first creation
3. Wait screen during generation IS the onboarding (tips carousel)
4. After first listen, contextual next-step prompt

### Pricing

- **Free**: 3 episodes/month, all genres, standard voices
- **Pro ($9.99/month)**: Unlimited, premium voices, 60min episodes, series, Car Mode, offline
- **Upgrade trigger**: Premium voice preview after standard playback — let quality sell itself

### Retention Hooks

| Persona | Hook | Frequency |
|---------|------|-----------|
| Commuter | "Your morning briefing is ready" | Daily (weekday) |
| Bedtime Parent | "Tonight's story is ready" | Daily (evening) |
| Exam Crammer | "12 days left — 4 modules to go" | Daily (study period) |
| Horror/Romance | "Episode 4 of your series is ready" | Per release schedule |
| Book Club | "Meeting in 3 days — prep ready" | Monthly |

---

## Part 7: Engineering Plan

### Feasibility: HIGH

The codebase has all foundational patterns: `useDebounce`, `useAudioPlayer`, `useBridgeAudio` (cross-page audio), polymorphic card patterns in `/components/activity/`, status badge system, skeleton loading. The AudioPlayer already handles iOS Safari autoplay correctly.

### Phase 1: Foundation (1 week)

| Deliverable | Effort | Risk |
|-------------|--------|------|
| `useListPageLogic` hook (extract from courses page) | 3h | Low |
| `AudioContentCard` component (polymorphic) | 8h | Low |
| `AppLayout` shell (not wired yet) | 8h | Low |

### Phase 2: Library + Navigation (2 weeks)

| Deliverable | Effort | Risk |
|-------------|--------|------|
| Unified `/library` page (4 API calls, merge, filter, sort, paginate) | 40h | Medium |
| Integrate sidebar layout | 8h | Low |
| Route redirects (/courses → /library) | 4h | Low |
| `/create` template discovery page | 8h | Low |

### Phase 3: Player + Advanced (4 weeks)

| Deliverable | Effort | Risk |
|-------------|--------|------|
| Persistent AudioPlayer (move to layout.tsx, use bridge pattern) | 40h | Medium-High |
| Car Mode + Q&A buttons on detail pages | 28h | Medium |
| Mobile bottom nav with player integration | 16h | Medium |

**Total: 155 hours, 7 weeks**

### New Files

```
app/create/page.tsx
app/library/page.tsx
hooks/useUnifiedLibrary.ts
hooks/useListPageLogic.ts
components/content/AudioContentCard.tsx
components/content/ContentDetailShell.tsx
components/content/TypeBadge.tsx
components/content/StatusIndicator.tsx
components/AppLayout.tsx
components/AppSidebar.tsx
components/BottomNav.tsx
components/home/DashboardSections.tsx
```

### Files to Modify

```
app/page.tsx — refactor signed-in view
app/layout.tsx — add persistent player + bottom padding
components/Navbar.tsx — simplify to 4 direct links
components/AudioPlayer/AudioPlayer.tsx — move state to bridge pattern
```

---

## Part 8: Success Metrics

| Metric | Current | Target | Timeframe |
|--------|---------|--------|-----------|
| "Where is my content?" support queries | Unknown | Zero | Phase 2 launch |
| Library page load time | N/A (no unified library) | < 2s on 3G | Phase 2 |
| Audio completion rate | Unknown | > 70% | Phase 3 |
| DAU/MAU ratio | Unknown | > 40% | 3 months post-launch |
| Creation-to-listen conversion | Unknown | > 80% | Phase 2 |
| Content created via wrong path showing on hub | Common | Zero | Phase 2 (content_purpose field) |

---

## Appendix: Agent Contributions

| Agent | Role | Key Decision |
|-------|------|-------------|
| PM Architecture | Information Architecture | 4 pages, modes are implicit metadata |
| UX Navigation | Menu & Nav Design | Top navbar + contextual sidebar, mobile bottom tabs |
| UX Pages | Page Layouts | 5 detailed page specs with Tailwind classes |
| Engineering | Feasibility | 155h, 7 weeks, HIGH feasibility |
| UX Visual | Design System | Zinc palette, CSS custom properties for type theming, dark mode |
| User Advocate | Priority & Trade-offs | Bedtime Parent is #1 business value, 3 MVP changes |
| Product Strategy | Positioning & GTM | "Audio content that knows you", kill persona onboarding |
