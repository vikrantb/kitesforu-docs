# KitesForU UX Patterns
<!-- last_verified: 2026-02-07 -->

## Brand
- Primary color: Orange (#F97316 family)
- Theme: Kites — playful, aspirational, educational
- Tagline: "Give Your Ideas Wings to Fly"
- Tone: Encouraging, professional, inspiring
- Emoji usage: Heavy in marketing (rocket, microphone, sparkles), minimal in UI

## Navigation Structure
```
/                               Landing page (hero, features, how-it-works, pricing preview)
/create                         Podcast creation (single form)
/create2                        Podcast creation v2
/courses/create                 Course creation (topic, duration, style, episodes, audience)
/interview-prep/create          Interview prep (wizard: resume -> JD -> help context)
/courses                        Course library (user's courses)
/courses/[id]                   Course detail + episode player
/progress/[jobId]               Podcast generation progress
/clarify/[jobId]                Clarification flow
/pricing                        Pricing page (4 tiers + PAYG)
/debug/[jobId]                  Podcast job debug (admin)
/debug/course/[courseId]        Course debug (admin)
/admin/model-router             Model router dashboard (admin)
```

## Design System
- Framework: Next.js App Router + Tailwind CSS
- Components: Radix UI based, in components/ui/
- Responsive: Mobile-secondary (desktop-first design)
- Dark mode: Not implemented
- Accessibility: Partial (some aria-labels, needs improvement)

## User Flow: Interview Prep (Current)
1. User lands on /interview-prep/create
2. Inputs: Resume (text/upload), JD (text/URL), Help context (free text, 1000 char)
3. Clicks "Generate" -> API extracts, creates course
4. Redirected to course page -> ALL episodes auto-generate immediately
5. User waits -> episodes appear as completed
6. User can play episodes

## User Flow: Course Creation (Current)
1. User goes to /courses/create
2. Fills form: topic, duration, style, episode count, language, audience
3. Clicks "Create" -> course + all episodes auto-generate
4. Redirected to course page -> episodes appear progressively

## User Flow: Podcast Creation (Current)
1. User goes to /create
2. Fills: topic, duration, style, audience
3. Clicks "Generate" -> redirected to /progress/[jobId]
4. Sees pipeline stages completing
5. Audio ready -> play/download

## UX Conventions
- **Podcast creation**: Minimal friction, single form, quick path
- **Interview prep**: Heavier input, wizard-style (but currently single page with sections)
- **Course creation**: Moderate input, single form
- **Progress tracking**: Stage-by-stage with status indicators
- **Debug pages**: Dense, expandable sections, for technical users

## Content Config (Duration Labels)
From lib/content-config.ts:
- 10 seconds -> "Quick test"
- 1 minute -> "Ultra-brief (testing)"
- 5 minutes -> "Quick lessons"
- 15 minutes -> "Detailed"
- 60 minutes -> "Full episode"

## Known UX Issues
1. Mobile experience is secondary — desktop-first
2. No onboarding for new users
3. Course episodes auto-generate without user review of curriculum
4. No "save draft" for interview prep inputs
5. No progress tracking within a course (which episodes listened to)
6. Episodes can't be individually regenerated
7. No way to preview/edit curriculum before episode generation
8. Limited error recovery UX (just "try again")

## Style/Form Options
- Styles: Explainer ("Clear, educational"), Storytelling ("Engaging narrative"), Interview ("Dynamic Q&A"), Motivational ("Inspiring call to action")
- Episode counts: 3 ("Quick introduction"), 5 ("Standard"), 7 ("Comprehensive"), 10 ("Deep dive")
