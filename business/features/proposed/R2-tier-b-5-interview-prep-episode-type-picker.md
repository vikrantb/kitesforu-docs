# R2 Tier B #5 — Interview-Prep Episode-Type Picker

**Status**: PROPOSED (awaiting Codex audit before any code per triangulation rule)
**Priority**: P1 — highest-impact remaining move in the interview-prep arc
**Effort**: 2-3 days, multi-repo (api + frontend; workers-side contract shipped 2026-04-23/24)
**Affected repos**: `kitesforu-api` (route), `kitesforu-frontend` (picker UI), `kitesforu-schemas` (add `episode_type` field to creation payload)
**Depends on (shipped)**: workers #330 / #337 / #338 / #339 / #340 / #342 / #350 / #352 / #353 / #354 — episode_type contract, gate, regen wire, classifier, end-to-end cascade. Activation runbook: `unit_economics_tracking/activation_runbook_2026_04_24.md`.
**Origin**: Tier B item #5 in `content_improvement_ideas/13-second-pass-critique-and-synthesis.md:65`:

> "Episode-type presets for interview prep (file 11.8's Recognition / Drill / Autopsy / Pressure / Final-prep). Surface as picker cards on `/interview-prep` landing. Each card routes into a dedicated generation mode with the right content spine from 11.5. This is the single biggest 'make interview-prep feel like a coach' move the package supports — and the spine structures are already specified."

---

## 1. Problem — why a picker matters

The workers-side episode_type contract is **fully shipped end-to-end** (tested by workers #354's 10-hop cascade). When a job's `preferences["_episode_type"]` is set to one of the 7 documented values, the composer activates a specialized craft block (#337) and the quality gate runs episode_type-specific verification (#338). Regen fires on violation with a targeted hint (#339). Post-gen metrics land in Firestore for operator aggregation (#343/#347).

But nothing sets `preferences["_episode_type"]` from the UI today:

- **Explicit user intent**: no UI knob. A user creating an interview-prep session cannot tell the system "I want a pressure simulation" or "I want an autopsy of last week's answer."
- **Classifier coverage** (#342): conservative phrase-match; fires on ~5-10% of titles where the user happens to write "Night before your interview" or "Autopsy of my STAR answer." The other 90%+ of titles never trigger the specialized craft.

Net effect: **the highest-value quality work in this session is sitting dormant for the majority of interview-prep jobs**. Users get generic interview-prep content regardless of whether they need drill practice, pressure simulation, autopsy review, or day-of rehearsal. The coach-vs-narrator distinction the Tier A work unlocked requires the listener to know which mode they're in — and the system can't tell them because the UI doesn't ask.

## 2. Shape of the change

Three surfaces change; the contract between them is already established.

### 2.1 Frontend — picker cards on `/interview-prep`

Add a **5-card picker** as the primary call-to-action above the existing prompt box on `/interview-prep`. Each card:

- Large mode title + one-sentence explanation of when to use it
- Subtle "what this sounds like" preview (1-2 line sample interaction)
- Visual signal indicating expected duration + intensity
- On click: pre-fills the creation form with the episode_type AND prepends card-specific placeholder guidance to the topic input

The 5 cards map to the 5 episode_types that fit interview-prep use cases (the other 2 — `lecture` and `recognition` — fit educational content better):

| Card                | episode_type   | Use-when                                      | Expected duration |
|:--------------------|:---------------|:----------------------------------------------|:------------------|
| **Drill Practice**  | `drill`        | "I want to practice STAR answers out loud"    | 10-20 min         |
| **Mock Pressure**   | `pressure`     | "I want a simulated high-stakes interview"    | 15-25 min         |
| **Answer Autopsy**  | `autopsy`      | "I want to dissect a weak answer I gave"      | 10-15 min         |
| **Day-of Rehearsal**| `final_prep`   | "Interview is tomorrow — I need a calm reset" | 10-15 min         |
| **Pattern Spotting**| `recognition`  | "I want to recognize common question patterns"| 15-25 min         |

Pick-one UX (radio-style). "Not sure" opens a 5-question wizard that resolves to a recommendation.

**Frontend files to touch** (based on the existing `/interview-prep/hub` shape from R2-phase2):

- `app/interview-prep/page.tsx` — add `<EpisodeTypePicker>` above the existing creation form
- `components/interview-prep/EpisodeTypePicker.tsx` (new) — the 5-card component
- `lib/episode-type-presets.ts` (new) — card metadata (title, description, placeholder, icon)
- `components/smart-create/IntentSection.tsx` — accept optional prefilled `episode_type` + placeholder from the picker

### 2.2 API — accept `episode_type` on creation payload

Add optional `episode_type` field on the create-podcast request (and analogous endpoints for course-mode if applicable). Route's one-liner job: validate against the documented vocabulary, pass through to Pub/Sub payload on `preferences._episode_type`.

**Schemas** (`kitesforu-schemas`):

```python
class CreatePodcastRequest(BaseModel):
    # ... existing fields ...
    episode_type: Optional[Literal[
        "drill", "retrieval", "pressure", "autopsy",
        "lecture", "recognition", "final_prep"
    ]] = None
```

Schemas repo bump: minor version. Existing fields unchanged — additive only.

**API** (`kitesforu-api`):

- `routes/podcasts.py` — accept + validate `episode_type`, write to `job_state["preferences"]["_episode_type"]`
- Test: submit request with each of 7 episode_type values → confirm round-trips to Pub/Sub message preferences

### 2.3 Frontend creation flow — thread the selection

The existing SmartCreate flow (`components/smart-create/IntentSection.tsx`) calls an API creation route. When the picker has set `episode_type`, include it in the POST payload. When no picker selection (user typed directly without picking a card), omit the field — the classifier (#342) will still have a shot at resolving it if the flag is on.

## 3. Non-goals / explicit scope limits

To keep this shippable in 2-3 days and prevent scope creep:

- **No new worker code**. Workers-side contract is done. If a new episode_type shape surfaces as needed, that's a separate PR in workers.
- **No Firestore schema change**. `preferences` is already a dict on the job doc; adding a new key is additive.
- **No analytics event for picker clicks** — ship the picker first, add analytics in a follow-up once we measure whether cards get used.
- **No card-specific generation temperatures or outline shapes** — the episode_type craft block (#337) already specializes the prompt. Don't add a second knob.
- **No "classifier fired and would have routed to X, override picker?" confirmation** — keep picker as the source of truth when set; classifier is fallback when unset.

## 4. Acceptance criteria

- [ ] User visiting `/interview-prep` sees a 5-card picker above the creation form.
- [ ] Each card has a distinct title, one-sentence explanation, and duration/intensity signal.
- [ ] Clicking a card pre-fills the creation form with the episode_type AND card-specific placeholder text.
- [ ] Submitting the form POSTs `episode_type` on the creation payload.
- [ ] API validates `episode_type` against the 7-value vocabulary; rejects unknown values with a 422.
- [ ] `preferences._episode_type` appears in the Pub/Sub message to the workers.
- [ ] (Verified via existing cascade) Workers composer activates the right craft block; gate fires episode_type-specific check; regen applies targeted hint.
- [ ] Firestore `stages.quality_gate.final_prep_no_new_material` (or whichever episode_type-gated check applies) appears on jobs created via the picker.
- [ ] Accessibility: cards are keyboard-navigable, have proper ARIA labels, screen-reader reads the one-sentence explanation.
- [ ] Test coverage: frontend unit test for `EpisodeTypePicker` rendering + click behavior; API integration test for each episode_type value round-tripping to Pub/Sub.
- [ ] Tooltip flag for "Not sure which mode?" wizard — per the user's shipping playbook, this is the "Tooltip flag PR" in the atomic unit (1 Code + 1 Docs + 1 Tooltip).

## 5. Measuring whether the picker works

Post-ship, two metrics tell us the picker succeeded:

1. **Picker activation rate**: fraction of `/interview-prep` creation sessions that click a card. Baseline = 0%. Target = ≥ 50% within 2 weeks (users are actively choosing mode instead of going to the generic form).
2. **Gate-firing rate**: fraction of interview-prep jobs where an episode_type-gated check (`final_prep_no_new_material` etc.) appears in `stages.quality_gate`. Pre-picker baseline = classifier-only firing rate (~5-10% of titles). Post-picker target = ≥ 50% (picker-set jobs) + residual classifier baseline.

Both metrics are already persisted by tonight's workers arc — no new instrumentation needed. `python scripts/analyze_regen_effectiveness.py` from workers #347 already surfaces per-reason resolution rates; adding a picker-coverage histogram alongside is a ~10-line extension.

## 6. Dependency graph

```
This picker                ← this proposal ships the UI + API layer
       ↑
workers episode_type axis  ← shipped 2026-04-23/24 (see runbook)
       ↑
ctx.episode_type signal    ← workers #330 plumbed the field
```

And the full cascade (already verified by workers #354):

```
picker → API → preferences._episode_type → workers composer ctx
  → EpisodeTypeSpecializationBuilder activates craft block
  → LLM generates specialized script
  → run_quality_gate runs episode_type-gated check
  → violations → regen with targeted hint
  → diff_regen_reasons → Firestore metrics for effectiveness tracking
```

The picker is the last missing piece. Every link below it is shipped and tested.

## 7. Risks & mitigations

| Risk                                                | Mitigation                                              |
|:----------------------------------------------------|:--------------------------------------------------------|
| Users ignore cards, picker activation rate is low   | Add a "Not sure which mode?" wizard as 6th pseudo-card. If still low after 2 weeks, move the picker into the smart-create flow itself rather than a separate landing. |
| Cards confuse users who don't know interview-prep lingo ("autopsy"?) | Card copy skips jargon — title is plain-English ("Answer Autopsy" → "Dissect a weak answer"). One-sentence explanations prioritize user-intent over system-intent. |
| Picker routes to the wrong episode_type (e.g. user picks "drill" but really wants "pressure") | Classifier is still a fallback layer even when picker is set — but its output doesn't override the picker. The user who picks drill and gets pressure content is the "wizard wasn't specific enough" failure, not the classifier's. Address via wizard iteration post-ship. |
| Workers-side behavior changes break the picker | Existing workers tests (`test_episode_type_end_to_end_cascade.py` from #354) pin the contract. Any future workers change that breaks the contract fails the cascade test at commit time. |

## 8. Rollout plan

1. **Schemas PR**: add `episode_type` to the request model. Bump minor version.
2. **API PR**: accept + validate + thread to Pub/Sub payload.
3. **Frontend PR**: ship `EpisodeTypePicker` component, wire to SmartCreate.
4. **Tooltip flag PR**: "Not sure which mode?" wizard behind a feature flag for staged rollout.
5. **Docs PR**: update user-facing docs (Mintlify) with the 5 modes and when to use each.

Per the user's shipping playbook: the Code + Docs + Tooltip-flag PRs ship as one atomic unit. The Schemas + API PRs precede them.

## 9. Codex audit asks

Per the triangulation rule, this proposal should be reviewed by Codex before any code lands. Specific questions for the audit:

1. **Card copy tone**: does the 5-card vocabulary (Drill / Pressure / Autopsy / Rehearsal / Pattern) align with real users' mental model of interview prep? Or are we imposing a taxonomy from file 11.8 that users don't think in?
2. **Wizard vs picker**: a 5-card picker assumes the user knows which mode they need. A wizard (7 questions → recommendation) is more forgiving but adds friction. Which is right for the FIRST ship?
3. **Placement**: picker ABOVE the existing creation form vs embedded inside the SmartCreate flow. Affects discovery; don't want to create two parallel creation paths.
4. **Classifier + picker interaction**: if a user types "Night before your interview" in the topic and doesn't click a card, should the classifier silently route to final_prep, or should we force a picker click?
5. **Episode-type scope**: this proposal ships 5 cards (interview-prep-relevant). Should `lecture` and `recognition` also get picker cards for educational content in a separate proposal, or bundle them here?

## 10. Why this is the best next move after tonight's arc

- **Maximum leverage on sunk investment**: 24 workers PRs shipped the contract. One picker ships its user-facing surface. Ratio is enormous.
- **Only remaining blocker to 100% coverage**: classifier (#342) resolves ~5-10% of titles. Picker extends to every user who wants it.
- **Measurement-ready**: post-picker metrics (picker activation, gate firing) are already persisted by tonight's arc. No new instrumentation.
- **Self-contained**: 3 repos (schemas + api + frontend), 5 PRs total, 2-3 days. No cross-team coordination.
- **Next highest-leverage move is substantively larger** (LLM-as-judge for remaining 3 gates needs cost-benefit analysis; series memory × episode_type needs frontend + persistence work).

**After this picker ships, the interview-prep arc is truly complete end-to-end**: user intent → UI picker → API → workers → craft + gate + regen → observability. The last mile of tonight's 24-PR arc is the user being able to actually SET the episode_type that everything else is built to respect.
