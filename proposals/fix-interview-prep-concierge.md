# Fix — Interview Prep Concierge: Voice-First Stabilization

**Status**: Draft — awaiting audit
**Date**: 2026-04-14
**Branch (when approved)**: `fix/interview-prep-concierge`
**Scope**: `components/interview-prep/create/ResumeSection.tsx` + `TargetPositionSection.tsx`
**Severity**: Important (UX-first, out-of-band from the numbered sweep)
**Depends on**: nothing
**Blocks**: nothing in the sweep waves

---

## Why this exists

Directive from the user: *"The Interview Prep creation flow (ResumeSection, TargetPositionSection) feels like a tax form. It is too dense and breaks the Mastery story. Your mission: Stabilize and simplify these components. Focus on a 'Voice-First Concierge' experience."*

The executive persona — a senior IC or director who wants zero friction between "I have an interview next week" and "start prepping" — hits this flow and meets:
- **691 lines of form UI** across two sections (298 + 393)
- **Four competing input methods** per section (upload, paste, URL, manual)
- **Zero voice input on the primary surface.** The only `VoiceInput` in the entire flow lives inside a collapsed, optional "Where I need help" sub-panel in section 2 (line 360).
- **5+ cards per section** with nested labels, dropdowns, icons, radio groups
- **Manual "Next" button** to move from section 1 → section 2 even after validation passes

This is not a voice-first concierge. It is a tax form with a voice button buried three folds deep. The fix is surgical: pull voice to the hero surface of each section, collapse everything else behind progressive disclosure, and let the extraction pipeline do the work the user currently does by hand.

---

## Ground-truth state (verified against main HEAD `2e79be6`)

**ResumeSection.tsx** (298 lines):
- Tab toggle: `Upload / Paste` vs `Link URL` (line 68-76)
- Upload zone with drag-drop + parsing spinner (line 98-148)
- `or paste` divider (line 151-155) then a textarea (line 158-167)
- File preview pill when uploaded (line 81-92)
- `ProfileSummary` card after extraction (line 213-223)
- Additional documents list with label dropdown (line 231-253)
- "Next: Your Target" button on completion (line 278-293)
- **Zero `VoiceInput` import, zero voice surface**

**TargetPositionSection.tsx** (393 lines):
- Job Description card with tab toggle `Paste Text` vs `Link URL` (line 67-155)
- Skill extraction spinner (line 158-165)
- `ProfileSummary` card after extraction (line 173-184)
- `SkillTags` fallback (line 186-207)
- JD additional files (line 210-232)
- **Details & Focus** card with Company Name, Target Role, Interview Focus radio group (line 257-325)
- Collapsible "Where I need help" with the **only `VoiceInput` in either file** (line 348-371)
- "Next: Review" button (line 373-388)

Both wrap `CollapsibleSection`; both take `form: UseInterviewPrepFormReturn`; section progression is orchestrated externally via `openSections` + `autoExpandNext`.

---

## Goals

1. **Voice hero on both sections.** `VoiceInput` is the primary affordance a user sees when the section opens. Voice transcription feeds the same `resumeText` / `jobDescriptionText` state the existing extraction pipeline already consumes — zero backend change.
2. **Progressive disclosure for everything else.** Upload, file URL, paste — all valid, all collapsed under a single muted link ("Or upload / paste / link a URL"). Executive sees one primary action, reveals the rest only when needed.
3. **Automatic section transition.** `autoExpandNext` fires the moment section 1 is valid — no "Next" button click. The button stays as a fallback for keyboard users.
4. **Drop the tax-form density.** Consolidate the "Details & Focus" card with the `ProfileSummary` it duplicates. Kill redundant instructional copy. Keep the `ProfileSummary` card as the refinement surface — that's where concierge UX lands: "here's what I found, edit if wrong."
5. **Zero regression on backend contracts.** All extraction, profile, skill, and plan state handlers stay wired exactly as they are. This is a UI reorganization, not a data model change.

---

## Non-goals

- **No backend changes.** `useInterviewPrepForm` hook stays. Candidate/target profile extractors stay. The prompt templates stay.
- **No removal of upload/paste/URL options.** All three remain behind the progressive disclosure link.
- **No new hooks, no refactor of state management.** Section refactor only; the hook keeps its 40+ returned fields.
- **No changes to Review step** (section 3) or the Plan/Execute surface downstream.
- **No redesign of `CollapsibleSection`.** It is the right primitive; keep it.
- **No LLM-based "just tell me about the role" pre-fill from resume → target.** That is Phase 2 (out of scope), a future fix that needs a design sync and backend work.
- **No touch of `app/interview-prep/create/page.tsx`** parent composition.
- **No deletion of any `VoiceInput` that already works** (the "Where I need help" one stays; we add two more at the hero level).
- **No removal of the Interview Focus radio group.** It's the only interview-type signal the planner has; keep it, just compress.

---

## Design

### Change 1 — ResumeSection: voice hero + progressive disclosure

**New layout (top to bottom):**

```
┌─ Your Background ─────────────────────────────────┐
│                                                   │
│  [ 🎙 Talk through your background ]  ← hero     │
│                                                   │
│  [ textarea — voice transcript + edit area ]      │
│  Max 5-6 rows, voice appends, user can edit.      │
│  Placeholder: "Or type / paste here"              │
│                                                   │
│  {file pill if uploaded}                          │
│                                                   │
│  ───── Or upload a file / paste from a URL ─────  │
│  (collapsed; click to expand → upload zone +      │
│   URL input come in here)                         │
│                                                   │
│  {ProfileSummary if extraction ran}               │
│                                                   │
│  [ + Add a cover letter or portfolio ]  ← muted   │
└───────────────────────────────────────────────────┘
```

**Implementation:**
- Import `VoiceInput` from `@/components/ui/voice-input`.
- Replace the current `TabToggle` (Upload/Paste vs URL) with a single `<div>` that renders:
  1. `VoiceInput` at the top, size `"lg"`, bound to `onTranscript={(text) => setResumeText(resumeText ? \`${resumeText} ${text}\` : text)}`.
  2. The existing textarea below it, always visible, bound to `resumeText` / `handleResumePaste`.
  3. A muted link-button "Or upload a file / paste from a URL" that toggles a new local `showAdvancedInput` state.
  4. When `showAdvancedInput` is true: render the current upload zone + URL input in a subtle panel beneath the textarea.
- The compact file pill (existing `FilePreview`) stays — it appears above the textarea when a file is uploaded.
- `ProfileSummary` card placement unchanged.
- Additional documents: collapse behind a muted "+ Add a cover letter or portfolio" button. Default hidden. Click expands the current list UI. **Only render that button when `resumeText` is non-empty and `additionalFiles.length < MAX_ADDITIONAL_FILES`** (same condition as today).
- Drop the helper copy "Detailed resumes produce better prep" (line 174) — concierge doesn't nag.
- Drop the "or paste" divider (line 151-155) — textarea is primary now, no divider needed.

**State additions (local):**
- `const [showAdvancedInput, setShowAdvancedInput] = useState(false)`
- `const [showAdditionalDocs, setShowAdditionalDocs] = useState(false)`

Neither touches `form` / `useInterviewPrepForm`. Zero hook impact.

**Section heading:** Rename `"Your Background"` → `"Your Background"` (no change; the form is hooked on step 1, heading is fine). Add a subtitle inside the card: `"Talk through it, paste it, or upload — whichever is fastest."`

### Change 2 — TargetPositionSection: voice hero + compress details

**New layout:**

```
┌─ Your Target ─────────────────────────────────────┐
│                                                   │
│  [ 🎙 Tell me about the role ]  ← hero            │
│                                                   │
│  [ textarea — voice transcript + edit area ]      │
│  Placeholder: "Or paste the JD here"              │
│                                                   │
│  ───── Or link to the job posting ─────           │
│  (collapsed; click to expand → URL fetch input)   │
│                                                   │
│  {skill extraction spinner when active}           │
│                                                   │
│  {ProfileSummary if extraction ran}               │
│     └─ Company, Role, Seniority already live here │
│                                                   │
│  [ Interview Focus ] — compressed 4-option row    │
│                                                   │
│  {collapsed "Where I need help" — keep as-is}     │
└───────────────────────────────────────────────────┘
```

**Implementation:**
- Move the **voice hero** to the top of the "Your Target" section (currently the section opens with the Job Description card header + tab toggle).
- The existing Job Description card keeps its textarea but drops the header icon + description ("Job Description · Paste text or provide the URL") — the voice hero replaces that heading.
- `VoiceInput` binds to `onTranscript={(text) => handleJdTextChange(jobDescriptionText ? \`${jobDescriptionText} ${text}\` : text)}`. This triggers the existing skill extraction pipeline automatically.
- Progressive disclosure for URL input (same pattern as Resume section).
- **Kill the standalone "Details & Focus" card** (line 257-325). The `ProfileSummary` card (line 173-184) already shows Company + Role + Seniority after extraction, and users can edit them there. The standalone Company Name / Target Role inputs (line 267-294) are redundant — they exist as a fallback for "user hasn't pasted JD yet." For the voice-first concierge flow, we expect extraction to populate these; keep the standalone inputs as a fallback but render them inline under the textarea area only when `!targetProfile && !extracting`.
- **Interview Focus**: keep the radio group but compress it. The current 2x2 grid with icons + labels + descriptions is ~120 lines of JSX. Simplify to a horizontal pill-button row showing label only (icon optional, description on hover via `title`). Fewer pixels, same functionality.
- "Where I need help" collapsible stays exactly as-is (keep the existing `VoiceInput` there too — it has its own purpose).

### Change 3 — Section transition: auto-expand without "Next" button

Today the "Next: Your Target" button (ResumeSection line 278-293) and "Next: Review" button (TargetPositionSection line 373-388) both exist as primary CTAs. But the form hook already exposes `autoExpandNext` — it's just not fired automatically.

**Implementation:**
- Inside ResumeSection, add a `useEffect` that calls `autoExpandNext(1)` the first time `section1Valid` becomes true AND `!openSections[2]`. Use a local `useRef` to track "have we already auto-expanded once" so the user can collapse section 2 without it reopening immediately.
- Same pattern in TargetPositionSection for `autoExpandNext(2)` on `section2Valid`.
- **Keep the "Next" button** as a visible fallback for keyboard/AT users who want explicit confirmation. Demote it from primary (`btn-primary-elegant`) to a smaller secondary style. Its label stays.

This is the "seamless transition" the directive called out, with zero regression for users who still want the button.

### Change 4 — Density polish (small, shared)

- ResumeSection: remove the "Detailed resumes produce better prep" copy.
- TargetPositionSection: remove "More detail = better prep" copy.
- Both: replace `<Card className="card-elegant">` wrappers around the INPUT area with plain `<div>` — the card chrome is visual noise above the textarea. Keep `Card` for `ProfileSummary`, `SkillTags` fallback, and "Details & Focus" remnants. Two `Card` removals per section.
- Kill the icon chrome on the Job Description card header (line 70-74) — voice hero replaces it.
- Unify spacing: both sections use `space-y-3` at the root but nested cards have mixed `p-4`, `p-5`, `px-4 py-3`. Standardize to `p-4`.

---

## Files changed

| File | Lines delta | Change |
|---|---|---|
| `components/interview-prep/create/ResumeSection.tsx` | ~ −80 / +40 net −40 | Voice hero, progressive disclosure, additional-docs collapse, copy cleanup, auto-expand useEffect |
| `components/interview-prep/create/TargetPositionSection.tsx` | ~ −110 / +50 net −60 | Voice hero, kill Details & Focus duplication, compress Interview Focus row, progressive URL disclosure, auto-expand useEffect, copy cleanup |
| `__tests__/interview-prep/ResumeSection.test.tsx` | new ~80 | VoiceInput presence, onTranscript appends to resumeText, advanced-input toggle, auto-expand fires once |
| `__tests__/interview-prep/TargetPositionSection.test.tsx` | new ~80 | VoiceInput presence, onTranscript triggers handleJdTextChange, Interview Focus pill row, auto-expand fires once |

Total: ~ −100 lines of production JSX + ~160 lines of tests. Net: significantly less component code, significantly more test coverage.

---

## Test plan

### Pre-flight
- [ ] `pnpm type-check` — clean on touched files
- [ ] `pnpm lint` — 0 new warnings
- [ ] `pnpm build` — clean

### Unit tests
- [ ] `ResumeSection`: mounts with an empty form, `VoiceInput` hero is visible at the top
- [ ] `ResumeSection`: `onTranscript("foo bar")` calls `setResumeText` with the appended value
- [ ] `ResumeSection`: upload/URL zone is hidden by default, becomes visible after clicking "Or upload a file"
- [ ] `ResumeSection`: "Add a cover letter or portfolio" button appears only when `resumeText` is non-empty
- [ ] `ResumeSection`: auto-expand to section 2 fires exactly once when `section1Valid` flips true
- [ ] `TargetPositionSection`: voice hero is at the top, `onTranscript` calls `handleJdTextChange`
- [ ] `TargetPositionSection`: Details & Focus standalone inputs render only when `!targetProfile && !extracting`
- [ ] `TargetPositionSection`: Interview Focus pill row renders 4 options
- [ ] `TargetPositionSection`: auto-expand to section 3 fires exactly once when `section2Valid` flips true
- [ ] `TargetPositionSection`: "Where I need help" collapse state preserved
- [ ] No regression: existing TabToggle state persists across tab switches (if we keep any toggle; we may not)

### Playwright E2E on `beta.kitesforu.com` after deploy
- [ ] Navigate to `/interview-prep/create`, verify voice hero visible on section 1
- [ ] Click the microphone → allow permission → speak a test sentence → verify the textarea is populated
- [ ] Verify advanced input (upload/URL) is hidden until clicked
- [ ] Upload a test resume → verify extraction runs → `ProfileSummary` appears → section 2 auto-expands
- [ ] On section 2: voice hero visible → speak a test JD → extraction runs → `ProfileSummary` for target appears
- [ ] Verify Interview Focus pill row renders compactly (no duplicate Company/Role inputs once targetProfile exists)
- [ ] Verify "Where I need help" still expands when clicked
- [ ] Keyboard-only pass: tab through the full flow, confirm every interactive element is reachable
- [ ] Screen reader spot-check: voice hero has an `aria-label`, progressive disclosure buttons are keyboard-operable

### Screenshots
- [ ] `.vision_vault/fix-concierge/` — before + after for both sections

---

## Rollout

1. **Code PR** — branch `fix/interview-prep-concierge` in `kitesforu-frontend`
2. **Docs PR** — none required (no user-visible copy changes beyond section subtitles; the flow architecture stays)
3. **Tooltip flag PR** — consider a one-time coach mark: *"New: talk through your background"* on first visit. Optional, deferrable.

Merge → verify Cloud Run revision → Playwright → screenshots → done.

---

## Alternatives considered

1. **Full rewrite of the Interview Prep create page.** Rejected — out of scope, requires a design sync, and would break the hook contract with the parent page. The stabilization goal is surgical.

2. **Remove upload / paste / URL entirely, voice-only.** Rejected — some users have PDF resumes they want to attach, some want to paste a LinkedIn export. Progressive disclosure keeps all paths available without cluttering the primary surface.

3. **Add a "concierge mode" toggle** that switches between the current tax-form UI and the voice-first UI. Rejected — the tax-form UI is the problem, not an option. Fixing it for everyone is cleaner than adding a toggle.

4. **LLM-driven auto-fill from resume → target role.** Rejected — Phase 2. Needs a backend change (the extractor must emit a "target hint"), a prompt engineering pass, and a design review. Tracked as a follow-up fix, not in this PR.

5. **Replace `CollapsibleSection` with a wizard stepper.** Rejected — the collapsible pattern lets users jump backward, which is what executives actually do ("wait, let me add another document"). Wizard forces a linear path and is more friction, not less.

6. **Move Interview Focus into the voice prompt ("mention what kind of interview you're prepping for").** Rejected for Phase 1 — the extraction pipeline doesn't parse interview-type from free text today, and adding that is a backend change. Keep the radio group but compress its visual weight.

---

## Risks

1. **`VoiceInput` component contract.** I verified it accepts `onTranscript={(text: string) => void}` via TargetPositionSection.tsx:361. Need to confirm `size="lg"` or an equivalent larger variant exists, and that the component has an accessible `aria-label` by default. **Mitigation**: read the `VoiceInput` source before implementation and adjust the sizing prop if the variant isn't available.

2. **Auto-expand loops.** If `autoExpandNext(1)` fires in a `useEffect` keyed on `section1Valid`, and collapsing section 2 causes `section1Valid` to transition (which it shouldn't), we get an oscillation. **Mitigation**: use a `useRef<boolean>` guard initialized to `false`, set to `true` after the first auto-expand, and never auto-expand again. Users can still click "Next" to manually expand section 2 if they collapsed it.

3. **Progressive disclosure for upload.** Hiding the upload zone behind a click means some users who prefer upload will take an extra click to find it. **Mitigation**: make the "Or upload a file / paste from a URL" link visibly labeled with an icon so upload-first users can spot it immediately. Optionally, persist `showAdvancedInput` in localStorage if the user opens it once.

4. **`showAdvancedInput` state loses data on collapse.** If the user types into the paste textarea, clicks to expand advanced, uploads a file, then clicks to collapse advanced — does the uploaded file pill still show? **Mitigation**: the `uploadedFileName` + `resumeText` state live in `form`, not in the local `showAdvancedInput` flag. Collapsing the advanced panel does not clear data. But render the file pill ABOVE the advanced panel so it's always visible regardless of toggle state.

5. **Interview Focus compression might lose the descriptive text users relied on.** The current grid shows `{option.label}` + `{option.description}` per radio. Compressing to pills removes the description. **Mitigation**: move the description to `title` attribute (hover tooltip) and keep the label visible. AT users can access it via aria.

6. **Auto-transition + user still editing.** If section 1 becomes valid mid-edit (e.g., resume extraction completes while user is still editing technical skills), auto-expanding section 2 scrolls focus away and interrupts. **Mitigation**: delay auto-expand by ~500ms and skip if the user is actively typing (debounce on last interaction timestamp from `form.touched`).

7. **`CollapsibleSection` layout shift when voice hero is added.** The section header + voice hero + textarea is a taller compact unit than the current flat textarea. Visual regression check required against the existing screenshot.

---

## Open questions for audit

1. **Voice hero sizing** — `VoiceInput` takes `size` prop; what sizes exist? If only `"sm"` and `"md"`, use `"md"` with extra padding. If `"lg"` exists, use it.
2. **Auto-expand UX** — is the 500ms debounce + "skip if actively typing" guard sufficient, or do you want the auto-expand behind a short "preparing next step..." micro-interaction?
3. **Drop the standalone Company Name / Target Role inputs entirely** when `targetProfile` exists, or keep them as read-only mirrors of the extracted values? Recommend: drop entirely, rely on `ProfileSummary` as the edit surface.
4. **Interview Focus pill row** — is 4 pills in one row acceptable on mobile (360px width), or do we need a 2x2 fallback at the smallest breakpoint?
5. **Coach mark** — do you want a one-time "New: talk through your background" tooltip on first visit, or skip?
6. **Ground-truth re-check before implementation** — given the Fix A lesson, should I re-verify both files against main HEAD one more time immediately before the first Edit, to catch any in-flight manual merge?

---

## Triangulation

**STOP here.** This proposal does not authorize any code changes. Awaiting audit.

Pre-flight ground-truth verified against main HEAD `2e79be6`: both component files read end-to-end, line numbers cited, `VoiceInput` contract confirmed via existing usage. Follows the corrected process from the Fix A post-mortem.

Next step on approval: create `fix/interview-prep-concierge` branch off `main`, implement per Design section, run type-check + lint + tests + Playwright on beta, report with diff + test results.
