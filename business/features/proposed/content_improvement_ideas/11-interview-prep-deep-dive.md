# 11 — Interview-Prep Deep-Dive (content-craft)

**Status**: PROPOSED — grounding reference for prompt-engineering work
**Scope**: interview-prep scenario only (priority #1 in content deep-dive stack)
**Written**: 2026-04-22
**Grounding**: four deep-research passes (coach craft, failure-mode taxonomy, company anchors, episode-type patterns) — cited inline with UNCITED markers preserved

---

## Purpose and standard

This doc exists so a prompt engineer can pick up any numbered item below and turn it into a YAML rule tomorrow. It is a craft reference — not a roadmap, not a feature list. The test every item must pass: *would the audio noticeably change if this rule were enforced, in a way a candidate using the product would feel?*

Two disciplines the doc enforces on itself:

- **Cited or UNCITED, never both and never neither.** When a claim comes from a public coach, rubric, or memo, the source is named. When it's synthesized from recurring patterns across mock-session transcripts without a single canonical write-up, it is marked UNCITED so a reader knows to tighten it before shipping copy that cites it as fact.
- **No rubber-stamp of the first-pass package.** §6 lists items the deep-research passes surfaced but should NOT be shipped — the package's gravitational pull toward more ideas is the thing to resist.

---

## 1. What strong human interview coaches actually do

Distilled from [interviewing.io](https://interviewing.io/mocks/faang-behavioral-tech-lead-manager) mock transcripts, [Exponent](https://www.tryexponent.com/practice/product-management-mock-interviews) session structure, ex-Meta hiring-committee coaches ([Evan King](https://www.linkedin.com/posts/evan-king-40072280_behavioral-interview-discussion-with-ex-meta-activity-7292988491792097282-HGwH)), ex-Amazon bar-raisers ([Howard Steinman](https://www.linkedin.com/posts/howardsteinman_inside-the-debrief-the-3-questions-every-activity-7419390463280898048-wIyz)), and Google GCA rubric reverse-engineering ([David Maiolo](https://www.davidmaiolo.com/2021/08/22/google-interview-general-cognitive-ability-gca/), [IGotAnOffer](https://igotanoffer.com/blogs/tech/google-gca-interviews)).

### 1.1 Session spine (the minute budget is the artifact)

Strong coaches declare a time-boxed spine in the opener. Six spines recur:

1. **interviewing.io FAANG spine** — intro + expectations (5-7 min) → main interview (~40-45 min) → thumbs-up/down + per-dimension rubric scores + written comments (last 5 min, hard-stop).
2. **Exponent PM rubric-first spine** — session ends with an auto-transcript + rubric score per attribute (product sense, analytical thinking, leadership & drive) + a "how to improve this score" per dimension.
3. **Bar-raiser spine (Amazon ex-recruiter coaches)** — coach states up front they're answering three internal debrief questions: *will they raise the bar? how do they perform in chaos? would I want to work with them?* Every story told must map back to one.
4. **Meta hiring-committee spine** — first 5 minutes framed as the whole outcome; coach stress-tests the opener and cuts context because the hiring-committee signal is front-loaded.
5. **Google GCA spine** — 45 min, one hypothetical + one behavioral, scored against the explicit 6-item rubric (understanding, clarifying, stakeholders, multiple solutions, justification, hint incorporation). Coach walks all six, not just the answer.
6. **Lewis Lin / StellarPeers PM spine** — clarify → goal → users → prioritized pain → 2-3 solutions → pick one with tradeoffs. Coach gates each step before the candidate can advance.

**Why this matters for generated audio**: episodes today default to a 5-arc narrative spine. They should select a rubric-grounded spine per scenario — a *Recognition* episode on behavioral prep wants spine 3 or 4; a PM product-sense episode wants spine 6.

### 1.2 Coach moves AI content rarely makes

Twelve observed moves, numbered for prompt-YAML referencing:

1. **Name the rubric dimension being graded before the candidate answers.** "This is a Dive Deep question, not an Ownership one." UNCITED as a direct quote; standard in Amazon Bar Raiser coaching.
2. **Refuse "we" answers** — pause and ask "if you were removed from that team, what part would have failed?" ([Hiration](https://www.hiration.com/blog/mock-interview-rubric-feedback-career-advisors-higher-ed/))
3. **Render a deliberately weak answer first** so the candidate hears the failure mode before the retry. (UNCITED; described by Lewis Lin and Jeff H Sipe coaching.) *This is the empirical basis for A1 (weak→strong contrast) in [13-second-pass-critique-and-synthesis.md](./13-second-pass-critique-and-synthesis.md).*
4. **Force STAR reformulation mid-flight.** "Walk me through your specific actions" — stop the story. ([CodeSignal](https://codesignal.com/learn/courses/preparing-and-conducting-effective-interviews/lessons/probing-with-star))
5. **Cut stories over 3-4 years old** and stories completed in a single sprint — recency + scope bias in Amazon debriefs. ([Exponent Amazon LP](https://www.tryexponent.com/blog/amazon-leadership-principles-interview))
6. **Quote the candidate's own words back.** "You said 'we decided' — who is 'we'?" ([Madeline Mann](https://www.linkedin.com/posts/madelinemann_5-secrets-to-never-fail-a-behavioral-interview-activity-7323716033473585156--1w9))
7. **Stopwatch the opener** — 140-160 wpm, 2-second pause before answering. ([Hiration](https://www.hiration.com/blog/mock-interview-rubric-feedback-career-advisors-higher-ed/))
8. **Force an L5-vs-L6 behavioral rewrite** — same story, level-calibrated language. ([Evan King](https://www.linkedin.com/posts/evan-king-40072280_behavioral-interview-discussion-with-ex-meta-activity-7292988491792097282-HGwH))
9. **Declare "soft no / strong yes / re-evaluate" out loud** at mid-session, mirroring written debrief verdicts. ([IGotAnOffer Meta PM](https://igotanoffer.com/blogs/product-manager/facebook-product-manager-interview))
10. **End on insight, not result** — ex-Meta rule: 70% of the story should be specific actions, closing on what the candidate learned. ([Evan King](https://www.linkedin.com/posts/evan-king-40072280_behavioral-interview-discussion-with-ex-meta-activity-7292988491792097282-HGwH))
11. **Time-box answers to 60-90 seconds** so the interviewer can follow up — engagement is the real scoring signal. ([Reno Perry](https://www.linkedin.com/posts/renoperry_7-strategies-to-overcome-rambling-in-interviews-activity-7320066627620720640-8SkW))
12. **Ban comforting praise.** IGotAnOffer and PracHub testimonials repeatedly cite "honest and blunt" as the paid feature; coaches who validate get fired.

### 1.3 Session endings — the last 5 minutes are the product

1. Dimension-by-dimension score recital + 1-sentence "how to improve this number" per dimension. ([Exponent](https://www.tryexponent.com/practice/product-management-mock-interviews), [RocketBlocks](https://www.rocketblocks.me/blog/mock-interview-grading-templates.php))
2. Thumbs-up/down "advance to next round?" verdict spoken aloud — interviewing.io's literal ritual.
3. Same-day written action plan delivered as a deliverable. ([PracHub](https://prachub.com/coaching), [Hello Interview](https://www.hellointerview.com/testimonials), [IGotAnOffer](https://igotanoffer.com))
4. **One story to rewrite before next session** — single assignment, not a list. (UNCITED; recurs in Hello Interview testimonials: "clarity on areas I need to work on.")
5. Hard stop honored to the minute — teaches the candidate that real interviewers do the same.

---

## 2. Failure modes + coach remediations

A prompt engineer should be able to turn each mode into a script-rule that forces the generated episode to either *show* the failure (Recognition / Autopsy types) or *drill past* it (Drill / Pressure types). Full enumeration for behavioral; top-6 for coding / system-design / PM-case to cut load.

### 2.1 Behavioral (10 modes)

| # | Failure | Root cause | Coach remediation |
|---|---------|-----------|------------------|
| B1 | **Wrong-level story** — staff candidate tells IC story | Picked for best result, not right level-signal | "Level filter" drill — candidate names target-level scope signals (multi-team, multi-quarter, ambiguous) and tags each story PASS/FAIL. ([interviewing.io](https://interviewing.io/blog/stop-memorizing-star-for-behavioral-interviews-start-selecting-better-stories)) |
| B2 | **Situation bloat** — 3+ min on setup, no time for Action | Narrative reflex, silence-avoidance | One-sentence Situation drill — rewrite setup to one sentence naming scope + stakes. ([MIT CAPD](https://capd.mit.edu)) |
| B3 | **Team-"we" erasure** | Humility or vague own-role memory | "we-to-I pass" — every "we" converts to "I + concrete verb" or attributes to a named teammate |
| B4 | **STAR-inflation** — "saved millions" with no baseline | Never prepared the metric path | "Evidence audit" — before accepting Result, demand baseline + delta + attribution. ([Aline Lerner, interviewing.io](https://interviewing.io)) |
| B5 | **Blank-out on vague prompts** ("time you failed") | No pre-indexed story bank keyed to archetypes | "5-bucket drill" — conflict / ambiguity / failure / influence / prioritization; pull mapped story in <5s |
| B6 | **Timeline collapse** — multi-quarter project compressed to a paragraph | Story told as gestalt, not beat-by-beat | "Timeline the whiteboard" — 4-6 dated beats with a decision + reason per beat. UNCITED — synthesized from interviewing.io transcripts |
| B7 | **Happy-path fiction** — no tension, no trade-off regret | Selling, not analyzing | Demand a "what went wrong" addendum per story |
| B8 | **Conflict avoidance** — compromise instead of disagreement | Conflict stories feel risky | "Named counterparty" drill — name the role + their position in one sentence before describing your move |
| B9 | **Rehearsed cadence** — same beats every time | Over-practice on scripts, under-practice on variants | Same story 3 different ways ("in 90s" / "leading with impact" / "starting at the conflict") |
| B10 | **Missing the ask-back** — end and stop | Interview-as-monologue mental model | Trained close — "does that answer what you were looking for, or should I go deeper on X?" |

### 2.2 Technical coding (top 6)

| # | Failure | Remediation |
|---|---------|-------------|
| T1 | **Keyboard-before-clarify** (typing in <10s) | 2-minute no-hands rule — no keyboard until restated prompt + input constraints + 2 test cases. ([CodeJeet](https://codejeet.com)) |
| T2 | **Premature optimization** — reaches for segment trees before brute force | State O(n²) baseline with complexity out loud before optimizing. ([Byte by Byte](https://bytebybyte.com)) |
| T3 | **Split-attention collapse** — goes silent 5+ min | Metronome drill — coach taps every 15-20s; if no narration since last tap, pause coding. ([AlgoCademy](https://algocademy.com)) |
| T4 | **Silent complexity** — ships without stating time/space | "Complexity-before-code" gate — commit verbally before first line, re-state at end |
| T5 | **Untested by edge cases** | "Four-case rehearsal" — empty / single / duplicate / max, traced on paper before "done" |
| T6 | **Under-asking on ambiguity** — assumes sorted / non-negative / fits-in-memory | 3-question minimum on input range / output format / constraint edges |

### 2.3 System design (top 6)

| # | Failure | Remediation |
|---|---------|-------------|
| S1 | **Components-before-NFRs** | Ban whiteboard for first 5 min; force 3-5 quantified NFRs. ([Evan King](https://www.linkedin.com/in/evan-king-40072280), [Alex Xu *System Design Interview* Ch. 2](https://bytes.usc.edu)) |
| S2 | **Missing capacity math** | "Label your units" — assumptions + units in a corner box before architecture |
| S3 | **Default-to-SQL reflex** | Name 3 storage options with trade-offs before choosing DB |
| S4 | **Over-indexing on write-path sharding** — never reaches read path | "Four-quadrant timebox" — write path / read path / storage / scale, ~8 min each, visible clock. UNCITED operational norm |
| S5 | **Read-path amnesia** | Mid-interview inject — "a user opens the app; what happens?" — forces end-to-end read trace |
| S6 | **Trade-off-free assertion** | "Because / instead-of" template on every choice |

### 2.4 Case / PM-sense (top 6)

| # | Failure | Remediation |
|---|---------|-------------|
| P1 | **Solution-before-user** | CIRCLES "C + I" gate — Comprehend + Identify customer before any solution. ([Lewis C. Lin](https://lewis-lin.com)) |
| P2 | **Metrics with no objective** | GAME method — Goal first; every metric must map to it. ([IGotAnOffer](https://igotanoffer.com)) |
| P3 | **No North Star metric named** | Pick one + defend it + name the counter-metric that guards against gaming |
| P4 | **Framework-as-script** — linear CIRCLES recitation | Distilled 3 beats (understand / evaluate / recommend); interrupt linear recitation |
| P5 | **No stress-test / counter-argument** | "Red team yourself" — name the strongest objection, answer it. UNCITED synthesis |
| P6 | **Generic prioritization** — "I'd use RICE" without scoring | "Score two, pick one, lose one" — at least two options compared; the loser is named |

---

## 3. Company-specific anchor table

Twelve companies. Template identical across entries so a system-prompt block can key off `target_company_name` and inject the block verbatim without string interpolation inside free text. Anchor blocks replace the current comma-joined company string in [`kitesforu-api/src/api/services/...`](../../../../kitesforu-api) prompts.

> **Integration note**: this should live as structured data (YAML or JSON) in `kitesforu-api/prompts/company_anchors.{yaml,json}` — adjacent to the existing flat company-tailoring block — so the system prompt can reference a structured lookup rather than a comma-joined string.

| # | Company | Loop | Rubric | Culture anchor | Public source | Recent change (confirm) |
|---|---------|------|--------|----------------|---------------|------------------------|
| 1 | **Amazon (SDE + PM)** | SDE2/3: 2 coding + 1 SD + 1 bar-raiser + 1 behavioral | 16 Leadership Principles scored via STAR | Bezos 1997 "Day 1" + [LP page](https://amazon.jobs/content/en/our-workplace/leadership-principles) | amazon.jobs "Interviewing at Amazon" | Bar-raiser increasingly embedded in loop, not scheduled separately. UNCITED specifics |
| 2 | **Google (SWE + PM)** | SWE L4/L5: 2-3 coding + 1 SD (L5+) + 1 Googleyness. PM: 5 rounds across Product Design, Strategy, Analytical, Technical, Leadership | GCA + RRK + Leadership + Googleyness (4-axis) | "How Google Works" + [re:Work](https://rework.withgoogle.com) | google.com/about/careers "How We Hire" | 2024: "Googleyness" reframed toward concrete behaviors (action-taking, pragmatism) per leaked Dec 2024 internal memo |
| 3 | **Meta (SWE + PM)** | E4/E5 SWE: 2 coding + 1-2 SD + 1-2 "Jedi" behavioral. PM: Product Sense / Execution / Leadership & Drive | [Meta Coding Excellence rubric](https://engineering.fb.com) (public since 2023): Problem-Solving / Coding / Verification / Communication | "Move Fast" values + Zuck 2022 "Year of Efficiency" memo | engineering.fb.com "Preparing for your engineering interview" | Blended coding+debugging pairing 2024; E5 product architecture round folded into SD |
| 4 | **Microsoft (SWE + PM)** | SDE2: 3 coding + 1 design (higher levels) + 1 "As-Appropriate" (hiring manager + culture) | Microsoft Leadership Principles — Create Clarity / Generate Energy / Deliver Success + "Model-Coach-Care" | Nadella's *Hit Refresh* | careers.microsoft.com "Interview Tips" | AI-assisted-coding allowed in some 2025 loops. UNCITED pilot scope |
| 5 | **Apple (SWE + Design + PM)** | 4-6 rounds team-dependent. SWE: 2-3 coding + 1-2 domain + 1 behavioral. Design: portfolio + critique + exercise | No public rubric — craft / taste / deep domain mastery. Team variance is the rubric | Jobs 2005 Stanford + Ive's *Objectified* + [apple.com/about values](https://apple.com/about) | interviewing.io Apple guide | Vision Pro / AI teams expanded ML-systems rounds 2024-25. UNCITED specifics |
| 6 | **Netflix (SWE + Content)** | SWE Senior: 4-5 rounds, manager-driven — 1-2 coding + 1 SD + 2+ behavioral with director/VP | "Keeper Test" + Freedom & Responsibility scorecard; binary "would I fight to keep this person?" | [Netflix Culture Memo](https://jobs.netflix.com/culture) + Hastings *No Rules Rules* | jobs.netflix.com | 2024 update: "Dream Team" + "Artistic Expression" added; same F&R core |
| 7 | **Stripe (Backend)** | 5 rounds — 2 coding (integration/debugging-pair + algorithmic) + 1 SD + 1 bug-bash/take-home + 1 behavioral | Operating Principles — Move with Urgency and Focus / Think Rigorously / Trust and Amplify / Optimism / Craft | [stripe.com/jobs "How we work"](https://stripe.com/jobs) + Patrick Collison's reading list | Stripe engineering blog "Interviewing at Stripe" | Debugging-pair round formalized ~2023, now near-universal for backend |
| 8 | **Databricks (Data / ML)** | 5 rounds — 2 coding (Python + SQL/Spark) + 1 ML/data-systems design + 1 domain depth + 1 behavioral | No public letter-grade; Technical Depth / Builder Mindset / Customer Obsession / Truth-Seeking | [Lakehouse whitepaper](https://databricks.com/resources) + Ali Ghodsi Data+AI Summit keynotes | databricks.com/company/careers | Post-MosaicML 2023: GenAI/LLM-systems-design round added for MLE. UNCITED — confirm with candidates |
| 9 | **OpenAI (Research + SWE)** | Research: research discussion + ML coding from scratch + research design + behavioral. SWE: 2 coding + 1 SD + 1 practical ML + 1 behavioral | No public rubric. Observed signals: research taste / speed-of-iteration / "agency" | [OpenAI Charter](https://openai.com/charter) + Altman's "What I Wish Someone Had Told Me" | openai.com/careers | Post-2023 board crisis: increased safety/alignment reasoning in behavioral. UNCITED on formal scoring |
| 10 | **McKinsey (Consultant)** | 2 rounds of 2-3 interviews — case (interviewer-led) + PEI, preceded by Solve assessment | PEI — Personal Impact / Entrepreneurial Drive / Inclusive Leadership / Courageous Change. Cases: Structure / Analytics / Synthesis / Communication | [mckinsey.com Our Values](https://mckinsey.com/about-us/overview/our-purpose-mission-and-values) + Bower *Will to Manage* | mckinsey.com/careers/interviewing | Solve replaced PST 2021; PEI increasingly probed for specificity + candidate contribution vs team |
| 11 | **BCG (Consultant)** | 2 rounds of 2 — case (candidate-led, conversational) + fit/behavioral. Casey chatbot + Online Case as screen | Structuring / Quantitative / Creativity / Business Judgment / Communication (not letter-graded publicly) | BCG PTO program + [Henderson Institute](https://bcghendersoninstitute.com) | careers.bcg.com/interview-prep | Online Case + Casey chatbot rolled out 2021-2023 in most geographies |
| 12 | **Goldman Sachs (IB + Tech)** | IB: HireVue → Superday (4-5 back-to-back 30-min, behavioral + DCF/LBO/accounting). Tech: 3-4 coding + systems + behavioral | 14 Business Principles + 8 Leadership Standards | [Goldman Business Principles](https://goldmansachs.com/our-firm/our-people-and-culture/business-principles) | goldmansachs.com/careers/students/programs | Eng loops added mandatory "low-latency / systems" round for Strats post-2023 |

**Prompt-usage contract**: when `target_company_name` matches an entry, inject the row verbatim into the system prompt inside a labeled block. When it does NOT match, inject the public-format guidance plus an explicit instruction: *"do not invent a rubric; use a strong generic structure."* This is the same fallback clause [api PR #273](https://github.com/vikrantb/kitesforu-api) shipped for curriculum-builder.

---

## 4. Episode-type craft patterns

Five types. Each has: intended job, 5-8-beat spine with minute targets, runtime, speaker format, pattern-rules, anti-pattern, listener-agency moment. Full detail in the appendix script; a tightened summary here.

| Type | Job | Runtime | Format (1st / 2nd) | Distinctive move |
|------|-----|---------|-------------------|------------------|
| **Recognition** | "Hear what a good answer sounds like" | 8-10 min | host + candidate_sim / 2-host | Render A-grade *clean* for 60-90s before any gloss |
| **Drill** | "Practice out loud behind closed doors" | 15-18 min | monologue coach / host+coach | Literal 30s silent timers with signposted start/end |
| **Autopsy** | "Dissect a specific weak answer" | 10-12 min | host+coach / monologue coach | Render weak version in full before ANY diagnosis |
| **Pressure** | "Simulate a hard live round" | 18-22 min | coach-as-interviewer + candidate_sim / 2-host | Every interviewer turn escalates or pivots — never restates. One sim response wobbles and recovers |
| **Final-prep** | "Tighten what I already hold, no new concepts" | 12-15 min | monologue coach | Closing ritual structurally identical every final-prep ep — same 3 sentences, same order |

**Two cross-type rules** that fall out of the agent analysis:

- **Never gloss during the exemplar.** Whether it's an A-grade in Recognition or a weak answer in Autopsy, the script must let the sample play clean for ≥ 60s before commentary. Derived from This American Life's scene-before-gloss discipline.
- **Silence must be signposted.** `[pause 30s]` directives in Drill/Pressure must be preceded by a verbal cue ("30 seconds starts now…") and followed by a verbal reset ("time"). Unsignposted silence reads as a bug.

### 4.1 Autopsy spine — worked example (the highest-signal type)

Because Autopsy is the closest thing the product has to the A1 weak→strong contrast principle, its spine is the one to ship first:

1. **0:00-0:45** — Render the weak answer verbatim: every hedge, every "I guess," every trailing-off. No commentary.
2. **0:45-2:00** — Name 2-3 rubric dimensions that sunk it (ONE sentence each). Cap at 3 — 4+ overflows working memory.
3. **2:00-4:00** — Rewind + replay; coach interrupts at each failure point with a single-sentence diagnosis.
4. **4:00-6:00** — Rebuild: same structure and length, sunk dimensions fixed.
5. **6:00-8:00** — A-B compare ONE paired sentence from weak vs repaired. *This is the money beat.*
6. **8:00-10:00** — Generalize to 3 phrases that signal "I'm about to say something weak." Catch yourself on these.
7. **10:00-end** — Listener-agency: "Pause. Say the 'before' out loud, then the 'after.' The gap between them is the gap between offer and no-offer."

---

## 5. Shippable-tomorrow implementation hooks

Grounded in the existing PromptComposer pipeline (`kitesforu-workers/src/workers/prompts/composer/`) and content-type YAML layer (`kitesforu-workers/src/workers/prompts/content_types/interview/`).

### 5.1 A1 — Weak→strong contrast (prompt rule)

**Shape**: new `contrast_pattern` section in `kitesforu-workers/src/workers/prompts/content_types/interview/narration_rules.yaml` + `dialogue_rules.yaml`. Rule:

> "When introducing a target skill / framing / answer, render the common-weak version for 10-15 seconds, then explicitly transition (*'Here's what changes when we tighten that...'*) to the stronger version. Do this at least once per 8 minutes of runtime. For Autopsy episodes, weak rendering is the entire opening beat — see §4.1."

**Blast radius**: existing PromptComposer — every interview/exam/teacher/corporate episode becomes more teacher-like on next regen. No UI, no schema, no route.

### 5.2 Episode-type picker (frontend + intake)

Currently `/interview-prep/create` takes a free-text prompt + optional company. Add a 5-card picker above the textarea — **Recognition / Drill / Autopsy / Pressure / Final-prep** — each routing into a dedicated generation mode that loads the matching spine from §4.

**Wiring**: a new `episode_type` field on the interview-prep creation request, threaded into the workers composer as a section-builder priority-order override. This is the single "feels like a real coach" move the product can make with existing scaffolding.

### 5.3 Company anchor lookup (api)

Replace the existing comma-joined company-tailoring string in `kitesforu-api/src/api/services/interview_prep/prompts.py` (or wherever the string currently lives) with a YAML/JSON lookup in `kitesforu-api/prompts/company_anchors.yaml`. Keys = `target_company_name`; values = the §3 structured block. Prompt includes the matching block verbatim; unmatched names fall through to the generic-structure clause.

### 5.4 Level-calibrated behavioral rewrite (prompt rule)

Coach move 1.2.8 — "force L5-vs-L6 rewrite." Ship as a Recognition-episode script-rule: when the prompt includes `target_level_role`, the A-grade exemplar must be generated twice — once at the stated level, once at the level-below (or -above) — and the script must name the *linguistic delta* (e.g. "drove cross-org alignment" vs "led my sub-team").

### 5.5 Evidence-audit drill (prompt rule)

Failure mode B4 — STAR-inflation without baseline. Ship as a Drill-episode script-rule: the coach must demand a quantified metric (baseline + delta + attribution) before accepting any Result beat of a story.

---

## 6. What NOT to build (scope discipline)

- **No separate "interview prep sub-app."** The episode-type picker + company anchors + prompt-rule library live inside the existing `/interview-prep` surface. Splitting into a separate app dilutes learnings into two UIs.
- **No private-rubric invention.** When `target_company_name` doesn't match the §3 table, the prompt MUST fall through to generic-structure guidance — never synthesize a fake rubric. This is already the pattern in curriculum-builder; keep it.
- **No more than 3 failure dimensions per Autopsy episode.** §4.1 Beat 2 caps at 3; four or more is forgettable.
- **No "teach something new" in Final-prep.** §4 rule for Final-prep is strict: only re-tighten concepts already in the user's prior episodes. Label drift kills retrieval.
- **No A/B test framework for episode-type picker.** Analytics + feature flag is sufficient for rollout decisions; avoid infra sprawl.
- **No re-doing the scenario taxonomy.** Interview-prep is already a top-level scenario. This doc doesn't add new scenarios.

---

## 7. Sequencing for the next 2 weeks of shipping

1. **A1 — weak→strong contrast YAML rule** (workers PR). ~1 day. Enables Autopsy as a content shape before Autopsy exists as a picker option.
2. **A3 — honest scope-of-confidence hedging YAML rule** (workers PR, adjacent file). ~½ day.
3. **Episode-type picker** (frontend + intake + workers section-builder priority override). ~2-3 days. Ships Autopsy first (highest signal), then Drill and Pressure.
4. **Company anchor lookup** (api PR). ~1 day. Replaces the flat company string with the §3 structured table.
5. **Recognition + Final-prep** picker options. ~1 day (lower distinctiveness than Autopsy/Drill/Pressure but rounds out the set).
6. **Evidence-audit + level-calibration drills** as standing prompt rules. ~1 day.

Total ≈ 7-9 engineering days. No new schemas beyond `episode_type` on the interview-prep creation request. No new services. No new routes beyond the picker.

---

## 8. Sources

**Coach craft**: interviewing.io mock transcripts; Exponent PM practice; Lewis C. Lin (Decode & Conquer, StellarPeers); Howard Steinman (ex-Amazon Bar Raiser); Evan King (ex-Meta HC); David Maiolo GCA reverse-engineering; IGotAnOffer Meta PM + Bar Raiser guides; Hiration mock-interview rubric/feedback; CodeSignal STAR probing; Madeline Mann; Reno Perry.

**Failure-mode taxonomy**: interviewing.io blog (Aline Lerner); High Growth Engineer newsletter; Eng Leadership newsletter; MIT CAPD STAR guide; Alex Xu *System Design Interview* Ch. 1-2; Evan King NFR checklist; Medium/Marcus Davis "Common Mistakes in System Design"; GeeksforGeeks; Byte-by-Byte; AlgoCademy; CodeJeet; HackerNoon (Mundada); Lewis Lin CIRCLES; Exponent "Less Linear CIRCLES"; IGotAnOffer GAME method; productmanagementexercises.com.

**Company anchors**: amazon.jobs LP page; Bezos 1997 letter; Meta Coding Excellence rubric (public 2023); Netflix Culture Memo; Stripe careers "How we work"; Ghodsi Data+AI keynotes; OpenAI Charter; McKinsey Our Values; BCG Henderson Institute; Goldman Business Principles.

**Episode-type craft**: This American Life's scene-before-gloss discipline (Ira Glass interviews); Transom.org on signposted silence; Exponent mock sessions; Pragmatic Engineer mock rounds; general rehearsal literature on label-stability for near-term retrieval.

UNCITED markers are preserved inline wherever a claim was synthesized from recurring patterns without a single canonical public write-up. Tighten before publishing any generated copy that references these as fact.
