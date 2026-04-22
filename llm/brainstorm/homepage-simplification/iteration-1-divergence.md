# Iteration 1 — Broad divergence

Purpose: generate 8 candidate homepage shapes without judgment. Judgment happens in iteration 2.

Each candidate answers one question: *what is the homepage fundamentally for?*

---

## Candidate A — "The input box is the product"

Shape: one centered input, a one-line promise above it, four example chips under it. Nothing else above the fold.

- Signed-out: "Audio tuned to your situation." Input: "What do you want to hear?" Chips: `Interview prep` · `Bedtime story` · `Podcast episode` · `Exam drill`. On submit → Clerk modal, then `/create-smart?text=…`.
- Signed-in: same shape. Below the fold: one "Continue" card (specific in-progress item) + Car Mode pill. Nothing else.

Precedents: Perplexity, v0.dev, Claude.ai, ChatGPT.

Kills: persona grid, value tiles, Mastery-Loop explainer, BottomCTAs, 4-card stat strip, FirstVisitBanner, "Continue listening → /library" duplicate.

Risk: the visitor who can't think of what to type stares at an animated placeholder and bounces.

---

## Candidate B — "Show, don't tell" (content-grid hero)

Shape: YouTube/Spotify-style — a live grid of 6-9 recently-produced public audio samples dominates the hero, with play-on-hover previews.

- Signed-out: "Listen to what KitesForU produces." Grid of 9 sample episodes (bedtime, horror, interview prep, comedy, science explainer, etc.). "Start your own" pill under the grid.
- Signed-in: grid of the user's own content + 3 public recs. "Make new" pill.

Precedents: YouTube, Spotify, Pinterest, Midjourney Showcase.

Kills: every piece of copy-based persuasion.

Risk: audio is hard to show-don't-tell (you can't scan thumbnails the way you scan video). Cold-start problem if public catalog is thin.

---

## Candidate C — "Pick your lane" (persona toggle)

Shape: two big cards above the fold — *"I want to listen"* vs *"I want to create"* — each opens a scoped surface.

- Signed-out: choice forced immediately.
- Signed-in: same; after first choice, state is remembered.

Precedents: Substack historical, Webflow, Framer.

Kills: the cop-out of serving both audiences with the same copy.

Risk: Agent 2 flagged this is the most-common-losing-pattern when the lanes overlap (KitesForU users are creator-consumers). A hard branch taxes every visitor to serve the split.

---

## Candidate D — "Dashboard only" (signed-in has no marketing; signed-out is a separate landing)

Shape: hard architectural split. `/` signed-in is pure dashboard (greeting, ONE hero CTA, ONE resume card). Signed-out is a separate stripped-down landing with one promise + one CTA. No shared components.

- Signed-out: ONE paragraph, ONE sample (autoplaying audio clip), ONE "Start with a sentence" input.
- Signed-in: ONE greeting, ONE in-progress-card-or-chat-launcher, ONE "Continue" item. Everything else demoted below or cut.

Precedents: Linear (signed-out), GitHub (signed-in dashboard), Notion (signed-in home).

Kills: both audiences sharing components, marketing ValueCards on signed-in, persona grid on home.

Risk: the signed-out page has to work harder than one promise can bear if KitesForU is genuinely dual-wedge.

---

## Candidate E — "Interview-prep forward, everything else honest-but-smaller"

Shape: lead with the flagship (interview prep), demote the rest to a honest sub-shelf.

- Signed-out: hero = interview-prep promise ("Master your next interview"), PRESERVED. Below fold: "Also for…" with 3 demoted cards (bedtime stories / podcasts / study audio), each one a small tile not a hero.
- Signed-in: CentralChatLauncher hero, interview-prep promoted if user hasn't tried it yet, else personalized.

Precedents: Cursor (developer tool that also works for designers/PMs, but leads with developers), Substack (writers first, readers second).

Kills: the pretend-equality between interview-prep and the other 4 content types on the homepage.

Risk: doubles down on the "executives" positioning that Agent 4 says repels 80% of actual users.

---

## Candidate F — "Situation-first" (Agent 4's recommended shape)

Shape: single honest promise that spans both halves of the real wedge.

- Signed-out: "Audio tuned to your situation — in about a minute." Input + sample-clip autoplay + one "Start with a sentence" CTA.
- Signed-in: same input, now personalized. Below: specific "Continue" + Car Mode.

Precedents: Linear ("for modern software teams" — one promise that honestly spans), Cal.com ("scheduling infrastructure" — single promise that covers several jobs).

Kills: the 9-half-propositions menu. Commits to the ONE real wedge Agent 4 identified: situation-specific voice-first audio generation.

Risk: "situation" is abstract. Needs concrete sub-examples under the input to earn the abstraction.

---

## Candidate G — "Resume first" (for signed-in users only; signed-out is completely separate)

Shape: the signed-in home does ONE thing — surface the specific in-progress item. Nothing else until that's handled.

- Signed-in above-fold: a card that says "Continue: *The Midnight Line*, Ep 3 of 5, 4:21 remaining" with a prominent Play. Below: a small input for new creation.
- Signed-out: the current `SignedOutHero` shape, but slimmed.

Precedents: Kindle home, Audible home, podcast apps.

Kills: the decision-fatigue problem for returning users. Single answer: "you were doing X, continue X."

Risk: users with nothing in progress hit an empty hero.

---

## Candidate H — "Conversational entry" (ChatGPT shape)

Shape: the homepage IS the chat. No greeting, no dashboard, just a threaded input that seeds a session.

- Signed-out: the landing IS `/create-smart` with Clerk-gate on submit.
- Signed-in: same — the homepage is the planner. Library is via shell.

Precedents: ChatGPT, Claude.ai, Character.ai.

Kills: the distinction between `/` and `/create-smart`. Collapses them.

Risk: removes the signed-out marketing surface entirely. Also collapses the "dashboard" job the Codex audit + Agent 3 both say the signed-in user needs.

---

## Summary matrix (for iteration 2 to judge)

| Candidate | Signed-out strength | Signed-in strength | Cuts the 47 CTAs? | Honest to the dual wedge? |
|---|---|---|---|---|
| A — Input-only | Medium | Medium | Yes | Partial (leads with creation, ignores consumption) |
| B — Content grid | Weak for audio | Medium | Yes | No (leads with consumption, buries creation) |
| C — Persona lanes | Medium | Medium | Yes but adds a branch | No (forces false binary) |
| D — Split by auth | Strong | Strong | Yes | Yes if copy is right |
| E — Interview-forward | Strong (if exec audience) | Medium | Partial | No (Agent 4 says this is the current miscast) |
| F — Situation-first | Strong | Strong | Yes | Yes (this is Agent 4's recommendation) |
| G — Resume-first | N/A | Strong | Yes | Partial (consumption-heavy) |
| H — Chat-is-the-page | Medium | Weak (no dashboard) | Yes | Partial (ignores the dashboard job) |

Candidates moving to iteration 2: **D, F, G**. B kept as a secondary-surface idea (the public showcase). Everything else cut.
