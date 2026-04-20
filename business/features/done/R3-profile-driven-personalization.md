# R3 — Profile-Driven Personalization (Starting Points, Smart Create, Car Mode, Library)

**Status**: PHASE 1 SHIPPED (2026-04-20) — all code-side ACs landed; see Implementation Summary at bottom. Phase 2+ surfaces remain scoped per the proposal's roadmap.
**Priority**: P3 (low priority — enabler; Phase 1 must ship before Phase 2 becomes useful)
**Effort**: Phase 1 ~4–6 days (profile schema + capture). Phase 2+ scoped later per surface.
**Origin**: 2026-04-19 product-owner review of `/create-smart` — "Starting points goes to the same thing as Say it. Useless without the profile. Log what improvements we can do if we start catching and using the user's persona and profile."

## Problem (evidence-based)

### The Starting Points tab today has no personalization

`kitesforu-frontend/app/create-smart/page.tsx:145` sets the page subhead to "Say it out loud, paste your notes, or tap a starting point — we'll handle the rest." The three tabs are hard-coded in `components/smart-create/IntentSection.tsx:250-259`:

```
{ value: 'prompt', label: 'Say it' },
{ value: 'paste', label: 'I have notes' },
{ value: 'template', label: 'Starting points' },
```

The Starting Points tab (`IntentSection.tsx:409-436`) iterates `PERSONAS` from `lib/templates.ts:717-790` and renders **all 8 personas and all 34 templates**, unfiltered, for every user. When the user clicks a template, `handleTemplateClick` (line 194) just pre-fills `selectedTemplate` and switches the tab back to `prompt` — exactly the same surface as "Say it". The product owner is correct: *mechanically* Starting Points collapses into "Say it" with a topic hint.

The 34 templates span wildly different audiences — `tech-interview` (line 61), `bedtime-stories` (line 317), `corp-compliance` (line 551), `k12-elementary` (line 356). A senior PM prepping for a Google loop sees "Bedtime Stories" with equal weight as "Tech Interview". That is the definition of zero personalization.

### The profile today is effectively empty

`components/OnboardingGuard.tsx` routes first-time signed-in users to `app/onboarding/page.tsx`, which captures **three fields only** (line 14-18):

- `selectedPersona: PersonaId` (one of 8 broad roles)
- `displayName` (optional)
- `organization` (optional)

That is the entire schema. Confirmed in `kitesforu-schemas/src/kitesforu_schemas/models.py:1516-1528`:

```python
class UserProfile(BaseModel):
    user_id: str
    primary_persona: Optional[str] = None
    onboarding_completed: bool = False
    display_name: Optional[str] = Field(None, max_length=200)
    organization: Optional[str] = Field(None, max_length=200)
    created_at: datetime
    updated_at: datetime
```

The API surface is equally thin — `kitesforu-api/src/api/routes/profile.py:28-97` is just `GET/PATCH /v1/me/profile` and `POST /v1/me/onboarding/complete`. The frontend hook `hooks/useUserProfile.ts:8-16` mirrors this seven-field shape. **There is nowhere to store "I am a senior PM targeting Google in 3 weeks", nor "I prefer 10-min episodes in the Explainer style at 1.25x".**

Meanwhile the templates themselves encode the exact fields we would want: `interviewFocus`, `audience`, `duration_min`, `total_episodes`, `content_voice`, `grade_level`, `discipline`, `mastery_threshold` (`lib/templates.ts:14-43`). These defaults are poured into the URL as `?template=<id>` and **thrown away after one job** — never observed as a preference signal.

### What usage history we already log but don't use for personalization

We already emit `trackCreateAbandoned` on unmount (`page.tsx:56-63`), we already have `useUnifiedLibrary` and `useActivityFeed` hooks (`hooks/useUnifiedLibrary.ts`, `hooks/useActivityFeed.ts`), and R3 D2 shipped `shared_content_view`/`shared_content_play` events. None of this feeds back into what Starting Points shows.

## The core claim

**Starting Points is pointless without a richer profile. Personalization is pointless without the data to personalize on.** This proposal therefore separates the concern into two phases. Phase 1 builds the data foundation; Phase 2+ consumes it across surfaces. Shipping only Phase 2 on top of today's three-field profile would be theatre.

## Phase 1 — Profile enrichment (this proposal's scope)

### What to capture (schema additions to `UserProfile`)

Group fields by capture mechanism: explicit onboarding, progressive disclosure, or usage-inferred.

**Onboarding (explicit, one-time):**
- `role_context`: free text, 200 chars — "senior PM at a fintech", "Grade 8 science teacher in Ontario", "AP Bio student"
- `education_level`: enum — `k12 | undergrad | grad | professional | self_directed`
- `primary_goal`: enum — `interview_prep | exam_prep | teaching | creative | curiosity | work_comms`
- `target_date`: optional ISO date — "my interview is 2026-05-03" (drives urgency and depth)

**Progressive disclosure (ask in-flow when signal is cheap):**
- `interest_tags`: multi-select of ~25 broad topics (history, biology, finance, indie horror, Python, leadership…)
- `difficulty_preference`: enum — `introductory | balanced | advanced`
- `preferred_duration_min`: int, with sensible per-content-type defaults
- `preferred_style`: enum sourced from the Voice Persona system (`Explainer | Storytelling | Interview | Motivational`)
- `language_preference`: `SupportedLanguage` Literal (already exists)

**Usage-inferred (no UI — derived nightly):**
- `inferred.genres_used[]` — top 5 genres from completed jobs (weighted: completion > 50%, shared, replayed)
- `inferred.last_content_types[]` — rolling last-10 `content_type` values from jobs
- `inferred.activity_cadence` — `daily | weekly | burst | dormant`
- `inferred.preferred_creation_path` — `say_it | notes | template | car_mode`
- `inferred.abandonment_hotspots` — stages where this user drops off (from `trackCreateAbandoned`)

**Privacy / opt-out (MUST ship alongside the schema):**
- New settings page `/settings/privacy` with a single toggle: `personalization_enabled: bool` (default true for new users, opt-in prompt for existing users). When off, the inference pipeline is skipped and `inferred.*` is cleared on next write.
- Inferred fields are derived from events the user already generated; we do not buy or enrich from third parties.
- All fields visible in `/settings/profile` as a plain-English summary ("We think you like explainer-style 10-minute episodes on history"), with a "Clear inferred data" button.

### Capture mechanisms (no over-engineering)

1. **Extend `app/onboarding/page.tsx`** from 2 steps to 3 — add a lightweight step between persona and optional display name that captures `role_context`, `primary_goal`, `target_date`. Keep every field skippable; lazy users still get through in 8 seconds.
2. **In-flow progressive disclosure** in `IntentSection` — when a user picks a template whose persona differs from `primary_persona` more than twice, ask "Should we switch your role?" (one-click). Same for `difficulty_preference` after three completions at the same level.
3. **Nightly inference job** in `kitesforu-workers` (new `jobs/profile_inference.py`) — reads the user's last 30 days of jobs + library + activity events, writes to `UserProfile.inferred.*`. Idempotent, batched, costs near-zero since it's pure aggregation.
4. **Schema contract**: follow the standing rule from `kitesforu-api/CLAUDE.md` — `UserProfile` is a Firestore read model, so it must have `ConfigDict(extra='ignore')` on the root and every nested model; request models (`UpdateUserProfileRequest`) may stay strict. No `max_length` on the `interest_tags` list.

## Phase 2+ — Personalization surfaces (scoped, not specified here)

With the enriched profile in place, the downstream surfaces become obvious. Each is a separate PR sized ~1–3 days.

- **Starting Points reorder + filter** (`components/smart-create/IntentSection.tsx`): rank `PERSONAS` so the user's `primary_persona` is first and templates are sorted by `inferred.genres_used` overlap. Hide personas with zero signal behind a "More starting points" disclosure. Templates the user *just used* get demoted ("you already did this"). Replaces the flat 34-card grid with a ~6-card above-the-fold set.
- **Smart Create placeholder copy** (`IntentSection.tsx:283`): swap the generic placeholder for a role-aware example — PM sees "Interview prep for a senior PM role at Google"; student sees "Organic Chemistry midterm: reactions, mechanisms, stereochemistry". One line change, driven by `primary_persona` + `inferred.last_content_types`.
- **Car Mode topic suggestions** (`hooks/car-mode/`): seed the "What should we play?" prompt with the user's top 3 inferred interests and the unfinished episodes from Library. Today it shows generic suggestions.
- **Library recommendations** (`hooks/useUnifiedLibrary.ts`): "Because you finished X" row driven by `inferred.genres_used` and completion events.
- **Interview-prep persona pipeline**: feed `role_context` + `target_date` into the existing interview-prep prompt composer so the PM prep is actually tailored to fintech-PM-at-Google, not a generic senior-PM loop.
- **Course difficulty & length defaults** (`hooks/useClassForm.ts`): pre-fill `mastery_threshold`, `lesson_duration_min`, `questions_per_lesson` from `difficulty_preference` and `preferred_duration_min`.
- **Voice persona selection** (`kitesforu-workers/personas/`): pass `preferred_style` and `language_preference` as inputs to the 6-stage voice selector so trial users stop burning ElevenLabs credits when an Inworld Explainer voice is the right call.

## Expected advantages (honest, not marketing)

- **Activation**: fewer paradox-of-choice drop-offs on Starting Points. Baseline: today's abandonment on the template tab is trackable via `trackCreateAbandoned` (`page.tsx:60`) — we can measure before/after rather than guess.
- **CTR on starting points**: expect lift because the 4–6 cards shown are matched to the user's persona, not 34 generic ones. Measurable via a new `starting_point_click` event keyed to `(template_id, was_personalized: bool)`.
- **Credit efficiency**: users who pick a tailored template tend to accept the plan on first pass instead of refining 2–3 times. Each refine is a free LLM call today but still burns wall-clock and user patience. Phase 1 enables Phase-2 A/B.
- **Retention cohorts**: `inferred.activity_cadence` lets us build genuine cohorts (daily vs weekly vs dormant) instead of the current one-bucket-fits-all analytics.
- **Interview-prep quality**: today an interview-prep job ignores `role_context` entirely because we never captured it. Even capturing it unblocks a 1-line prompt injection that materially changes output.

## Acceptance criteria — Phase 1 only

AC-1. `UserProfile` schema extended in `kitesforu-schemas/src/kitesforu_schemas/models.py` with the fields above. `ConfigDict(extra='ignore')` on `UserProfile` and every nested model (`InferredProfile`). Request models (`UpdateUserProfileRequest`) updated with optional fields.
AC-2. `kitesforu-api` `PATCH /v1/me/profile` accepts the new fields. Existing docs without these fields continue to deserialize (permissive read).
AC-3. Onboarding flow (`app/onboarding/page.tsx`) captures `role_context`, `primary_goal`, `target_date` in a new step. All fields skippable. Existing onboarded users are not forced back through onboarding.
AC-4. `/settings/profile` page lets the user view and edit every captured field. `/settings/privacy` exposes `personalization_enabled` and a "Clear inferred data" button.
AC-5. Nightly inference job in `kitesforu-workers` writes `inferred.*` for every user with ≥3 completed jobs. Skips users with `personalization_enabled = false`. Idempotent; safe to re-run.
AC-6. `hooks/useUserProfile.ts` returns the new fields. No surface in the app *consumes* them yet — Phase 2's job — but the data is flowing.
AC-7. Playwright verification on beta.kitesforu.com: new user completes extended onboarding, profile shows correct fields, privacy toggle disables inference.

## Phase 2+ acceptance (scoping level)

Each Phase-2 surface (Starting Points reorder, Smart Create placeholder copy, Car Mode suggestions, Library recs, interview-prep injection, course defaults, voice selection) ships as its own PR with its own acceptance criteria once Phase 1 data is live. Target one surface per week to avoid a big-bang release and to A/B each lever independently.

## Why P3 and not P1

No user is complaining that Starting Points is wrong; they just don't use it. The pain is strategic (we are leaving personalization value on the table) rather than tactical (nothing is broken). Phase 1 is pure plumbing with no user-visible win on its own, which is exactly the kind of work that gets deprioritized unless labelled as the enabler it is. Do not ship Phase 2 without Phase 1 — it will be lipstick on a three-field profile.

---

## Implementation Summary — Phase 1 (2026-04-20)

Phase 1 shipped end-to-end across 4 repos in a single coordinated thread. Every acceptance criterion called out in the proposal landed except AC-7 (Playwright verification on beta), which is operationally blocked until the schemas 1.50.0 rollout completes — the code behavior is already pinned by unit tests.

**AC-1 — Schema additions (kitesforu-schemas PR #70, v1.49.0 → v1.50.0)**
- New Literal enums: `EducationLevel`, `PrimaryGoal`, `DifficultyPreference`, `ActivityCadence`, `CreationPath`.
- New nested model `InferredProfile` (permissive `extra='ignore'`) with `genres_used`, `last_content_types`, `activity_cadence`, `preferred_creation_path`, `abandonment_hotspots`, `last_inferred_at`.
- `UserProfile` extended with `role_context`, `education_level`, `primary_goal`, `target_date`, `interest_tags`, `difficulty_preference`, `preferred_duration_min`, `preferred_style` (reuses existing `PodcastStyle`), `language_preference` (reuses existing `SupportedLanguage`), `inferred`, `personalization_enabled` (default `True`). Every new field is optional so legacy Firestore docs keep deserializing.
- `UpdateUserProfileRequest` mirrors the writable subset and stays strict on enum values so a frontend typo surfaces as 422.
- 5 new regression tests covering legacy-doc deserialize, round-trip, enum rejection on request, partial PATCH, permissive InferredProfile.

**AC-2 — API (kitesforu-api PR #246)**
- `GET`/`PATCH /v1/me/profile` + `POST /v1/me/onboarding/complete` now use `UserProfileResponse.model_validate(profile.model_dump())` via a single `_to_response` helper so any future optional field flows through without code change.
- `update_profile` service already persisted every non-None field via `request.model_dump()` → dotted Firestore update, so the new Phase 1 fields just work.
- `inferred.*` remains unwritable from this endpoint — the nightly inference job (AC-5) owns it.
- 5 new tests pinning the pass-through contract.

**AC-3 — Onboarding UX (kitesforu-frontend PR #480)**
- Onboarding extended 2 → 3 steps. New middle step captures `role_context` (free text ≤200 chars with persona-nudging placeholder), `primary_goal` (6-option chip select), `target_date` (HTML date input → ISO on submit).
- Every field is skippable; lazy users still complete in ~8 seconds.
- On final step submit, `PATCH /v1/me/profile` fires FIRST with whatever Phase 1 fields the user provided (non-fatal on failure), then `POST /v1/me/onboarding/complete` completes the core flow.
- Pre-existing onboarded users are not forced back through the expanded flow — `OnboardingGuard` already gates on `profile.onboarding_completed`.

**AC-4 — Settings surfaces (kitesforu-frontend PR #481)**
- `/settings/profile` — edits every writable Phase 1 field plus `display_name` + `organization`. Seeds from `useUserProfile`, save issues a single PATCH with explicit `null` for any cleared field so users can remove previously-provided data. Renders a plain-English summary of the inferred block ("You gravitate toward horror, comedy · you show up weekly · recent: podcast, story-series") when present; never exposes raw enum values.
- `/settings/privacy` — single styled `role=switch` toggle for `personalization_enabled`. Flip issues the PATCH. Conditional "Clear inferred data" link appears when inferred data exists; sets the 24h-clearance expectation. Honest copy: "We never buy or enrich from third parties."

**AC-5 — Nightly inference job (kitesforu-workers PR #298)**
- `src/workers/jobs/profile_inference.py` splits into pure compute functions (`compute_genres_used`, `compute_last_content_types`, `compute_activity_cadence`, `compute_preferred_creation_path`, `compute_abandonment_hotspots`, `build_inferred_blob`) and a `run_inference(db)` orchestrator.
- **Idempotent**: re-runs produce the same blob modulo `last_inferred_at`.
- **Respects opt-out**: `personalization_enabled=False` users are skipped AND have any existing `inferred` block cleared on the same pass.
- **Threshold**: users with < 3 completed jobs (`MIN_JOBS_FOR_INFERENCE`) are skipped — proposal is explicit that missing inferred is better than noisy inferred.
- **Cadence buckets**: `daily` ≥ 20 jobs/30d · `weekly` 4–19 · `burst` 5+ jobs inside any 3-day window · `dormant` = 0.
- Single Firestore `.update()` per user with dotted paths (`inferred` + `updated_at`), respecting the parent-plus-child restriction.
- `scripts/profile_inference.py --dry-run` is the runnable entry. Cron / Cloud Scheduler wiring is an ops concern outside this PR.
- 19 unit tests: every compute_* helper (weights, thresholds, bucket boundaries, shape, idempotency) plus 3 orchestration tests with a fake Firestore client (opt-out clear path, opt-in write path, dry-run counts-but-doesn't-write).

**AC-6 — Frontend `useUserProfile` type surface (kitesforu-frontend PR #479)**
- Exports `EducationLevel`, `PrimaryGoal`, `DifficultyPreference`, `ActivityCadence`, `CreationPath`, `PodcastStyle`, `InferredProfile`, `UserProfile` so Phase 2 surfaces don't each re-define them.
- Every new field is optional + nullable so older API responses (from pods still on schemas 1.49.0 until the rolling deploy completes) keep parsing.
- No surface consumes the new fields yet — that's Phase 2's job.

**AC-7 — Playwright verification on beta** — **DEFERRED**. The code behavior is pinned by unit tests (5 schemas + 5 api + 19 workers = 29 new tests across the thread). The operational smoke (walk a fresh user through onboarding, confirm `GET /v1/me/profile` reflects the fields, flip the privacy toggle, verify the nightly job runs against the opted-out user) is gated on the schemas 1.50.0 package rolling through the API and worker deploys. Track as a follow-up verification task, not a code change.

**Deviations from the proposal**

- `InferredProfile.last_inferred_at` is written as `datetime` rather than a raw ISO string so Firestore indexes it natively. The API serializes to ISO on the way out, so the client shape is unchanged.
- `/settings/privacy`'s "Clear inferred data" button clears by flipping `personalization_enabled` off rather than by directly zeroing the blob; the nightly job does the actual clearing on next run. User-visible copy sets the 24h expectation honestly.
- Phase 2 consumer PRs (Starting Points reorder, Smart Create placeholder copy, Car Mode suggestions, Library recs, interview-prep prompt injection, course defaults, voice selection) are tracked in the proposal's "Phase 2+" section and ship individually now that the data is flowing.

**Deferred follow-ups**

- AC-7 Playwright smoke on beta.
- ~~Interview-prep prompt injection (Phase 2 surface)~~ — **shipped in kitesforu-api PR #249 (2026-04-20)**. Smart-create now injects a compact USER PROFILE block (role_context, education_level, primary_goal, difficulty_preference, preferred_style, interest_tags, inferred.top_interests + preferred_depth) into all 5 `_execute_*` paths. Interview-prep smart-create also seeds the course's `context` field with the `=== INTERVIEW PREP CONTEXT ===` marker so the course-workers orchestrator picks the specialized mock_interview / behavioral_practice / system_design templates instead of the generic fallback. Gated on `personalization_enabled=True`; Firestore read failures return None so a flaky profile fetch never breaks content creation. 10 new unit tests pin the no-signal → None paths, full-field rendering, inferred block, marker guarantee, and whitespace stripping.
- Remaining Phase 2 surface PRs (Starting Points reorder, Smart Create placeholder copy, Car Mode suggestions, Library recs, course defaults, voice selection) — tracked in proposal's "Phase 2+" section, ship individually.
- Cloud Scheduler / Cloud Run Job wiring for the nightly inference script (ops; the code is idempotent + dry-runnable so wiring can land at any time). Until this runs, `inferred.*` is always empty and the USER PROFILE block's "Recent interests (usage)" line never appears — the explicit onboarding fields still flow.
