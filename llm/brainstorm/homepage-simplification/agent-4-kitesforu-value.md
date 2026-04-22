# Agent 4 — KitesForU User Intent & Value

## The actual wedge

**One sentence:** KitesForU turns *your* specific situation (your resume, your exam, your kid's bedtime, your commute) into a professionally produced, voice-first audio experience in under two minutes — and, in interview prep, layers an adaptive mastery loop on top of it.

No competitor hits all four of: on-demand topic **+** customizable tone/voice **+** genuine audio production quality **+** mastery/coaching feedback. Podcasts are on-demand but not personalized. Audiobooks are produced but not on-demand. TTS tools are instant but sound generic. Interviewing.io has coaching but no audio curriculum. That intersection is the wedge.

Honesty check: the *audio generation* half of the wedge is real and shipped (18 genre profiles, 20 named personas, voice intelligence system, 53s generation, $0.083/ep). The *mastery loop* half is real only inside interview prep — everywhere else "mastery" is aspirational copy, not a shipped experience. The homepage currently hides the first and over-promises the second.

## The real users

Evidence for "who stuck" is **missing** from the repo — I found no retention data, no cohort analysis, no usage-by-persona breakdown. The `user-persona-bible-index.md` enumerates 27 aspirational personas from deep research, but nothing tells me which are *actually using* the product past week 1. What follows is inferred from where shipped effort is concentrated.

Where the product has actually invested engineering in the last 60 days (Mar–Apr 2026):
- **Interview prep stack**: mastery ribbon, STAR rubric, Elo adaptation, voice mock, company-specific research, curriculum builder, gap analysis — ~30+ PRs, its own `/interview-prep/*` namespace, its own `SignedOutHero`.
- **Audio creation quality**: voice personas, naturalness system, genre craft rules, quality auto-regeneration — ~60+ PRs, the thing the pipeline actually excels at.
- **Car Mode**: promoted to a homepage peer button, hands-free consumption of episodes.

Those three areas are where the product has voted with its commits. So:

**Primary persona (inferred — evidence: proposal named "Active Interviewer", `/interview-prep/*` surface area, SignedOutHero hero copy):**
A mid-to-senior career professional with an interview in the next 2–6 weeks. They type "Google PM interview" or upload their resume + JD. They want company-specific prep they can both *practice* (mock) and *consume* (audio course on commute). They're willing to pay because missing this interview has a five-figure cost.

**Secondary persona (inferred — evidence: voice intelligence system, persona system, Car Mode hardening, episode feedback capture):**
An audio-native consumer who uses KitesForU the way others use podcast apps — but for topics no podcast covers. Horror stories, bedtime tales, meeting prep, a 5-minute explainer on a Hacker News thread. Drives to Car Mode. The per-episode economics ($0.083) imply this is the volume persona.

**The users the product THINKS it's for but probably isn't:**
- **K-12 teachers, university professors, corporate trainers.** These show up as three of the nine scenarios, claim 22 flavor variants in the persona bible, and are name-checked throughout. But: no shipped compliance mode, no SCORM admin panel (LMS export exists as a standalone repo, not wired into the product homepage), no multi-seat billing, no classroom management. These are prospectus personas, not shipped personas. The homepage should stop advertising to them until the surface actually exists.
- **"Executives" as a standalone segment.** The `SignedOutHero` says "engineered for executives" — but nothing in the product behavior gates or tailors to seniority beyond the Elo difficulty slider. It's aspirational positioning language, not an audience.
- **Solo podcasters with RSS distribution.** Proposed (leverage option #2), not shipped. The current homepage chip "Make a podcast" over-promises against this.

## Jobs-to-be-done

Ranked by evidence of shipped support, not by aspirational roadmap:

1. **"I have an interview in N weeks — get me ready."** (Highest evidence: mastery loop, STAR rubric, Elo, company research, curriculum builder, voice mock, mastery trace, persistent gap analysis.)
2. **"Turn this thing I'm curious about into a podcast episode I can listen to on my commute."** (Second-highest: voice personas, genre craft rules, streaming generator, Car Mode, 53s generation, episode feedback.)
3. **"Make me a bedtime / horror / comedy story, produced well enough that I'd play it on a real speaker."** (Third: voice intelligence system, sub-genre calibration, series memory proposal, persona-driven TTS.)
4. **"I need to sound smart in a meeting / on an exam in the next 24 hours."** (Weaker evidence: curriculum builder handles this generically via scenario-type routing, but there is no "Meeting Prepper" surface; it collapses into generic /create-smart.)
5. **"I have an audio I liked — continue or extend it."** (Weakest: 'Continue listening' on home, library resume, but Series Memory is proposed, not shipped. Don't lead with this yet.)

Jobs **NOT** to claim on the homepage despite the scenario list suggesting them: classroom lesson authoring, corporate training rollout, university lecture companion, professional writeup generation. The `writeup` scenario in `scenario-guidance.ts` exists in the router but has no destination page that rewards the user. It's a scenario label in search of a product.

## The single thing a visitor must understand

If a visitor only takes one concept away in 5 seconds:

**Concept:** *You describe your situation in your own words; KitesForU produces the audio experience (and, for interviews, the practice loop) that fits exactly your situation — in about a minute.*

Three phrasings:

- **One-liner:** "Describe it. We produce it. In about a minute."
- **Button copy:** "Say what you want to hear."
- **Full sentence:** "Tell us what you're preparing for, curious about, or falling asleep to — KitesForU turns it into a produced, voice-first experience tuned to your situation, in under two minutes."

## What the homepage currently says vs what it should say

**Current signed-out hero copy** (from `components/home/SignedOutHero.tsx` lines 122–140):

> Eyebrow: "The B2B Mastery Companion"
> H1: "Master your next interview."
> Subhead: "Professional skill acquisition through voice-first simulations, adaptive mock interviews, and custom audio curricula — engineered for executives."
> CTA: "Begin your first session"

**Current signed-in hero copy** (from `components/home/HeroSection.tsx` lines 88–104, same file also used in the legacy branch):

> Eyebrow: "The B2B Mastery Companion"
> H1: "Master your next interview."
> Subhead: "Professional skill acquisition through voice-first simulations and deep research."

**Diagnosis:**
- The signed-out page positions KitesForU as a **B2B interview-prep SaaS for executives**. This is ~1/3 of what the product actually ships. It excludes the entire audio-creation wedge (podcasts, bedtime, horror, explainers, Car Mode) — which is where ~60+ of the last 90 PRs went.
- The signed-in page keeps the same "Master your next interview" headline, then the page body immediately contradicts it with "Make a podcast / Tell a story / Study audio" chips, persona cards for storytelling/writeup/k12, and a Car Mode button. The homepage **says one thing and does another**.
- "B2B Mastery Companion" is jargon. A visitor has to parse three abstract nouns ("B2B", "Mastery", "Companion") to extract zero concrete information.
- "Professional skill acquisition through voice-first simulations" is a consulting-deck sentence. No user has ever said "I need to acquire professional skills."
- "Engineered for executives" actively repels the bedtime-parent, horror-listener, and exam-crammer users that the product already serves.

**Three alternative hero-copy drafts:**

1. *Topic-first:*
   H1: "What do you want to hear next?"
   Sub: "Type or say any topic — an interview you're prepping for, an exam, a bedtime story, a podcast you wish existed — and KitesForU produces the audio experience, tuned to your situation, in about a minute."
   CTA: "Start talking"

2. *Situation-first:*
   H1: "Your situation. Professionally produced. In a minute."
   Sub: "Describe what you're preparing for or curious about. We pick the voice, pace, depth, and format — and for interviews, we layer in a coaching loop."
   CTA: "Describe your situation"

3. *Dual-wedge-explicit (honest about both halves):*
   H1: "Audio that fits your situation."
   Sub: "Tell KitesForU what you're preparing for, studying, or telling the kids tonight. We produce the voice-first experience that fits — and for interviews, adaptive mock sessions that actually improve you."
   CTA: "Start with a sentence"

## Language inventory

Words the current homepage leans on, and whether they earn their space:

| Word / phrase | What it's doing now | What it should do | Verdict |
|---|---|---|---|
| "B2B" | Signals enterprise buyer; there is no enterprise purchase flow in the product | Nothing. Cut. | **Cut** |
| "Mastery" | Core metaphor for the interview loop (Mastery Trace, Mastery Loop, Mastery Ribbon) | Keep *inside* interview-prep surface; too narrow for homepage | **Demote to interview-prep only** |
| "Companion" | Positions product as ongoing relationship | Vague, consultant-speak | **Cut** |
| "Executives" | Filters audience, excludes 80% of actual users | Repels real users | **Cut** |
| "Professional skill acquisition" | Academic register, zero concrete | Should be "prep", "practice", "learn", "listen" | **Rewrite** |
| "Voice-first" | Real differentiator for interview mock AND for audio consumption | Keep — but pair with a concrete output word ("voice-first audio", "voice-first practice") | **Keep, sharpen** |
| "Simulations" | Accurate for voice mock; oversells for everything else | Restrict to interview-prep card, not headline | **Scope down** |
| "Curricula" | Accurate for Custom Interview Series; Latin-register | Replace with "course" or "audio series" | **Rewrite** |
| "Trace" | Internal noun (Mastery Trace) leaked to marketing copy | Users don't know what a Trace is | **Cut from homepage** |
| "Pick your path" (legacy) | Hedges the decision onto the user | Homepage should reduce decisions, not multiply them | **Cut** |
| "Interview Ready / Ideas to Audio / Study Anywhere / Stories that Live" (value tiles) | Four competing promises — each reads like a separate product | Pick one, or unify under "Audio for your situation" | **Cut or unify** |
| "Master your next interview" | Strong, specific, concrete — but only covers 1/3 of the product | **Keep as a sub-promise inside the interview-prep surface**, not as the global H1 | **Demote to interior page** |

## Competing value propositions to cut

The homepage is currently a menu of nine half-propositions. Verdict per proposition:

| Proposition | Evidence shipped? | Verdict |
|---|---|---|
| "Master your next interview" | Yes, heavily shipped | **Keep** — as the flagship vertical card, not the global H1 |
| "Make a podcast" | Partially — generation works, RSS distribution does not | **Demote** — say "Make an audio episode", not "podcast" (podcast implies RSS) |
| "Turn coursework into audio" | Partial — generic generation works, no classroom/LMS surface | **Demote** — fold into generic "study audio" |
| "Tell a story" | Yes — voice personas, genre rules, sub-genres | **Keep** as a visible secondary |
| "Hands-free, eyes-free" (Car Mode) | Yes — shipped and hardened | **Keep** — but as a consumption-mode peer, not as an equal-weight creation mode |
| "Continue listening" | Yes — library resume | **Keep** — belongs in signed-in dashboard, not signed-out hero |
| "STAR precision / Adaptive difficulty / Mastery Trace" | Yes — but these are *features* inside interview-prep, not top-level promises | **Demote** — visible only on the interview-prep card, not the home value tiles |
| "Interview Ready / Ideas to Audio / Study Anywhere / Stories that Live" (the four bottom tiles) | These are literally four separate products on one row | **Cut three, keep one** — or replace all four with one unified "Audio tuned to your situation" frame |
| "Pick Your Path" (persona grid) | This *is* the decision the homepage should be making FOR the user, not surfacing TO them | **Cut from above the fold** — already collapsed behind `<details>` in unified layout, keep it there |

## What KitesForU is NOT

Crisp exclusions the homepage should stop suggesting:

1. **Not a B2B enterprise SaaS.** No multi-seat billing, no admin panel, no SSO onboarding flow exists in the product. The "B2B" eyebrow on the hero is fiction. Remove it. (If enterprise ever becomes real, give it a `/enterprise` page; don't let it colonize the consumer homepage.)
2. **Not a generic AI chat / general-purpose LLM workspace.** The central chat input invites "say anything" but the backend is specialized for audio experience production and interview coaching. When users type "write me a Python function" they'll leave disappointed. Either the input should be scoped ("Describe what you want to *hear*" or "Describe what you want to *prepare for*") or examples should anchor the scope.
3. **Not a classroom / LMS / corporate training platform.** The scenario labels (`k12-lesson`, `university-lecture`, `corporate-training`) are roads with no destination. Either ship the compliance/SCORM/roster surfaces to make them real, or drop them from the homepage until you do.

## One paragraph of honest marketing copy

> **Audio tuned to your situation — in about a minute.** Tell KitesForU what you're preparing for, curious about, or falling asleep to. We produce a voice-first audio experience that fits: the right voice, pace, depth, and format for exactly your situation. For interviews, we layer in an adaptive practice loop that improves with every answer. One input box. One minute. Your situation, in your ears.

(77 words. Leads with the wedge — situation-specific voice-first audio generation — and honestly scopes the "coaching" half of the product to where it actually ships: interviews.)

---

## Files referenced (absolute paths)

- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/app/page.tsx` — current routing, unified vs legacy layout
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/components/home/SignedOutHero.tsx` — current signed-out hero copy (the "B2B Mastery Companion / Master your next interview" hero)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/components/home/HeroSection.tsx` — current signed-in legacy hero copy (same headline as signed-out)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-frontend/lib/scenario-guidance.ts` — the 9 scenarios the router believes in
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-docs/proposals/user-persona-bible-index.md` — 27-persona bible (aspirational, not validated)
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-docs/business/strategy/seven-leverage-options-2026-04-19.md` — what leadership actually considers high-leverage
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-docs/business/features/done/interview-mastery-suite.md` — what the interview-prep half actually ships
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-docs/business/features/done/R1-unified-chat-entry.md` — explicit product-owner note that the general-purpose central input is "the simplest, most powerful entry point"
- `/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-docs/llm/brainstorm/homepage-simplification/codex-pm-ui-usability-audit-2026-04-21.md` — independent audit flagging the same two-stories problem
