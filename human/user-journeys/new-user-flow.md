# New User Journey

> Step-by-step walkthrough of the first-time user experience.
> Use this to identify confusion points and improve onboarding.

---

## Persona → Entry Point Matrix

| Persona | Likely entry point | First creation | Confusion risk |
|---------|-------------------|----------------|----------------|
| **Content creator** | "I need a podcast" | Quick Audio | Low — clear path |
| **Educator** | "I want to build lessons" | Classroom | Medium — may not find Classroom immediately |
| **Job seeker** | "Help me prep for interviews" | Interview Prep | Low — dedicated nav item |
| **Student** | "I want to study for exams" | Study Guide template | Medium — "Study Guide" vs "Audio Series" |
| **Marketer** | "I need blog content" | Written Content | Low — clear path |
| **Explorer** | "What can this do?" | Smart Create | Low — AI guides them |

---

## The Journey: Step by Step

### Step 1: Landing Page → Sign Up

**What happens:** User arrives (likely from search, social, or referral). Sees the value prop. Clicks "Get Started."

**Current flow:**
1. Landing page shows content examples
2. "Get Started" → Clerk sign-up (Google, email, or SSO)
3. After sign-up → redirected to dashboard

**Potential confusion:**
- "What exactly can I create?" — Landing page should show concrete examples of each content type
- No free trial friction — good, no credit card required

---

### Step 2: First Dashboard Visit

**What happens:** User sees the main dashboard. The nav bar has two key dropdowns: **Create** and **Browse**.

**Current flow:**
1. Dashboard shows recent activity (empty for new user)
2. Create dropdown has 8 items
3. Browse dropdown has 5 items

**Potential confusion:**
- 8 items in Create might overwhelm a first-time user
- "Smart Create" vs specific types — unclear which to pick
- "Study Guide" and "Creative Story" look like separate entity types but are actually Audio Series templates

**Recommendation:** Smart Create should be visually prominent as the recommended starting point for new users. A first-time tooltip or banner like "Not sure where to start? Try Smart Create" would help.

---

### Step 3: Creating First Content

#### Path A: Smart Create (recommended for new users)

1. User clicks "Smart Create" → lands on `/create-smart`
2. Chat mode (default): AI asks "What would you like to create?"
3. User describes their idea in natural language
4. AI asks 2-3 clarifying questions (content type, duration, style)
5. AI generates a plan → user reviews
6. User approves → creation begins

**This is the smoothest path.** The AI handles format selection, so the user doesn't need to know the difference between Quick Audio and Audio Series upfront.

#### Path B: Direct creation (knows what they want)

1. User picks a specific type from Create dropdown (e.g., "Quick Audio")
2. Routed to Smart Create with a pre-selected template
3. Chat or batch form to fill in details
4. Generation starts

**Works well for returning users** who already know the content types.

#### Path C: Template-based

1. User picks "Study Guide" or "Creative Story" from Create dropdown
2. Routed to Smart Create with that template pre-loaded
3. Template provides sensible defaults

**Works well for specific use cases** that map cleanly to a template.

---

### Step 4: Waiting for Generation

**What happens:** Content generation takes 2-15 minutes depending on type and length.

**Current flow:**
1. User sees a progress indicator
2. For Audio Series: curriculum generates first, user can review before episodes
3. For Quick Audio: single progress bar
4. User can navigate away — Activity page shows status

**Potential confusion:**
- "How long will this take?" — No time estimate shown
- For Audio Series: the curriculum review step can confuse users who expect everything to be automatic
- Draft status on Smart Create courses: there's a brief period where the course shows as "draft" before syllabus generation starts (5-minute detection window handles this)

---

### Step 5: Consuming Content

**What happens:** Content is ready. User listens/reads.

**Current flow:**
1. Audio: inline player on the detail page
2. Written content: rendered text with format-specific styling
3. Classroom: lesson player + quiz after each lesson

**Potential confusion:**
- If audio files were deleted by GCS lifecycle (fixed — lifecycle rule removed, but older content may be affected): user sees "Regenerate" button with amber warning
- Share button is always visible — good for discoverability

---

### Step 6: Finding Past Content

**What happens:** User returns and wants to find something they created before.

**Current flow:**
1. Browse → My Activity: unified feed with all content types
2. Browse → My Series: just Audio Series and Interview Prep
3. Browse → My Classes: just Classrooms
4. Browse → My Writeups: just Written Content
5. Quick Audio is only visible in Activity (no dedicated "My Quick Audio" page)

**Potential confusion:**
- "Where are my Quick Audio episodes?" — They're in Activity but not in any dedicated sub-page. This is fine for now (most users have few Quick Audios) but may need a dedicated view as volume grows.
- "My Series" label vs "Audio Series" creation label — slight mismatch ("Series" is abbreviated in Browse)

---

## Confusion Points Summary

| Issue | Severity | Current Mitigation | Suggested Fix |
|-------|----------|-------------------|---------------|
| 8 items in Create dropdown | Low | Smart Create is first item | Add "Recommended" badge to Smart Create |
| Study Guide / Creative Story look like entity types | Low | They route to templates | Add "(Audio Series)" subtitle in dropdown |
| No Quick Audio dedicated page | Low | Available in Activity | Add if Quick Audio volume grows |
| "My Series" vs "Audio Series" | Low | Consistent enough | Consider "My Audio Series" when space allows |
| No generation time estimate | Medium | Progress indicators exist | Add estimated time remaining |
| Curriculum review step surprise | Medium | Shows review UI | Add brief explanation before review |

---

## Recommended Onboarding Improvements

### Short-term (no new features needed)
1. **First-visit banner**: "Welcome! Start with Smart Create — describe what you want, and we'll figure out the rest."
2. **Empty state messaging**: Already implemented on Activity page — ensure each empty state explains the content type clearly.
3. **Tooltip on section headers**: Add "?" icon on Activity page section headers linking to entity descriptions.

### Medium-term (small features)
1. **Guided tour**: 3-step overlay pointing to Create, Activity, and the first content type.
2. **"What should I create?" quiz**: Quick 2-question flow that recommends a content type based on the user's goal.
3. **Sample content**: Pre-populated example for each content type so users can see what the output looks like before creating their own.

### Long-term (help system)
1. **Help page**: `/help` with entity descriptions, FAQ, and how-to guides.
2. **Contextual help**: "Learn more" links in creation flows that explain what each setting does.
3. **Video tutorials**: Short walkthroughs for each content type.

---

*This document should be updated whenever the navigation or creation flows change.*
