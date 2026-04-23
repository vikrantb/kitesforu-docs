# 12 — Exam-Prep Deep-Dive (content-craft)

**Status**: PROPOSED — grounding reference for prompt-engineering work
**Scope**: exam-prep scenario only (priority #2 in content deep-dive stack; sibling to [11-interview-prep-deep-dive](./11-interview-prep-deep-dive.md))
**Written**: 2026-04-22
**Grounding**: four deep-research passes — cognitive science of exam prep, per-exam trap-pattern taxonomy, per-exam structural anchors, exam-distinctive episode-type craft — cited inline with UNCITED preserved

---

## Purpose

Same standard as the interview-prep sibling: every numbered item must pass *"could a prompt engineer turn this into a YAML rule tomorrow?"*. Exam-prep shares the 5-episode-type frame from interview-prep (Recognition / Drill / Autopsy / Pressure / Final-prep) but *supersedes* three of them with domain-distinctive variants (Miss Autopsy / Timed Section Rehearsal; see §4).

The thesis: exam-prep content should be louder about **retrieval** and **desirable difficulty** than any other scenario the product supports. That's the wedge — AI audio naturally drifts toward fluent narration; great exam-prep audio fights that drift.

---

## 1. Cognitive science of exam prep, as script directives

Distilled from Karpicke & Blunt 2011, Roediger & Butler 2011, Dunlosky 2013 review, Cepeda et al. 2006 meta-analysis, Bjork's desirable difficulties, Dunlosky & Rawson 2012 on metacognition.

### 1.1 Retrieval practice — the testing effect

1. **Say-it-out-loud retrieval.** "Say out loud, right now, which formula applies. Five seconds." SSML pause + reveal. Spoken commitment is the retrieval event. *(Karpicke & Blunt 2011.)*
2. **Distractor pre-generation.** "Name three wrong answers this question type usually baits you into, before I tell you the right one." Retrieves the error schema, not just the fact.
3. **Pre-question retrieval, not recap.** Open every concept segment with a 10-15s quiz on the *previous* concept. Recaps are restudy; questions are retrieval. *(Roediger & Butler 2011.)*
4. **Generate-then-compare.** "Pause the episode. Work the problem. Unpause when you have an answer." Audio waits via long pause + bell cue.
5. **Free-recall at segment close.** "Say back the two things you'd tell a friend about this concept." Free recall outperforms cued recall for transfer.
6. **Elaborative interrogation.** "Why does this formula have an n-1 and not an n? Answer out loud." Rated highly effective. *(Dunlosky 2013.)*
7. **Application retrieval, not definitional.** Never "what is X?" — always "given these facts, which ratio flags the distortion?". Definitional prompts permit recognition; application requires retrieval.

### 1.2 Spacing for linear audio

1. **Intra-episode callback.** Every concept introduced in minutes 0-5 gets a retrieval callback at minutes 15-25. Question, not recap.
2. **Inter-episode recycling.** Each episode's cold-open pulls from episodes the user completed 1, 3, and 7 days ago — expanding-gap, fixed slots. *(Cepeda et al. 2006 — optimal gap ≈ 10-20% of retention interval.)*
3. **Final-prep sweep 2-3 weeks before test date.** Touch every concept in descending-gap order (oldest first).
4. **End-of-episode mini-quiz recycles LAST WEEK's concept.** Not this episode's — that's the "I just heard it" recognition illusion.
5. **Interleaved callback across domains.** In a GMAT quant episode, drop one verbal retrieval prompt. Cross-domain spacing prevents context-bound encoding.

### 1.3 Desirable difficulties (Bjork)

1. **Delayed answer.** After a hard question, 20-30s of worked reasoning aloud before reveal. Don't fill the silence with "so let's think about this" — let ambiguity sit.
2. **Reverse-ordered exemplars.** Lead with the hardest case; follow with the clean textbook case. Forces schema construction before pattern-matching.
3. **Interleaved question types within a segment.** Rate/work, then mixture, then rate/work. Blocking inflates in-session performance, deflates transfer.
4. **Contrastive pairs.** Two superficially similar problems with different correct approaches back-to-back. "This looks like 1.3.3 but isn't — why?"
5. **Production before exposition.** Ask the user to predict the rule before stating it. Even wrong predictions improve encoding (pretesting effect).

### 1.4 Metacognition + error-driven learning

1. **Pre-answer confidence rating 1-5.** Then reveal. The gap between confidence and correctness is the learning signal. *(Dunlosky & Rawson 2012.)*
2. **Error-dimension self-naming.** After a miss: "Name which step you got wrong before I diagnose it — setup, formula, or arithmetic?" Self-diagnosis > tutor diagnosis.
3. **Postdiction.** "How many of the last five did you get right? Answer, then I'll tell you." Postdictions are better-calibrated than predictions.
4. **Explicit miscalibration callout.** High confidence + wrong answer: script names it. *"You said 4-of-5 — this is exactly the item type you're overconfident on."*
5. **"Teach it back" close.** "Explain this to someone who's never seen it, in 20 seconds." Forces the illusion-of-knowing to collapse.

### 1.5 What does NOT work (explicit forbid list for prompt rules)

1. **Re-reading / re-listening.** Among the least effective techniques tested. *(Dunlosky 2013.)* Do not script "let's listen to that again" segments — replace with a retrieval question.
2. **Emphasis cues** ("remember this part"). Low utility; no active processing demanded.
3. **Massed practice blocks.** Twenty rate-problems in a row feels productive, transfers poorly. Always interleave.
4. **Summary-heavy recaps.** Restating = restudy. If a recap exists, convert it to a question stack.
5. **Fluency-mimicking delivery.** Smooth, confident, uninterrupted narration produces a "feeling of knowing" without the knowing. Script-level disfluencies (pauses, self-corrections, "wait, actually—") are a feature, not a defect.

---

## 2. Trap-pattern taxonomy — top 4 per exam family

Full agent output produced 42 traps across 7 families. Shipping the top 3-4 per family by cognitive-hook distinctiveness — the rest are tail cases the generation prompt can cover opportunistically. Format: **Name** — cognitive hook → remediation. *(Source.)*

### 2.1 GRE Verbal *(ETS Official Guide)*

- **G1.1 Trap-word-of-the-day** — surface-match heuristic; test-takers lock onto register-echo vocabulary and assume familiarity = fit → require blank-meaning to be inferred from the sentence pivot before scanning options.
- **G1.2 Contextual-synonym shift (SE)** — pair produces same-meaning sentence, not same-dictionary word → substitute each pair and verify full-sentence equivalence.
- **G1.4 RC inference scope trap** — distractors overgeneralize a passage-local claim → pick the most minimal inference strictly licensed by the text.
- **G1.6 Primary-purpose zoom trap** — distractors describe a paragraph's role, not the whole passage → match scope to full passage.

### 2.2 GMAT Quant *(GMAC Official Guide)*

- **M2.1 Specific-looking statement-2 (DS)** — concrete numbers feel sufficient; may fix one variable when two are needed → restate the question algebraically; check if each statement pins the exact quantity asked.
- **M2.3 Pretty-answer bias** — aesthetic preference for clean integers; GMAT deliberately places elegant decoys → treat clean numbers as a flag to re-derive.
- **M2.4 C-trap (both statements illusion)** — when each alone gives a value, C is tempting; actually each alone is sufficient → answer D → re-test each statement in isolation after combining.
- **M2.5 Integer-constraint omission** — ignoring "positive integer," "distinct," "non-zero" qualifiers → underline all constraint words before solving.

### 2.3 MCAT (CARS + Science) *(AAMC Official Guide)*

- **C3.1 Primary-purpose vs main-idea split** — distractors swap "what the passage says" with "why the author wrote it" → for primary-purpose, pick an infinitive verb ("to argue") and verify against the whole arc.
- **C3.2 Attitude/tone undercurrent** — overt thesis and author's subtle stance diverge → flag hedges ("ostensibly," "purports") as attitude markers.
- **C3.3 Application/transfer trap** — question introduces an outside scenario; candidates re-skim for keyword matches → restate the passage's rule in one sentence, then test.
- **C3.5 Experimental-design IV/DV confusion (science)** — distractors describe a measured variable as if controlled → label each variable (IV/DV/controlled) before answering.

### 2.4 CFA I-III *(CFA Institute Curriculum)*

- **F4.1 Soft-violation ethics case** — collegial behavior (sharing research draft with favored client) violates Fair Dealing (III(B)) → map every case action to a specific Standard before picking "no violation."
- **F4.2 Dual-obligation conflict** — candidates pick the "more obvious" duty, missing that Standards require both met via disclosure → default to disclosure + equitable treatment when duties collide.
- **F4.4 Capitalize-vs-expense classification** — IFRS/US-GAAP treatment difference changes ratios candidates then compute on the wrong base → confirm reporting standard first.
- **F4.6 Level-III behavioral bias mislabel** — closely-related biases (representativeness vs availability; loss aversion vs regret aversion) → match the decision mechanism, not the emotional vocabulary.

### 2.5 Bar exam — UBE *(NCBE Subject Outlines)*

- **B5.1 Red-herring fact pattern** — facts cue a high-salience rule (Miranda) when the dispositive issue is adjacent (custody determination) → issue-spot by element checklist, not by keyword.
- **B5.2 Hearsay-exception masquerade** — looks like present-sense impression but fails contemporaneity; FRE 801(d) non-hearsay is a different category → apply FRE 803 elements individually.
- **B5.3 Conflicts-of-law default** — candidates apply forum law by default; MEE tests when Second Restatement's "most significant relationship" displaces it → state the choice-of-law rule before applying substantive rules.
- **B5.6 Con-law standing vs merits** — candidates argue constitutional merits without clearing injury/causation/redressability → standing analysis always precedes merits. *(Lujan v. Defenders of Wildlife.)*

### 2.6 AP *(College Board CED)*

- **A6.1 AB Calculus sign-tracking in optimization** — students find critical points, drop sign analysis of f' → first-derivative sign chart or second-derivative test is mandatory.
- **A6.3 English Lang "sophistication" point misread** — rubric rewards complex argument moves (qualification, counterargument synthesis), not vocabulary → sophistication is structural, not lexical. *(AP English Language Scoring Guidelines, Row C.)*
- **A6.4 Evidence-commentary imbalance (English Lang)** — quote-stacking caps Row B → every piece of evidence needs an interpretive sentence linking to thesis.
- **A6.5 AP Bio confounding-variable blind spot** — FRQ embeds an uncontrolled variable; students propose a test that can't isolate cause → list all plausible confounds; specify what's held constant.

### 2.7 Tech certifications (AWS / Azure / GCP)

- **T7.1 Works-but-not-recommended** — all four answers solve the problem; only one aligns with the Well-Architected Framework → filter by best-practice guidance, not mere feasibility.
- **T7.2 Cost-vs-availability priority mismatch** — stem specifies "cost-optimized"; candidate defaults to multi-AZ/HA out of habit → extract the optimization pillar from the stem ("cost," "operational," "resilient") and filter.
- **T7.3 Overkill-service trap** — picking Aurora / Cosmos / Spanner when managed single-region meets stated requirements → match service to stated RPO/RTO/scale, not to the premium tier.
- **T7.6 Managed-vs-self-managed miss** — self-managed on EC2/VM feels flexible but violates Operational Excellence → if a managed service covers requirements, pick it unless stem specifies custom control-plane needs.

---

## 3. Per-exam structural anchors

Identical template across exams; injected verbatim into the system prompt when the user names a target exam. Template: format+duration / score+threshold / sections+pace / recent changes (dated) / official-prep anchor / scoring psychology / common mis-paths.

### 3.1 GRE (General Test)

- **Format**: CBT, section-level adaptive within Verbal/Quant; ~1h 58min. 1 AWA + 2 Verbal + 2 Quant.
- **Score**: Verbal 130-170, Quant 130-170, AWA 0-6. Competitive CS PhD: Q ≥ 167 UNCITED; top humanities V ≥ 163 UNCITED.
- **Pace**: Verbal 27q/41min (~91s); Quant 27q/47min (~104s); AWA 1 essay/30min.
- **Recent change**: **Shortened GRE launched September 22, 2023** — Argument essay removed, unscored experimental removed.
- **Anchor**: ETS Official GRE Super Power Pack / POWERPREP.
- **Scoring psych**: Section-level adaptive — Section 1 performance caps Section 2 ceiling.
- **Mis-paths**: (a) memorizing vocab without reading strategy for Text Completion context; (b) skipping AWA practice; (c) assuming Quant = heavy calculator use (it's on-screen only).

### 3.2 GMAT Focus Edition

- **Format**: CBT, 2h 15min — three 45-min sections.
- **Score**: Total **205-805** (new scale). Section 60-90. M7 median ~735+ UNCITED; top-10 competitive ≥ 705 UNCITED.
- **Pace**: Quant 21q/45min (~129s); Verbal 23q/45min (~117s); Data Insights 20q/45min (~135s).
- **Recent change**: **GMAT Focus launched November 7, 2023**; legacy GMAT retired Feb 2024. AWA + Sentence Correction removed; Data Insights replaced Integrated Reasoning.
- **Anchor**: Official Guide for GMAT Focus + 6 Official Practice Exams on mba.com.
- **Scoring psych**: Review/edit allowed (up to 3 flags/section) — trade-off between confidence and revision UNCITED.
- **Mis-paths**: (a) over-indexing on Quant despite equal weighting; (b) under-prepping Data Insights as "new"; (c) burning flags on early questions.

### 3.3 MCAT

- **Format**: CBT, ~7h 30min seated (6h 15min testing). Not adaptive.
- **Score**: Each section 118-132; total **472-528**. Matriculant median ~512 UNCITED; top-20 competitive ≥ 518 UNCITED.
- **Pace**: Four sections, each 59q (CARS is 53q) / 90-95 min.
- **Recent change**: No structural change since **2015 overhaul** that added Psych/Soc and replaced Verbal Reasoning with CARS.
- **Anchor**: AAMC Official MCAT Prep Bundle (Question Packs, Section Banks, FL1-FL5).
- **Scoring psych**: All four sections equal-weighted — no "safe" section to under-study.
- **Mis-paths**: (a) not practicing under timed conditions until M-1 month; (b) treating CARS as unstudyable; (c) over-studying content, under-practicing AAMC-specific question logic.

### 3.4 CFA Levels I-III

- **Format**: CBT, each level ~4h 30min (two sessions).
- **Score**: Pass/fail per level; MPS standard-setting per sitting. Level I pass ~35-45% UNCITED.
- **Sections**: L-I 180 MCQ across 10 topics; L-II item-set vignettes; L-III essay + item-set + **Pathway choice (Portfolio Mgmt / Private Markets / Private Wealth)**.
- **Recent change**: **Level III Pathways introduced 2025**; **Practical Skills Module (PSM) now required at all three levels** (L-III PSM added 2025); enrollment fee eliminated April 29, 2025.
- **Anchor**: CFA Institute Learning Ecosystem (curriculum + mock exams).
- **Scoring psych**: Ethics is disproportionate — the "ethics adjustment" shifts borderline candidates UNCITED.
- **Mis-paths**: (a) skipping Ethics until late; (b) memorizing formulas without L-II item-set interpretation practice; (c) treating L-III essay like MCQ.

### 3.5 Bar exam — UBE (current format)

- **Format**: Two-day, ~12h total (6h/day).
- **Score**: 400-point scale; jurisdictional passing threshold (typically 260-280; NY = 266).
- **Weighting**: **MBE 50% / MEE 30% / MPT 20%**. MBE 200q over 6h; MEE 6 essays/3h; MPT 2 performance tasks/3h.
- **Recent change**: **NextGen UBE launches July 2026** — MBE/MEE/MPT retire after Feb 2028; NextGen uses integrated question sets + Legal Research PT; 9h unified test day; phased rollout through July 2028.
- **Anchor**: NCBE Study Aids + MBE Online Practice Exams. NextGen: Official Examinees' Guide.
- **Scoring psych**: MBE dominates (50% weight; essays scaled to MBE) — weak MBE cannot be rescued by essays UNCITED.
- **Mis-paths**: (a) treating MEE as memorization when IRAC structure matters more; (b) skipping timed MPT practice; (c) studying all MBE subjects equally instead of weighting by question density.

### 3.6 AP (family)

- **Format**: Most AP exams 2-3h; Section I MCQ + Section II free-response.
- **Score**: 1-5 scale. Many selective colleges grant credit at 4+; Ivy-selective often require 5 UNCITED.
- **Representative pace**: AP Calc BC 3h 15min (45 MCQ/1h 45min + 6 FRQ/1h 30min); APUSH 3h 15min (55 MCQ + 3 SAQ / 1 DBQ + 1 LEQ); AP English Lang 3h 15min (45 MCQ/1h + 3 essays/2h 15min).
- **Recent change**: **AP Precalculus launched Fall 2023**; AP Physics 1/2/C frameworks revised Fall 2024; AP CS A framework revised Fall 2025.
- **Anchor**: AP Classroom (College Board) + AP CED PDFs.
- **Scoring psych**: MCQ/FRQ weighted ~50/50; no guessing penalty since 2011 UNCITED — blank answers strictly dominated.
- **Mis-paths**: (a) ignoring FRQ rubrics; (b) no practice under reading+writing time constraints; (c) studying for high-school final grade, not scoring conventions.

### 3.7 USMLE Step 1 / Step 2 CK

- **Format**: Step 1 ~8h (7 blocks of ~40 MCQ, ~280 items) UNCITED; Step 2 CK ~9h (8 blocks, ~318 items) UNCITED. Prometric.
- **Score**: **Step 1 = pass/fail only since January 26, 2022.** Step 2 CK = 3-digit scaled; passing 214 UNCITED; competitive for top residencies ≥ 255 UNCITED.
- **Recent change**: Step 1 P/F transition Jan 26, 2022; Step 2 CK added Interactive Testing Experience UNCITED.
- **Anchor**: USMLE official + NBME self-assessments; de facto: First Aid for USMLE Step 1 + UWorld UNCITED.
- **Scoring psych**: Post-P/F, residency weight shifted to Step 2 CK — score now the primary quantitative residency signal UNCITED.
- **Mis-paths**: (a) under-prepping Step 1 and failing; (b) Anki without UWorld clinical reasoning; (c) scheduling Step 2 CK too late for ERAS.

### 3.8 AWS / Azure / GCP tech certs

- **Format**: Vendor-proctored. AWS Associate 130 min / 65q; AWS Pro 180 min / 75q. Azure Fundamentals 45 min; Associate/Expert 100-120 min. GCP Pro 2h / 50-60q.
- **Pass thresholds**: AWS Foundational 700, Associate 720, Pro/Specialty 750 (100-1000 scaled); Azure 700 (1-1000 scaled) UNCITED; GCP pass/fail only, threshold not published UNCITED.
- **Top 3 per vendor**: AWS — Cloud Practitioner (CLF-C02), SAA-C03, SAP-C02. Azure — AZ-900, AZ-104, AI-102. GCP — Associate Cloud Engineer, Professional Cloud Architect, Professional Data Engineer.
- **Recent change**: AWS SAA-C03 (Aug 2022); AI-102 revised for generative AI (2024) UNCITED; GCP added Generative AI Leader cert (2024-2025) UNCITED.
- **Anchor**: AWS Skill Builder + Official Practice Sets; Microsoft Learn Practice Assessments; Google Cloud Skills Boost + Official Study Guide.
- **Scoring psych**: Unequal domain weighting (AWS SAA: Secure 30% / Resilient 26% / Performant 24% / Cost 20%) punishes ignoring any domain.
- **Mis-paths**: (a) video courses without labs; (b) memorizing services without trade-off selection; (c) dump sites instead of official practice.

### 3.9 SAT / ACT (admissions)

- **SAT format**: Digital, adaptive, ~2h 14min. R&W two 32-min modules (27q ea); Math two 35-min modules (22q ea).
- **SAT score**: 400-1600. Ivy/top-20 competitive ≥ 1500 UNCITED.
- **ACT format (post-2025 Enhanced)**: ~2h 5min core (English/Math/Reading); Science optional; essay optional.
- **ACT score**: Composite 1-36. Top-20 competitive ≥ 34 UNCITED.
- **Recent change**: **Digital SAT rolled out March 2024 US**; **Enhanced ACT launched Spring 2025**.
- **Anchor**: College Board **Bluebook** app + Khan Academy Official Digital SAT Course.
- **Scoring psych**: SAT routing — Module 1 determines Module 2 ceiling; early accuracy has multiplicative effect on max score.
- **Mis-paths**: (a) practicing paper SAT after digital launched; (b) over-studying ACT Science post-optional; (c) ignoring built-in Desmos/annotate in Bluebook.

---

## 4. Episode-type craft — what's exam-distinctive

The interview-prep Recognition / Drill / Final-prep types port as-is (domain-neutral first-exposure, drill, last-24h consolidation). The remaining three are **superseded** by exam-distinctive variants:

| Interview-prep type | Exam-prep replacement | Why |
|---|---|---|
| Autopsy | **E3 Miss Autopsy** | Exams have discrete right-answer + timestamped decision trees — tighter forensic unit |
| Pressure | **E5 Timed Section Rehearsal** | Exams need full-section endurance simulation, not one-hard-question pressure |
| (n/a) | **E1 Retrieval** | Pure retrieval-practice episodes have no interview analogue |
| (n/a) | **E2 Distractor Hunt** | Teaching trap-pattern mechanisms before answer-teaching is exam-specific |
| (n/a) | **E4 Spaced Review** | Spacing is critical for exam retention over weeks; interview prep is sprint-timed |

### E1 Retrieval episode (12 min, solo host)

Spine — E1.1 cold-open list of 8 target items, no "we'll learn about" / E1.2 calibration warm-up (1 gimme) / E1.3 Retrieval prompt 1: "pause, say answer, rate confidence 1-5" / E1.4 reveal + one-line why / E1.5 loop for items 2-7 / E1.6 rapid gauntlet (8s/item, no pause prompts) / E1.7 item 8 = bridge to tomorrow's Spaced Review.

- **Rule**: every prompt MUST include "say it out loud" + "rate 1-5 before reveal." Confidence-before-answer is the desirable difficulty.
- **Rule**: reveal-to-next-prompt ≤ 30s. Longer gaps let the learner rationalize misses as hits.
- **Anti-pattern**: host explains the concept *before* the retrieval prompt — converts retrieval into recognition.
- **Agency moment**: after gauntlet, "Which item do you want promoted to tomorrow's spaced review?"

### E2 Distractor Hunt episode (18 min, two-host dialogue)

Spine — E2.1 scene-before-gloss cold open (student falling for trap) / E2.2 name the trap family with one-sentence mechanism / E2.3-E2.5 three worked examples (host dissects why the trap option is *gravitationally attractive* BEFORE touching the correct answer) / E2.6 contrast pair (trap fires vs doesn't) / E2.7 3-item field checklist / E2.8 bridge to Miss Autopsy.

- **Rule**: correct answer NOT revealed until the trap mechanism is fully explained. Front-load the wrong answer's seduction.
- **Rule**: three worked examples share the trap skeleton, NOT the surface topic. Three biology stems teach biology; one bio + one history + one math teach the *trap*.
- **Anti-pattern**: generic "watch out for tricky answers" without a named trap family.

### E3 Miss Autopsy (10 min, solo host + learner's timestamped answer audio)

Spine — E3.1 replay stem verbatim, no commentary / E3.2 replay learner's chosen answer + elapsed time + confidence rating (data-forward) / E3.3 reconstruct decision tree with timestamps ("at second 12 you eliminated C; at 28 you were deciding A vs D") / E3.4 name the divergence point + trap family (links to E2 taxonomy) / E3.5 counterfactual correct-reasoning chain at the same pace / E3.6 pattern check across last N misses (Error-Log collapsed here) / E3.7 prescription (one Retrieval + one Distractor Hunt queued).

- **Rule**: beat E3.3 timestamped to the learner's actual timing. "At second 28" beats "when you were deciding."
- **Rule**: prescription must queue concrete next-episode items. Autopsy without follow-up is theater.
- **Anti-pattern**: autopsying a question the learner got right-but-slow. That's a Timed Section Rehearsal item.

### E4 Spaced Review (15 min, solo host)

Spine — E4.1 frame ("12 items from last 5 episodes — 4 from yesterday, 4 from 3 days ago, 4 from last week") / E4.2 1-day items (4 × 45s) / E4.3 3-day items (4 × 60s with micro-re-teach) / E4.4 7-day items (4 × 70s with full re-teach if missed) / E4.5 miss-promotion (re-queue at 1-day) / E4.6 hit-demotion (confidence-5 hits extended to 14-day) / E4.7 preview tomorrow's recurrences.

- **Rule**: items MUST re-use exact phrasing from origin episode. Paraphrasing changes the retrieval cue.
- **Rule**: gap structure computed from the user's actual episode history, not a fixed 1/3/7 template.
- **Anti-pattern**: cramming 20+ items to "cover more." Spaced rep works on gap-spacing, not volume. 12 is the ceiling.

### E5 Timed Section Rehearsal (30-35 min, solo host with silence as co-presence)

Spine — E5.1 section frame with pace target / E5.2 start bell (host silent) / E5.3 pace cues at 5/10/15/20/25 min (≤ 8s each, ZERO teaching content — "7 minutes in, you should be on question 4-5") / E5.4 2-minute warning (tone shift, tighter) / E5.5 stop bell / E5.6 post-section debrief (pace cues beaten vs lagged) / E5.7 flag 1-2 questions for tomorrow's Miss Autopsy based on time-vs-outcome.

- **Rule**: pace cues ≤ 8s, zero teaching content. Teaching mid-section trains the wrong habit.
- **Rule**: silence between cues is real silence (+ room tone), NOT music. Music primes mood; silence primes the exam.
- **Anti-pattern**: host narrating learner's progress ("you're doing great, 5 to go!"). Real proctors don't do this.
- **Agency moment**: pre-E5.1 user picks pace mode — stretch (75s/q) / exam (90s/q) / speed (60s/q).

---

## 5. Shippable-tomorrow implementation hooks

Grounded in `kitesforu-workers/src/workers/prompts/` structure.

### 5.1 New content-type branch: `content_types/exam_prep/`

Today the content-type folder has `interview/`, `educational/`, etc. Exam prep is currently routed through `educational/`. It needs its own branch to carry the retrieval + desirable-difficulty YAML rules from §1 without leaking them into bedtime / storytelling / lecture content.

**Files**:
- `content_types/exam_prep/narration_rules.yaml` — retrieval/spacing/difficulty rules from §1 embedded as PromptComposer section builders at priority 120.
- `content_types/exam_prep/dialogue_rules.yaml` — same for two-host Distractor Hunt format.
- `content_types/exam_prep/episode_type_spines/*.yaml` — one file per §4 type (E1-E5) with the spine's beats as structured prompt fragments.

### 5.2 Per-exam structural anchor lookup

`kitesforu-api/prompts/exam_anchors.yaml` — keyed by exam-id (gre / gmat-focus / mcat / cfa-l1 / cfa-l2 / cfa-l3 / ube / ap-{calc-bc,apush,english-lang,bio} / usmle-step1 / usmle-step2ck / aws-{clf,saa,sap} / azure-{az900,az104,ai102} / gcp-{ace,pca,pde} / sat-digital / act-enhanced), values = §3 structured block. Replaces any currently inline "which exam are they prepping for?" prompt scaffolding.

### 5.3 Trap-pattern library

`kitesforu-api/prompts/exam_traps.yaml` — keyed by exam-id + trap-id (G1.1, M2.3, etc.), values = cognitive hook + remediation from §2. Used by the E2 Distractor Hunt episode-type composer to select a named trap as the episode focus instead of fabricating a generic "tricky question" example.

### 5.4 Retrieval-practice script section builder

New `prompts/composer_sections/retrieval_prompt.yaml` — emits the "pause, say out loud, rate confidence 1-5, reveal" block. Activates when `content_purpose == 'exam_prep'` AND `episode_type ∈ {retrieval, spaced_review, timed_rehearsal}`. Ships the §1.1 + §1.4 rules directly.

### 5.5 Pace-cue + silence directive

New `prompts/composer_sections/pace_cues.yaml` — emits `[pause Ns]` blocks with verbal cue templates at exact minute marks (5/10/15/20/25). Activates ONLY when `episode_type == 'timed_section_rehearsal'`. Leverages the existing `transition_resolver.py` pause directive honored by the TTS orchestrator.

### 5.6 Error-dimension self-naming rule

Add to `content_types/exam_prep/narration_rules.yaml` section §1.4.2: after any wrong-answer reveal in Drill / Retrieval / Miss Autopsy episodes, script must say "before I diagnose, name which step you got wrong — setup, formula, or arithmetic?" with a 5-sec pause. Single one-line YAML rule.

---

## 6. What NOT to build

- **No "exam prep sub-app."** Like the interview-prep wedge, the episode-type picker + exam-anchor lookup + trap-pattern library live inside the existing `/exam-prep` surface. No new routes.
- **No invented trap names.** When `target_exam` doesn't match the §2 table, the prompt MUST fall through to generic cognitive-science rules from §1 — never invent a trap family.
- **No music under E5 Timed Section Rehearsal.** The §4.E5 rule is load-bearing; music destroys exam-environment training.
- **No re-reading / recap segments.** §1.5.1 forbids. Convert any recap to a retrieval question stack.
- **No trap-list longer than 4 per exam in the audio itself.** Longer lists overflow working memory — the §2 library is a prompt-side lookup, not a recitation of every trap.
- **No interleaving the E2 Distractor Hunt across trap families within one episode.** One named trap family per episode — cross-family interleaving dilutes the pattern-extraction signal.
- **No static 1/3/7 spacing for E4.** §4.E4 rule: gap structure must be user-specific, computed from episode history. Static spacing is worse than none.

---

## 7. Sequencing

7-9 engineering days, shippable in parallel with the interview-prep chain (§5 of [11-interview-prep-deep-dive](./11-interview-prep-deep-dive.md)) because they touch adjacent files with no shared edits.

1. `content_types/exam_prep/` branch + narration_rules + dialogue_rules. ~1.5 days.
2. `exam_anchors.yaml` + `exam_traps.yaml` on api side. ~1 day.
3. `retrieval_prompt.yaml` section builder. ~½ day.
4. `pace_cues.yaml` section builder. ~½ day.
5. Episode-type picker on `/exam-prep/create` wiring E1/E2/E3/E4/E5 spines into PromptComposer priority-order overrides. ~2-3 days.
6. E3 Miss Autopsy requires capturing the learner's answer audio + confidence rating — needs a small schema addition to the exam-prep submission payload. ~1 day.

Total ≈ 7-9 eng days. No new services. No new routes. One tiny schema addition for answer capture on E3.

---

## 8. Sources

**Cognitive science**: Karpicke & Blunt, *Science* 2011; Roediger & Butler, *Trends in Cognitive Sciences* 2011; Dunlosky, Rawson, Marsh, Nathan & Willingham, *Psychological Science in the Public Interest* 2013; Cepeda, Pashler, Vul, Wixted & Rohrer, *Psychological Bulletin* 2006; Bjork's desirable difficulties (various); Dunlosky & Rawson 2012 on metacognition.

**Trap taxonomy**: ETS GRE Official Guide; GMAC Official Guide for GMAT Focus; AAMC Official Guide to the MCAT; CFA Institute curriculum (Code & Standards, FRA, Fixed Income, Behavioral Finance readings); NCBE MBE/MEE/MPT outlines + FRE + Restatement (Third) of Torts §14; *Lujan v. Defenders of Wildlife*; College Board AP CEDs (Calc, English Lang, Bio); AWS Well-Architected + SAA-C03 Exam Guide; Azure/GCP Well-Architected/Architecture Framework equivalents.

**Structural anchors**: ets.org, mba.com, aamc.org, cfainstitute.org, ncbex.org, collegeboard.org, usmle.org, aws.amazon.com/certification, learn.microsoft.com/certifications, cloud.google.com/certification, act.org.

**Episode-type craft**: Karpicke & Blunt 2011 (testing effect); Roediger & Butler 2011 (retrieval > restudy); Cepeda 2006 (optimal spacing ≈ 10-20% of retention interval); Bjork desirable difficulties; This American Life's scene-before-gloss discipline; Transom.org on silence-as-content.

UNCITED markers preserved inline wherever a claim was not backed by a single canonical public source (e.g. "Q ≥ 167 competitive for top CS PhDs", "AWS Azure scaled-scoring 700 threshold", "Anki without UWorld" community knowledge). Tighten before any generated copy references these as fact.
