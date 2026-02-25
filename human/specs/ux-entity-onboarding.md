# UX Improvements: Entity Discoverability & Onboarding

## Context

Entity naming system documented in `ENTITIES.md` and `entity-dictionary.md`. Navigation terminology unified in PR #174. This spec covers frontend UX improvements to help users understand what they can create.

## Changes

### 1. Smart Create "Recommended" Badge

- Add a small badge/pill next to "Smart Create" in Create dropdown
- Text: "Recommended" in a subtle accent color
- Purpose: Guide new users to the AI-guided flow

### 2. Template Type Subtitles

- Add "(Audio Series)" subtitle under "Study Guide" and "Creative Story" in Create dropdown
- Purpose: Clarify these are templates, not separate entity types

### 3. Browse Label Standardization

- Change "My Series" → "My Audio Series" in Browse dropdown
- Purpose: Match the entity name used in Create dropdown and Activity page

### 4. Activity Section Info Tooltips

- Add "?" icon next to section headers in Activity page grouped view
- On hover/click, show entity description from the entity dictionary
- Descriptions:
  - **Quick Audio**: "A single AI-generated audio episode on any topic"
  - **Audio Series**: "Multi-episode audio course with AI-generated curriculum"
  - **Classrooms**: "Interactive audio lessons with built-in quizzes"
  - **Written Content**: "Blog posts, newsletters, LinkedIn posts, and more"

### 5. Generation Time Estimates

- For items in generating/running status, show estimated time
- Heuristic-based (no API changes needed):
  - **Quick Audio**: "Usually 3–5 min"
  - **Audio Series**: "Usually 5–15 min per episode"
  - **Classroom**: "Usually 5–10 min per lesson"
  - **Writing**: "Usually 1–2 min"

### 6. First-Visit Welcome Banner

- Show once for new users (track via `localStorage`)
- Content: "Welcome! Not sure where to start? Try Smart Create — describe what you want, and we'll figure out the rest."
- CTA button: "Try Smart Create" → `/create-smart`
- Dismissible (X button), auto-hides after dismissal
- Shows on dashboard/home page only

### 7. Help Page (`/help`)

- New route: `/help`
- Sections:
  - **"What can I create?"** — Entity cards with descriptions
  - **"How it works"** — 3-step overview (Describe → AI Creates → Listen/Read)
  - **"FAQ"** — Common questions
- Link to help page from footer and from "?" tooltips
- Content sourced from `entity-dictionary.md` help content starters

## Files Modified

| File | Change |
|------|--------|
| `components/Navbar.tsx` | Badge, subtitles, label change |
| `components/NavDropdown.tsx` | Render badge and subtitle fields |
| `app/activity/page.tsx` | Info tooltips, time estimates |
| `app/(main)/page.tsx` | First-visit banner |
| `app/help/page.tsx` | New help page **(new file)** |
| `components/InfoTooltip.tsx` | Reusable tooltip component **(new file)** |
| `components/FirstVisitBanner.tsx` | Welcome banner **(new file)** |
| `hooks/useFirstVisit.ts` | localStorage tracking **(new file)** |

## Out of Scope

- API changes (generation time API field)
- Video tutorials
- Creation quiz ("What should I create?")
- Sample/example content

## Testing

- Visual verification on staging
- Check all tooltips render correctly
- Verify `localStorage` tracking works (banner shows once, stays dismissed)
- Check mobile responsiveness of all new UI elements
