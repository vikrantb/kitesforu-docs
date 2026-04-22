# Agent 1 — Homepage Friction Audit

Scope: diagnose, in evidence, what makes `app/page.tsx` "feel like it's doing too many things at once." No solutions proposed; that's Agent 3's job.

Evidence was taken from the live code at commit state on 2026-04-21. Both the signed-out flow (`SignedOutHero`) and the signed-in flow (`HomeDashboard` + either `UnifiedHome` or `LegacyHome`) were read end-to-end. The relevant feature flag `feature_home_unified_chat_live` defaults to `true` (`lib/feature-flags.ts:140`), so the **unified** signed-in layout is what real users see today.

Chrome surrounding the homepage (always mounted by `app/layout.tsx:109-124`):
- `components/Navbar.tsx` — top nav
- `components/BottomNav.tsx` — fixed mobile tab bar

---

## Signed-out homepage inventory

Rendered via `app/page.tsx:26-28` → `components/home/SignedOutHero.tsx`.

### Sections / blocks above the fold

Render order from `SignedOutHero.tsx:118-151`:

1. **Eyebrow label** — "The B2B Mastery Companion" — `SignedOutHero.tsx:122-126`. Treats the platform as B2B-executive only. Contradicts the rest of the product (K-12 teachers, bedtime stories, horror series — `lib/templates.ts:317, 264, 355`).
2. **Hero H1** — "Master your next interview." — `SignedOutHero.tsx:128-136`. Commits the entire homepage to a single vertical (interview prep). This is an actual product contradiction with the signed-in product catalog (see §"Specific evidence" below).
3. **Subhead** — "Professional skill acquisition through voice-first simulations, adaptive mock interviews, and custom audio curricula — engineered for executives." — `SignedOutHero.tsx:138-140`. Three product names stacked in one sentence; word "executives" narrows the audience again.
4. **Primary CTA** — `<Button>Begin your first session</Button>` opens Clerk modal — `SignedOutHero.tsx:142-146`.
5. **Micro-copy** — "Sign in with Google, Apple, or email — takes 30 seconds." — `SignedOutHero.tsx:148-150`.

### Sections below the fold

Same file, render order:

6. **Eyebrow + H2 "Three ways to practice. One Trace."** — `SignedOutHero.tsx:154-168`. Introduces a capitalized product concept ("Trace") with zero prior explanation.
7. **3 flagship product cards** — Voice-First Simulation, Interactive Mock Interview, Custom Interview Series — `SignedOutHero.tsx:34-59, 169-208`. Each card has its own `SignInButton` with a different `afterSignInUrl` (voice-preview, mock/text, interview-prep/create). That is **3 competing sign-in CTAs**, each routing to a different page.
8. **"How the Mastery Loop works" H2 + 3 numbered steps** — `SignedOutHero.tsx:212-242`. Step 01 copy ("Tell us the role… Voice, paste, or link — whichever is fastest") reads like onboarding the user hasn't asked for yet.
9. **4 value tiles** — STAR precision, Adaptive difficulty, Voice-first by default, Mastery Trace — `SignedOutHero.tsx:91-112, 245-265`. Four more proper-noun concepts ("STAR", "Elo-based calibration", "Mastery Trace", "event-sourced") the visitor has no context for.
10. **Final CTA card** — "Ready for the room?" + another `<Button>Begin your first session</Button>` — `SignedOutHero.tsx:267-285`.

### Competing CTAs (signed-out)

Clickable destinations grouped by intent:

- **Sign in and just start** (identical CTA, rendered 3× in different sections):
  - Hero `Begin your first session` → Clerk modal, no redirect — `SignedOutHero.tsx:142-146`
  - Final card `Begin your first session` → Clerk modal, no redirect — `SignedOutHero.tsx:279-283`
  - Navbar `Sign In` → Clerk modal, no redirect — `Navbar.tsx:88-92`
  - Mobile BottomNav `Sign in` (Profile tab) — `BottomNav.tsx:131-136`
- **Sign in + jump to a specific product** (each is also a sign-in CTA, just with a different `afterSignInUrl`):
  - "Sign in to open" on Voice-First Simulation card → `/interview-prep/mock/voice-preview` — `SignedOutHero.tsx:42, 184`
  - "Sign in to open" on Interactive Mock card → `/interview-prep/mock/text` — `SignedOutHero.tsx:49, 184`
  - "Sign in to open" on Custom Series card → `/interview-prep/create` — `SignedOutHero.tsx:57, 184`
- **Top-nav competing entry points visible without signing in**:
  - `Home` link — `Navbar.tsx:58`
  - Search trigger (opens command palette) — `Navbar.tsx:62`
  - `+ Create` button → `/create-smart` — `Navbar.tsx:63-69` (signed-out users clicking this are sent to a creation surface the hero page doesn't even mention exists)
  - `Pricing` — `Navbar.tsx:70`
- **Mobile BottomNav tabs visible to signed-out users**:
  - Home, Library, Create (floating `+` button → `/create-smart`), Search, Profile — `BottomNav.tsx:87-141`. The `Create` tab is visually elevated (`-mt-4 … rounded-full bg-brand-500 … ring-4` — `BottomNav.tsx:167-168`) which makes it the most prominent action on a mobile signed-out visit, while the hero screen tells them to sign in first.

Total distinct sign-in entry points on the signed-out homepage: **7** (3 hero/card, 1 final CTA, 1 navbar, 1 mobile bottom-nav Profile, 1 Pricing as a reach-around). Total distinct post-sign-in destinations baked into this page: **5** (`/create-smart` from navbar/bottom-nav, `/interview-prep/mock/voice-preview`, `/interview-prep/mock/text`, `/interview-prep/create`, `/` default).

---

## Signed-in homepage inventory

Rendered via `app/page.tsx:30-40`. Always renders `FirstVisitBanner` + `HomeDashboard`, then branches on the `feature_home_unified_chat_live` flag (default ON — `lib/feature-flags.ts:140`).

### Sections / blocks above the fold (unified path, the default)

Render order:

1. **FirstVisitBanner** — `app/page.tsx:33` → `components/FirstVisitBanner.tsx:7-42`. Gradient banner, "Welcome to KitesForU! … Try Start with AI" with a bright white pill button `Start with AI` → `/create-smart`. On a first visit this is the **first** creation CTA.
2. **Greeting header** — "Good morning, {name}." + contextual subline — `HomeDashboard.tsx:189-198`. Subline is one of 4 strings: "Pick a template…", "You have N items generating…", "N new episodes ready to play.", "What will you create today?" — `HomeDashboard.tsx:37-48`.
3. **CreatorStatsStrip** — 4 stat cards (Courses, Classes, Writeups, Podcasts) — `components/home/CreatorStatsStrip.tsx:40-73`. Each card is a link: when `count > 0` it goes to a library filter; when `count === 0` it routes to a **different** create surface (courses → `/create-smart`, **classes → `/classes/create`**, writeups → `/create-smart`, podcasts → `/create-smart`) — `CreatorStatsStrip.tsx:41-71`. Inconsistent create destinations across a single row.
4. **CentralChatLauncher** — "What do you want to create today? Say or type anything — we figure out the rest." Large input + mic + `Create` button → `/create-smart?text=...` — `components/home/CentralChatLauncher.tsx:145-201`. This H1 ("What do you want to create today?") **shares the same visual weight** as the greeting H1 above it — two H1-scale headings within ~200px of each other (`HomeDashboard.tsx:192` uses `text-2xl sm:text-3xl`, `CentralChatLauncher.tsx:147` uses `text-3xl md:text-5xl`).

### Sections below the fold (unified)

5. **Car Mode + Continue listening peer buttons** — `app/page.tsx:63-81`. Two equal-prominence buttons immediately under the chat input. Both orange/gradient + `rounded-2xl` + `py-4`. The left one is a gradient-filled primary (`bg-gradient-to-br from-orange-500 to-orange-600`); the right is a bordered secondary. Labels: "Drive (Car Mode) — hands-free, eyes-free" and "Continue listening".
6. **"Pick a template instead" collapsible `<details>`** — `app/page.tsx:86-115`. Summary pill contains 3 separate affordance signals: `📚` icon, "Pick a template instead", and parenthetical "(for a shortcut)". When expanded, it reveals the full 8-persona × ~4-template grid below.
7. **PersonaSection grid** (inside `<details>`) — `app/page.tsx:109-113` → `lib/templates.ts:717-790`. **8 persona cards** (Interview Prep, Quick Audio, Study Audio, Creative Stories, K-12 Classroom, University Lectures, Corporate Training, Self-Paced Learning), each containing **3–5 template chips** + an "Or start from scratch" link → `/create-smart` (`PersonaSection.tsx:72-97`).
8. **"Continue" section** (if in-progress items exist) — `HomeDashboard.tsx:217-240`. Horizontal-scroll carousel of up to 6 cards.
9. **"Recent" section** (if recent items exist) — `HomeDashboard.tsx:242-251`. Grid of up to 8 cards.
10. **QuickActionsSection** — `HomeDashboard.tsx:97-110`. 4 more cards titled "Quick Actions" with their own `→ All templates` link to `/create`.
11. **RecommendedSection** — `HomeDashboard.tsx:112-125`. 3 more cards titled "Recommended for you".
12. **4 bottom ValueCards** — "Interview Ready / Ideas to Audio / Study Anywhere / Stories that Live" — `app/page.tsx:118-141`. Same concepts as the 4 value tiles on signed-out hero but with different copy.
13. **BottomCTAs** — `Begin your next session` → `/create-smart` — `components/HomeCTAs.tsx:12-14`.

Note: the render order above is literal; steps 3–5 are in-between the greeting and the chat input. That means on a signed-in viewport the user sees **greeting → 4 stat cards → big "What do you want to create?" input → Car Mode + Continue listening → Pick a template toggle** before the fold on most laptops. That's five distinct calls to action stacked vertically before any content.

### Sections below the fold — Legacy path (flag OFF)

For completeness. `app/page.tsx:155-239` renders:

- `HeroSection` (`components/home/HeroSection.tsx`) with its OWN H1 "Master your next interview." + its own search input + its own 4-chip "QuickChips" row (Practice interview, Make podcast, Study audio, Tell story — `HeroSection.tsx:22-28`)
- "Pick Your Path" persona grid (expanded, not collapsed) — `app/page.tsx:160-176`
- 4 ValueCards — `app/page.tsx:178-201`
- "Find Your Starting Point" card with 8 more persona pill-links + BottomCTAs — `app/page.tsx:203-236`

Users on flag-OFF get **16+** creation entry points on a single page.

### Competing CTAs (signed-in, unified default)

Destination → where on the page you can click to reach it:

- `/create-smart` (bare): FirstVisitBanner pill, navbar `+ Create`, BottomNav `Create` tab, 3 of 4 CreatorStatsStrip empty-state CTAs, every persona card's "Or start from scratch", BottomCTAs. **≥7 distinct buttons on one screen point here.**
- `/create-smart?text=…`: CentralChatLauncher submit, voice submit.
- `/create-smart?template=<id>`: every TemplateChip inside PersonaSection (`TemplateChip.tsx:13-14`) + every QuickActionCard + every RecommendedCard (`HomeDashboard.tsx:85`). Across the persona grid + quick actions + recommendations, that's **30+ distinct template chip links** to `create-smart`.
- `/classes/create`: CreatorStatsStrip Classes empty-state only (`CreatorStatsStrip.tsx:52`). Inconsistent with peers.
- `/car-mode`: Car Mode peer button.
- `/library` and `/library?type=<kind>`: Continue-listening peer button, CreatorStatsStrip cards with count > 0, `See all` link, navbar `Library`, BottomNav `Library`.
- `/activity?kind=podcast`: CreatorStatsStrip Podcasts card when count > 0 — **different destination from the other 3 stat cards**, which go to `/library` (`CreatorStatsStrip.tsx:68`).
- `/create` (no `-smart`): QuickActionsSection "All templates" link — `HomeDashboard.tsx:102`. **This is the only surface that links to `/create`** and per `CLAUDE.md` "`/create` and `/create2` are redirects" — so the link is visible but lands on a redirect. Minor, but it's lint-rot exposed to the user.
- `/interview-prep/mock/text`: the one non-template quick chip in Legacy HeroSection.
- Clerk `UserButton`, `ThemeToggle`, credits pill (`Navbar.tsx:76-97`): secondary chrome.

---

## Friction findings — ranked by severity

### CRITICAL

- **Signed-out H1 hard-codes a single vertical that contradicts the product.** `SignedOutHero.tsx:128-136` says "Master your next **interview**." The signed-in home ships 8 personas including K-12 Classroom, Bedtime Stories, Horror Series, University Lectures, Corporate Training (`lib/templates.ts:717-790`). A first-time K-12 teacher who lands on the homepage has no affordance that the product is for them — the hero copy, all 3 flagship cards, all 3 "Mastery Loop" steps, and all 4 value tiles are interview-prep-specific. This is not "too many things at once" on the signed-out page; it's the opposite (mono-pitch), but it still produces confusion because the **navbar `+ Create` and BottomNav `Create` tab promise the rest of the product**. Marketing surface and product surface disagree.
- **Two competing "primary" surfaces on the signed-in home above the fold.** The greeting-as-H1 (`HomeDashboard.tsx:192`) and the chat-launcher-as-H1 ("What do you want to create today?" — `CentralChatLauncher.tsx:147`) both render at `text-2xl`/`text-3xl`+ within ~200px of each other. There is no single dominant focal point. Eye-tracking priority is ambiguous — the user reads one large heading, then immediately another equally large heading, with the stat strip between them and peer buttons (Car Mode / Continue listening) below.
- **Three coequal primary actions compete in the first viewport.** In render order: (a) `Start with AI` pill in FirstVisitBanner, (b) `Create` submit button in the central chat, (c) `Drive (Car Mode)` gradient button. Each is gradient/brand-colored, each is ~44px+, each is rounded-full/2xl. A returning user's eye has to choose between three "this is the one to click" affordances in a single screen.
- **Eight persona cards × up to 5 template chips = up to 40 creation entry points on the signed-in home.** `lib/templates.ts:717-790` declares 8 personas, each with a templates array of 3–5 (`interviewTemplates` has 4, `creativeTemplates` has 5, etc.). With QuickActions (4) and Recommended (3) added, the signed-in home exposes **47+ distinct "start a creation" paths** (1 chat input, 1 Car Mode, 1 Continue listening, 4 stat strip CTAs, 30+ persona template chips, 8 "Or start from scratch" links, 4 quick action cards, 3 recommended cards, 4 bottom ValueCards, 1 BottomCTAs, 1 navbar `+ Create`, 1 BottomNav `Create`). This is the literal "doing too many things at once" — *quantified*.

### HIGH

- **`<details>` progressive-disclosure pattern is a tell of unresolved IA, not a solution.** `app/page.tsx:86-115` wraps the 8-persona grid in `<details>` with the summary "Pick a template instead (for a shortcut)". The phrasing admits the conflict: we told you to use the chat, but also, here are 8 alternative ways in. The word "instead" signals to the user that they picked wrong by reading the chat input first. Hiding something behind a disclosure is a confession, not a design.
- **CreatorStatsStrip has inconsistent destinations.** `CreatorStatsStrip.tsx:40-73` — Courses/Writeups/Podcasts empty-states route to `/create-smart`, but Classes routes to `/classes/create`. When `count > 0`, Courses/Classes/Writeups all go to `/library?type=...` but Podcasts goes to `/activity?kind=podcast`. Same-shaped row, four different routing rules. A user who clicks "Podcasts: 3 total" and "Writeups: 2 total" ends up in two different apps.
- **The CTA hierarchy is violated.** A brand-gradient primary CTA is how the design system signals "the one thing to do next." On the signed-in homepage, the following **all** use `bg-gradient-to-r from-brand-` or `bg-gradient-to-br from-orange-`: FirstVisitBanner `Start with AI` pill, CentralChatLauncher submit, Car Mode button, BottomCTAs `Begin your next session`, BottomNav floating `+`. Five primary-weight CTAs means there is no primary.
- **Two different hero search inputs exist in parallel.** `HeroSection.tsx` (legacy, flag OFF) uses `classifyInput` to keyword-route to specialized pages; `CentralChatLauncher.tsx` (unified, flag ON) pushes to `/create-smart?text=…` and explicitly comments "Do NOT call `classifyInput` here" (`CentralChatLauncher.tsx:30-31`). Two homepage hero components with **opposite routing philosophies** live in the same tree. A flag flip changes what the box does. This is maintenance-friction that leaks into user-friction when the flag state is mis-set in review/production.
- **Signed-out value concepts are load-bearing proper nouns that get no definition above the fold.** "Mastery Trace", "Mastery Loop", "STAR precision", "Elo-based calibration", "event-sourced progress" — all appear in `SignedOutHero.tsx:108-112, 163-167, 91-112` before the user has clicked anything. "Trace" is referenced 3× before the one-line explanation at `SignedOutHero.tsx:110`.

### MEDIUM

- **FirstVisitBanner duplicates the chat launcher.** Its copy — "Not sure where to start? Try Start with AI — describe what you want, and we'll figure out the rest." (`FirstVisitBanner.tsx:28-30`) — is a paraphrase of the CentralChatLauncher subhead immediately below it ("Say or type anything — we figure out the rest." — `CentralChatLauncher.tsx:48-49`). The banner points at a button that leads to the exact surface whose input is already on the page.
- **Legacy `/create` redirect is still referenced from the signed-in home.** `HomeDashboard.tsx:102` — `<SectionHeader ... actionHref="/create" />`. Per `CLAUDE.md` gotcha #10 this is a redirect, so the user gets a URL-bar flicker.
- **Eyebrow label in two separate places says "The B2B Mastery Companion"** — `SignedOutHero.tsx:122-126` AND `HeroSection.tsx:89-93`. Third-person corporate phrasing on a consumer-feeling product; also conflicts with the signed-in greeting which is first-person contextual.
- **Mobile bottom-nav `+` Create button competes with the hero chat.** On mobile signed-in, the BottomNav's elevated `+` button (`BottomNav.tsx:167-168` — `-mt-4 … bg-brand-500 … ring-4`) is the most visually loud element on the screen at all times. The hero `Create` submit inside the chat is `rounded-xl bg-brand-500 px-4 py-2` — smaller. The persistent nav screams louder than the page's own primary CTA.
- **8 persona value propositions collapse to 4 ValueCards at the bottom** (`app/page.tsx:118-141`) that re-assert only 4 of them (Interview Ready, Ideas to Audio, Study Anywhere, Stories that Live). K-12 teachers, university instructors, corporate trainers, and self-paced learners — half the persona set — are silently dropped from the bottom summary.
- **"Recommended for you" has no visible reason.** `HomeDashboard.tsx:112-125` renders `RecommendedSection`. The backend default for a user with no primary persona in their profile is the static array `['comedy', 'concept-deep-dive', 'team-update']` (`home-quick-actions.ts:58`). Nothing in the UI tells the user why these 3 are recommended — it looks like personalization but it's a fallback constant for everyone without a detected persona.

### LOW

- Duplicate ValueCard definitions: `app/page.tsx:241-251` defines a local `ValueCard`; `SignedOutHero.tsx:244-265` inlines a near-identical structure. Maintenance debt.
- Typewriter placeholder rotation in two places (`HeroSection.tsx:44-71`, `CentralChatLauncher.tsx:68-98`) — animated placeholders are novel but also a moving target that competes with the user's own typing focus, and they run in both flag states.
- Final CTA card's "Ready for the room?" H2 (`SignedOutHero.tsx:270-274`) reintroduces interview-only framing ("room" = interview room) at the bottom, after 4 value tiles have already covered the same ground.

---

## Specific "too many things at once" evidence

- **How many distinct primary actions does the signed-out page offer in the first viewport?** Three: `Begin your first session` (`SignedOutHero.tsx:142-146`), `Sign In` in navbar (`Navbar.tsx:88-92`), and the elevated `+ Create` / mobile `+` tab on BottomNav (`BottomNav.tsx:104-111`) that bypass the hero entirely.
- **How many distinct primary actions does the signed-in page (unified, default) offer in the first viewport?** Five, all styled as primary: FirstVisitBanner `Start with AI` pill, CentralChatLauncher `Create` submit, Car Mode button, BottomNav `+` tab, navbar `+ Create`.
- **How many content types does the signed-in home visibly promote?** Seven named types surface without scrolling past the fold on a laptop: courses, classes, writeups, podcasts (CreatorStatsStrip labels — `CreatorStatsStrip.tsx:42-71`), plus audio "Continue listening" (`app/page.tsx:76`), Car Mode (`app/page.tsx:64-72`), and "templates" (`<details>` summary — `app/page.tsx:101`). The persona grid adds interview prep, study audio, creative stories, K-12, university, corporate training, self-paced learning (`lib/templates.ts:719-788`) — **8 more**.
- **How many value propositions does it state?** Signed-out tiles declare 4 (STAR precision, Adaptive difficulty, Voice-first by default, Mastery Trace — `SignedOutHero.tsx:91-112`). Unified signed-in adds 4 more (Interview Ready, Ideas to Audio, Study Anywhere, Stories that Live — `app/page.tsx:118-141`). Eight value tiles total, no clear hierarchy between them, no narrative thread connecting them.
- **Hierarchy violation** — there is no clear #1 action on either page:
  - Signed-out: 3 flagship cards + 2 "Begin your first session" buttons + 1 nav `+ Create` + 1 BottomNav `+` = **7 CTAs, all equally styled as primary within their section**. Cannot point at "the button."
  - Signed-in unified: FirstVisitBanner pill + CentralChatLauncher submit + Car Mode button + BottomCTAs + mobile BottomNav `+` = **5 primary-gradient buttons visible on typical viewports**.

---

## What a newcomer sees in the first 5 seconds

**Signed-out, desktop 1440×900, never seen KitesForU before:**

Eye lands top-left on the `KitesForU` logo (`Navbar.tsx:54`), scans right across `Home · Search · + Create · Pricing` (`Navbar.tsx:57-71`) — already a problem, they see `+ Create` and wonder if they can use the product without signing in. Eye drops to the hero. They read "The B2B Mastery Companion" (small caps eyebrow) — "B2B? I'm shopping this for myself." — then the 7xl gradient H1 "Master your next **interview**." Their frame is now "this is an interview tool." They read the subhead — three product names in one line: "voice-first simulations, adaptive mock interviews, custom audio curricula — engineered for executives." The word "executives" narrows further; the person who came looking to make bedtime stories for their kids now assumes they're on the wrong site.

Their eye finds the gradient `Begin your first session` button but hesitates — what *is* the session? They scroll. They meet three flagship cards, each with the same `Sign in to open` affordance, each promising a different product (Voice-First Simulation, Interactive Mock, Custom Series). Three sign-in walls for three products they haven't been sold on. They don't click. They keep scrolling hoping for a "what is this" moment. They meet "How the Mastery Loop works" — step 01, 02, 03 — this is product *process* documentation, not a pitch. They meet the 4 value tiles — STAR, Elo, Mastery Trace — four proper nouns they've never heard. Final CTA card repeats `Begin your first session`. They close the tab. The signed-in product catalog (classes, courses, writeups, stories, teacher tools) was never even hinted at.

Attention fragments after the H1 because the page proves, section by section, that it's a single-purpose interview tool — and the person's purpose may not be interviews.

**Signed-in, returning user, unified flag ON, landing on `/`:**

Eye lands on `Good morning, {name}.` (`HomeDashboard.tsx:194`) — a warm personal hook. Subline says "What will you create today?" (`HomeDashboard.tsx:47`). Eye drops to a 4-column stat strip — "Courses 3 · Classes 0 · Writeups 1 · Podcasts 8" — four little dashboards. They're not sure if they should click a stat (library) or ignore it. Eye continues down to ANOTHER large heading: "What do you want to create today?" (`CentralChatLauncher.tsx:147`) — nearly the same sentence the subline just asked. It's an input, so they focus on it, but a typewriter animation is cycling placeholders (`CentralChatLauncher.tsx:86-93`), so the caret target is moving. Just below, two gradient buttons compete: `Drive (Car Mode)` and `Continue listening`. Below those, a pill labeled `Pick a template instead (for a shortcut)` — the word "instead" makes them reconsider whether the input they were about to type in is the right path. They haven't made it past 500px of scroll, and they've been offered: an input, a voice mic, a car-mode mode, a continue-listening shortcut, a 4-card footprint grid, a collapsible template library, and a floating mobile `+`.

Attention fragments at step 2 — the user reaches the chat input, but the stat strip above it and the peer buttons below it both say "actually maybe start here." No surface has primacy. The user with low creation intent goes to Library via BottomNav; the user with strong intent either fights through the chat's animated placeholder or gives up and opens `<details>`. Nothing on the page reassures them they're *on* the page that will help them the most.
