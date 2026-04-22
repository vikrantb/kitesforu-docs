# Agent 3 — Information Architecture & Action Hierarchy

Scope: zoom out and structure. What does the homepage *job* actually reduce to once the rest of the app surface is taken into account?

## Current app-surface map

### Top-level routes (`kitesforu-frontend/app/`)

| Route | Purpose | Who it's for |
|---|---|---|
| `/` | Homepage (split: SignedOutHero vs signed-in dashboard + unified chat launcher) | Everyone |
| `/create-smart` | **Primary creation flow** — AI chat-based planner; every template deep-links here with `?template=…` | Signed-in (auth-gated inline) |
| `/create` | Landing surface: hero input + 5 template sections (Career / Learning / Teaching / Stories / Professional) — ALL cards deep-link to `/create-smart` | Everyone, but effectively a directory page |
| `/create2` | Pure redirect → `/create-smart` | Legacy compat |
| `/library` | **Primary consumption flow** — unified list with filter pills (`all / interview_prep / course / class / writeup`), search, sort | Signed-in |
| `/interview-prep` | Hub for interview-prep (own landing, `/mock`, `/hub`, `/create` sub-routes) | Interview persona |
| `/courses/[courseId]` + `/courses` | Course detail + list | Course consumers |
| `/classes` + `/classes/[classId]` + `/classes/learn` + `/classes/join` + `/classes/library` + `/classes/import-scorm` | Full class LMS surface | Teachers + learners |
| `/writeups/[writeupId]` + `/writeups/create` | Writing surface | Writers |
| `/studio/[jobId]` | Podcast Studio — SSE streaming segments, primary post-create destination | Creators mid-generation |
| `/listen/[courseId]` | Focused listening surface | Course consumers |
| `/progress/[jobId]` | Generation status | Any creator |
| `/clarify/[jobId]` | Mid-generation clarify flow | Any creator |
| `/car-mode` | Hands-free consumption mode (voice-first, driving) | Commuters |
| `/drive` | (Related to car-mode / live generator) | Commuters |
| `/dashboard` | Signed-in dashboard page | Signed-in (feels like a duplicate of home dashboard) |
| `/activity` | Activity feed | Signed-in |
| `/pricing` | Tier pricing | Everyone (conversion) |
| `/settings` | Account settings | Signed-in |
| `/onboarding` | First-run onboarding | New users |
| `/help` | Help | Everyone |
| `/whats-new` | Changelog | Everyone |
| `/admin` | Admin panels | Staff only |
| `/debug` | Debug panels | Staff only |

### Navbar (desktop, persistent everywhere)
`Logo | Home | Library (signed-in) | Search (⌘K palette) | + Create (→ /create-smart) | Pricing | Credits | Avatar + Theme toggle`

Mobile top bar: `Logo + Search icon + SignIn/UserButton` only — everything else routes through bottom-nav.

### BottomNav (mobile, persistent on every page)
5 tabs: `Home | Library | + Create (centerpiece, elevated) | Search | Profile`

### What is reachable ONLY from the homepage (not from shell)
1. **First-visit product story** (`SignedOutHero` — flagship tiles, Mastery Loop, value tiles, "Begin your first session")
2. **HomeDashboard** for signed-in users: greeting, "Continue" (in-progress items), "Recent," "Quick Actions" per-persona, "Recommended for you," CreatorStatsStrip
3. **FirstVisitBanner**
4. **CentralChatLauncher** — the unified "say anything" hero input (duplicates the `/create` hero input, and duplicates the `+Create` button destination)
5. **PersonaSection grid** — 8 personas × N templates, currently collapsed behind `<details>` in unified mode, primary surface in legacy mode
6. **Value-prop cards** (4 marketing tiles) + BottomCTAs ("Begin your next session" → `/create-smart`)
7. **"Drive (Car Mode)" peer button** + "Continue listening" button (duplicates `/library`)

Only items 1, 2, 3, and *maybe* 6 are truly homepage-native. Items 4, 5, 7 either duplicate the shell or duplicate `/create`.

---

## The "homepage job" question

### What the homepage currently tries to be
**Both simultaneously (the cop-out).** Evidence:

- `SignedOut` branch → full-product marketing story (hero, 3 flagship cards, Mastery Loop, value tiles, pricing-adjacent CTA). This is a first-visit *acquisition* page.
- `SignedIn` branch → greeting dashboard, Continue/Recent, Quick Actions, Recommended, CreatorStatsStrip, CentralChatLauncher, peer car-mode button, collapsed persona grid, value-tile cards (!), BottomCTAs.
- The signed-in branch **still renders the marketing ValueCard grid and BottomCTAs** — the same "stats you already know" cards a first-visit user sees. That's the cop-out: the same marketing content is served to both audiences.
- Feature flag `feature_home_unified_chat_live` is a partial simplification (collapses the persona grid) but leaves the duplicated surfaces intact.

### What it *should* be
**(b) be the primary entry for returning users to take their next action**, with a minimal signed-out variant.

Reasoning grounded in the route map:
1. Acquisition already has a dedicated surface: `SignedOutHero` is a standalone component. It can live under its own route or behind a clean split. The home URL does not need to serve two audiences.
2. `/create-smart` is the real conversion machine — it's chat-based, it adapts to every input type, and every path on the site already funnels through it. The homepage's job for a returning user is to get them to `/create-smart` (or back into an in-progress job) in **one click**.
3. `/library` + BottomNav + Navbar already solve "find my stuff." The homepage doesn't need to re-solve consumption — it needs to resolve *next-action ambiguity* for the user standing at `/`.
4. The homepage has the **HomeDashboard** component doing real signal work (Continue/Recent/Quick Actions). That's the thing that earns the URL. The persona grid, value cards, peer car-mode button, and collapsed-details grid are all low-leverage additions competing with it.

---

## Action hierarchy proposal

Candidate actions currently on the homepage, ranked by leverage:

### Primary (exactly one, above the fold, unmissable)
- **Start a new creation** — via `CentralChatLauncher` (voice + text → `/create-smart?text=…`).
  - This is the single highest-leverage action: it's the conversion funnel, it works for every content type, it's auth-gated downstream so signed-out users can still engage.
  - **Caveat**: if the user has an in-progress job, the primary should flip to **"Resume your generating episode"** (one-click back to `/studio/[jobId]`). Anyone mid-generation who lands on `/` is there by accident.

### Secondary (supports the primary; 1–2 max)
- **Continue listening** — link to the most recent unfinished item, resolves directly to `/listen/[id]` or `/studio/[id]`. Not a link to `/library` (that's in the shell). Link to the *specific next item*.
- **Drive / Car Mode** — only for users whose history shows audio-consumption behavior (mobile, recurring sessions). For a first-time signed-in user, this is noise. Promote conditionally.

### Tertiary (below the fold, optional)
- **Quick Actions** (persona-targeted 4-up template shortcuts) — keeps power-users one click from their habitual template.
- **Recommended for you** — personalization surface; earns its space only if the recommendation is actually tailored (current impl uses `getRecommendedTemplates(persona)`, which is static).
- **Recent** (3–5 items max, with a "See all" link to `/library`).

### Belongs-elsewhere (move to another page; name the page)

| Element | Where it should live | Why |
|---|---|---|
| PersonaSection grid (8 personas) | `/create` | `/create` already IS the template directory with 5 sections + 24 template cards. The homepage's persona grid is a worse-formatted duplicate of `/create`. |
| "Begin your next session" BottomCTA button | Delete. | It's just another button to `/create-smart` that duplicates the navbar `+Create` button and the CentralChatLauncher. Three CTAs to the same URL on one page. |
| ValueCard grid (4 marketing tiles) for signed-in users | `/pricing` or `/whats-new` | Value-props are acquisition content. A signed-in user does not need "Stories that Live" pitched to them. |
| SignedOutHero flagship grid (3 cards) | Keep on homepage signed-out variant, but ideally move to `/` route's signed-out branch only (see split proposal below). |
| "Continue listening" button → `/library` | Replace with deep-link to the actual in-progress item; if none, hide. A button that just opens Library duplicates the shell. |
| FirstVisitBanner | Promote to onboarding (`/onboarding`) or keep only for signed-in users with 0 items. Currently renders for all signed-in users. |
| CreatorStatsStrip | Dashboard (`/dashboard`) or `/activity` | Stats are a returning-power-user concern, not a next-action driver. |
| Search trigger on homepage body (if any) | Delete — it's in the navbar + bottom-nav. |
| Pricing link duplication | Already only in navbar — good. Do not add to home body. |

---

## Content-type hierarchy

KitesForU has 5 content types: **podcast, course, class, writeup, interview-prep**.

### Ranking for homepage promotion

1. **Interview-prep** — DOMINANT. Evidence:
   - `SignedOutHero` already leads with it exclusively: "Master your next interview," "The B2B Mastery Companion," all 3 flagship tiles are interview-prep (Voice-First Simulation, Interactive Mock, Custom Interview Series).
   - Library filter pills lead with `interview_prep` as its own top-level pill (it's technically a course subtype but is surfaced separately).
   - `/interview-prep` is the only content type that owns its own hub + mock + create sub-routes.
   - Most recent product investment per MEMORY (R1/R2/R3 roadmap, Mastery Trace, gap-analysis, RTL, scenario guidance, USER PROFILE injection) is interview-prep heavy.
   - Commercial signal: this is the B2B wedge. Everything else is supporting content.

2. **Podcast / course (audio series)** — STRONG SECONDARY. The infrastructure is the deepest here (the entire `kitesforu-workers` pipeline, personas, voice intelligence, Car Mode consumption). It's the "what KitesForU actually does under the hood" story.

3. **Class** — TERTIARY. Has its own full LMS surface (`/classes/*`), is teacher-facing, and doesn't compete for the same user intent as interview-prep. Belongs on `/create` but not on the homepage hero.

4. **Writeup** — TERTIARY. Paired with audio in some templates (newsletter, executive summary). Discoverable through `/create` template sections. Does not need homepage real estate.

5. **Podcast as standalone** — ABSORBED. Most "podcast" use cases now flow through templates (horror series, comedy, bedtime story). The template directory on `/create` handles this. No dedicated homepage promotion needed.

### Recommendation
- Homepage signed-out variant: **lead with interview-prep** (current `SignedOutHero` is correct).
- Homepage signed-in variant: **intent-agnostic.** The CentralChatLauncher absorbs all 5 content types into one input. Don't promote any specific type in the hero. Promote *per-user* based on their `primary_persona` via Quick Actions.

---

## Signed-out vs signed-in: one homepage or two?

### The architecture case for ONE homepage
- Fewer URLs, less state, simpler routing.
- Clerk's `<SignedIn>`/`<SignedOut>` primitives make the branch cheap.
- SEO benefits from a single canonical landing URL with marketing content.
- Fewer "where do I go?" decisions for internal linking.

### The architecture case for TWO homepages
- The jobs are **genuinely different**: a first-visit stranger and a returning power-user share zero primary actions. Collapsing both into `/` forces every element on the page to justify itself to both audiences, which is why the current page renders marketing ValueCards to signed-in users (the cop-out).
- The signed-in "homepage" is effectively a dashboard, not a homepage. Dashboards have different density, different section hierarchy, and different analytics (retention vs acquisition).
- Two homepages = the signed-out URL becomes a **proper landing page** that marketing can A/B test without affecting the dashboard. Today, every marketing tweak risks the dashboard and vice versa.
- `/dashboard` already exists in the route map — it's just unused for this purpose.

### Recommendation: **Two homepages, one URL.**

Keep `/` as the single URL (preserves SEO, existing inbound links, shell navigation). But treat the two Clerk branches as **distinct product surfaces with distinct components, distinct analytics events, and distinct owners.** Concretely:

- `SignedOut` → pure landing page. Single CTA. Delete the duplication with `/create` template content.
- `SignedIn` → pure dashboard. Delete the marketing ValueCards, delete the persona grid duplicate, delete the "Begin your next session" CTA-to-nowhere, delete the generic "Continue listening → /library" button.
- `/dashboard` → redirect to `/` (signed-in users land on the dashboard by visiting home; power-users who bookmarked `/dashboard` keep working).

The split is architectural, not URL-level. The anti-pattern is not "one URL for both" — it is "the same components try to serve both."

---

## Competing-with-the-shell audit

Elements currently on the homepage that are *also* in navbar or bottom-nav:

| Homepage element | Also in | Verdict |
|---|---|---|
| `CentralChatLauncher` (primary hero input → `/create-smart`) | Navbar `+Create` button + BottomNav elevated `+` center tab | **Keep on home, it's the high-fidelity version.** The navbar button is a shortcut; the hero input is the full affordance (voice + text + placeholder rotation). But recognize they target the same URL. |
| "Continue listening" button → `/library` | BottomNav `Library` tab + Navbar `Library` link | **Remove or redirect to specific in-progress item.** A generic button to `/library` is a redundant third entry point. |
| "Drive (Car Mode)" peer button | Not in shell | **Keep**, but demote. Car Mode is only reachable from the homepage and from inside a playing session. That's too few entry points for a flagship consumption mode; either add to the shell or promote only when history warrants. |
| `BottomCTAs` ("Begin your next session" → `/create-smart`) | Navbar `+Create` + BottomNav `+` + hero `CentralChatLauncher` | **Delete.** Four entry points to the same URL on the same viewport. |
| Persona grid (collapsed `<details>`) → `/create-smart?template=…` | Navbar `+Create` → `/create-smart` → template selection inside the planner | **Move to `/create`.** `/create` is already the template directory with more structure (5 sections, accent bars, difficulty/meta). The homepage persona grid is a weaker duplicate. |
| FirstVisitBanner | Onboarding | **Keep only for 0-item users.** Once the user has any item, this is noise. |
| SignedOut flagship tiles (3 interview-prep cards) | Not in shell (shell is empty when signed out) | **Keep**, this is the sign-out CTA farm; it is the page's job. |

**Rule of thumb**: if a click leaves the homepage to a URL also reachable from the shell, that click's entry point on the homepage must add *information* the shell doesn't (a status, a count, a recommendation, a preview). Pure "here's a button to the same place" duplicates should be cut.

---

## IA rules the simplification should honor

1. **If it's in the shell, don't put it on the homepage as a bare link.** Only include shell-duplicate destinations if the homepage version carries extra information (e.g., "3 items in progress" > a bare `/library` link).
2. **One hero CTA per viewport.** The signed-in home currently has four paths to `/create-smart` above the fold (hero input, peer button, navbar button, bottom-nav `+`). Pick one primary, demote the rest.
3. **Never show empty-state and non-empty-state side by side.** The current signed-in dashboard renders the marketing ValueCards *and* the user's dashboard simultaneously. Pick one based on user state.
4. **Every homepage element must serve the current user state.** Signed-out users don't need dashboards; signed-in users don't need "Stories that Live" marketing tiles. The `<SignedIn>`/`<SignedOut>` primitives must wrap each section, not just the top-level branch.
5. **Promote specific items, not categories.** "Continue listening" as a button that opens `/library` is worse than no button. A card that says "Continue: *The Midnight Line*, Episode 3 of 5" is worth the space.
6. **Template directories belong on `/create`, not `/`.** If a user wants to browse 24 templates, they're in browse-mode, not next-action mode. The homepage is for next-action.
7. **Personalization must be real to earn its space.** `Recommended for you` backed by `getRecommendedTemplates(persona)` (static per persona) is a lie. Either wire to actual history or delete the section.
8. **Content-type equality is a lie.** Interview-prep earns hero space on signed-out. No content type earns hero space on signed-in (the CentralChatLauncher absorbs them all). Don't promote 5 types equally.
9. **Dashboards should degrade gracefully for 0-item users.** First-visit signed-in users should see onboarding copy + template shortcuts, not empty Continue/Recent sections followed by marketing tiles.
10. **Feature flags are a half-measure.** `feature_home_unified_chat_live` collapses the persona grid behind `<details>` but keeps the marketing ValueCards and BottomCTAs. Pick an architecture and commit; feature-flagged coexistence of legacy + unified hides the cost of the legacy surface.

---

## Summary (structure, not design)

- Job of the homepage: **next-action router for returning users**; secondary: **conversion landing for strangers**. The two jobs share a URL but should NOT share components.
- Primary action: single hero input → `/create-smart`. Nothing competes with it.
- Secondary: surface *specific* in-progress / recent items, not generic shell-duplicate buttons.
- Content-type hierarchy: interview-prep dominates signed-out; signed-in is type-agnostic.
- Biggest cut: the persona grid (belongs on `/create`), the marketing ValueCards (belongs on `/pricing` or deleted), the redundant BottomCTAs, the generic "Continue listening → /library" button.
- Biggest consolidation: `/create` already owns template browsing; `/library` already owns consumption. The homepage should not re-implement either.

## Key file paths referenced

- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/app/page.tsx` — homepage root (split logic, unified vs legacy, all current homepage surfaces)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/components/Navbar.tsx` — shell navigation (desktop)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/components/BottomNav.tsx` — shell navigation (mobile)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/components/home/` — every homepage-only component (CentralChatLauncher, HomeDashboard, SignedOutHero, PersonaSection, HeroSection, CreatorStatsStrip, TemplateChip)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/components/HomeCTAs.tsx` — BottomCTAs component (duplicate CTA)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/app/create/page.tsx` — template directory (5 sections × 4-6 cards, already does what the persona grid tries to)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/app/create-smart/page.tsx` — actual creation funnel (chat planner, auth-gated inline)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/app/library/page.tsx` — consumption surface (filter pills for all 5 content types)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/lib/templates.ts` — persona + template definitions (source of truth for the persona grid)
