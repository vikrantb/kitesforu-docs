# Interview Prep — Three-Products-One-Label Clarification

**Status**: Draft — awaiting Codex audit
**Author**: UI backlog, Fix #3 of 12
**Date**: 2026-04-13
**Depends on**: none (isolated UI/copy change)
**Prior fixes merged**: Fix #1, Fix #2 (UI backlog)

---

## Problem

Three distinct products live under the "Interview Prep" brand but share a single label in the top-level Navbar and are inconsistently surfaced on the landing page. Users clicking "Interview Prep" in the Create dropdown get the landing page; users searching via Smart Create intent routing get dropped onto the podcast builder — and the two flows never acknowledge that the other exists.

The three products:

| Route | What it actually is | Current user-visible label |
|---|---|---|
| `/interview-prep/mock/text` | Adaptive text-based mock interview with STAR rubric + Elo | "Mock interview" (on landing), nothing in Navbar |
| `/interview-prep/mock/voice-preview` | Scripted voice-first demo of the Gold Standard UX | "Gold Standard voice demo" (on landing), nothing in Navbar |
| `/interview-prep/create` | Podcast builder that generates a multi-episode audio prep course from resume + JD | Nothing on landing (not linked), "Interview Prep" in Navbar Create dropdown |

**Root confusion**: the Navbar's single "Interview Prep" entry points at `/interview-prep` (the landing page), but the landing page has no card for the third product. So the Navbar entry is effectively advertising the podcast builder while the landing page hides it.

## Goal

Every user-visible surface that names these products must use the same three canonical labels, and each surface must give the user a clear path to all three.

### Canonical labels (confirmed)

1. **Interactive Mock Interview** → `/interview-prep/mock/text`
2. **Voice-First Simulation** → `/interview-prep/mock/voice-preview`
3. **Custom Interview Series** → `/interview-prep/create`

## Non-goals

- **No route renames.** 27 API references, intent classifier, analytics, and deep links all hardcode `interview-prep/create`. A namespace move is a separate, coordinated refactor — explicitly deferred.
- **No backend schema or enum changes.** `content_type: 'interview-prep'` stays in Smart Create.
- **No new pages.** Landing page, Navbar, and PlanSection only.

## Design

### A. Navbar (`components/Navbar.tsx`)

**Current** (line 14-23): `CREATE_ITEMS` is a flat list. The single Interview Prep entry:

```ts
{ href: '/interview-prep', label: 'Interview Prep', description: 'Mock interview coaching', icon: '\uD83D\uDCBC' }
```

**Proposed**: replace that single entry with a three-item section group using the existing `section` field supported by `NavDropdownItem` (same mechanism already used by "Audio", "Education", "Writing"):

```ts
{
  href: '/interview-prep/mock/text',
  label: 'Interactive Mock Interview',
  description: 'Adaptive text practice with STAR rubric',
  icon: '\uD83D\uDCBC',
  section: 'Interview Prep',
},
{
  href: '/interview-prep/mock/voice-preview',
  label: 'Voice-First Simulation',
  description: 'Talk through answers out loud',
  icon: '\uD83C\uDFA4',
},
{
  href: '/interview-prep/create',
  label: 'Custom Interview Series',
  description: 'AI-generated audio prep course from your resume',
  icon: '\uD83D\uDCDA',
},
```

**Why section grouping instead of a nested submenu component**: `NavDropdown` does not currently support nested children, and adding hover/click submenu logic (plus mobile accordion) is a separate component change with its own testing surface. The existing `section` header pattern already gives the user a single "Interview Prep" header with three explicit modes beneath it, keeping the Create dropdown visually consistent with how "Audio" and "Education" groups already work. Zero component changes, same user-visible outcome: one header, three modes.

If the user strongly prefers a true nested submenu, that becomes a second PR that extends `NavDropdown` with a `children` concept. Flagging as an alternative in the "Alternatives considered" section below.

### B. Interview Prep landing page (`app/interview-prep/page.tsx`)

**Current**: two-card grid (`md:grid-cols-2`) showing only text mock + voice preview. The `/interview-prep/create` podcast builder is completely absent.

**Proposed**:
- Change grid to `md:grid-cols-3` (or a 2+1 stack on desktop if a third card makes the row crowded — visual QA decision).
- Add a third `PathCard` pointing at `/interview-prep/create` with:
  - eyebrow: `"Audio Course"`
  - title: `"Custom Interview Series"`
  - description: `"Generate a multi-episode audio prep course from your resume and target role. Listen on your commute."`
  - cta: `"Build your series"`
  - icon: `<BookHeadphones />` or existing `lucide-react` equivalent (`Headphones`)
- Update the existing text-mode card eyebrow from `"Text Mode · Recommended"` to `"Text Mode · Recommended"` (keep) but change its title from `"Mock interview"` to `"Interactive Mock Interview"` to match the canonical label.
- Update the voice-preview card title from `"Gold Standard voice demo"` to `"Voice-First Simulation"`. The existing `StackedMicroTooltip` shadow-mode copy stays — that's orthogonal to this fix.

### C. Smart Create plan display (`components/smart-create/PlanSection.tsx:33`)

**Current**:
```ts
'interview-prep': { label: 'Interview Prep', icon: '\uD83D\uDCBC', color: '...' }
```

**Proposed**:
```ts
'interview-prep': { label: 'Custom Interview Series', icon: '\uD83D\uDCDA', color: '...' }
```

**Why**: Smart Create's intent classifier routes users to `content_type: 'interview-prep'` which produces the podcast builder. When the plan card says "Interview Prep" the user can't tell if they're about to build an audio series or start a mock interview. Binding that content-type to the third product's canonical label resolves the ambiguity. The text mock and voice preview aren't Smart Create outputs, so this mapping is unambiguous.

### D. Backend (`kitesforu-api`)

**No code changes.** The `intent.py` `AVAILABLE_ROUTES` entry that maps interview intent to `/interview-prep/create` stays correct — it's routing to the Custom Interview Series, which is what "build me an interview course" should produce. Only the frontend label needs to change.

**One follow-up**: the `label: "Interview Prep"` string inside `intent.py` `AVAILABLE_ROUTES` is used for LLM prompting (intent classification), not user-visible display. Leave it alone. Confirm during Codex audit that this string doesn't leak to the user.

## Files changed

| File | Change | Est. lines |
|---|---|---|
| `kitesforu-frontend/components/Navbar.tsx` | Replace 1 interview-prep item with 3 grouped items | ~15 |
| `kitesforu-frontend/app/interview-prep/page.tsx` | Add third PathCard + rename existing card titles + grid update | ~35 |
| `kitesforu-frontend/components/smart-create/PlanSection.tsx` | Update label + icon for `interview-prep` content type | 1 |
| `kitesforu-frontend/tests/**` | Navbar rendering test for new three-item group | ~20 |

**Total**: ~70 lines across 4 files. Single PR.

## Test plan

- [ ] `pnpm type-check` clean
- [ ] `pnpm lint` clean
- [ ] `pnpm build` clean
- [ ] Jest unit test: `Navbar` renders all three interview prep items under the "Interview Prep" section header when the Create dropdown opens
- [ ] Playwright on `beta.kitesforu.com` after deploy:
  - [ ] Open Create dropdown → verify "Interview Prep" section header visible → verify three distinct items with correct labels → click each, verify correct route loads
  - [ ] Navigate to `/interview-prep` → verify three cards visible → click Custom Interview Series card → verify lands on `/interview-prep/create`
  - [ ] Run Smart Create with prompt "help me prep for a backend interview" → verify PlanSection label reads "Custom Interview Series" not "Interview Prep"
- [ ] Screenshot all three surfaces (Navbar dropdown, landing, Smart Create plan) to `.vision_vault/fix-03/`

## Rollout

Per shipping playbook, this is one atomic unit:

1. **Code PR** (`kitesforu-frontend`, branch `fix/interview-prep-label-clarification`) — the changes above
2. **Docs PR** (`kitesforu-docs`) — update any user-facing docs that reference "Interview Prep" generically; confirm Mintlify nav mirrors the three-mode structure
3. **Tooltip flag PR** — none required; no new features, only relabels of existing surfaces. If the user-facing help center has a tooltip pointing at Create → Interview Prep, update that copy as part of the Docs PR.

Ship 1 and 2 together. Skip 3 unless audit identifies a help-center tooltip that references the old label.

## Alternatives considered

1. **True nested submenu under a single "Interview Prep" entry.** Requires extending `NavDropdown` to support `children`, implementing hover/click submenu logic on desktop, and an accordion expand pattern on mobile. Cleaner conceptually but adds a component-level change with its own testing surface and a11y considerations. Deferred — the section-header pattern achieves "one header, three explicit modes" with zero component risk.

2. **Namespace move: rename `/interview-prep/create` to `/audio-course/interview-prep` or similar.** Rejected per research findings: 27 API references, intent classifier hardcoding, analytics funnel dependencies, deep links from email templates. Deferred to a coordinated IA refactor with its own proposal.

3. **Drop the podcast builder entirely from the Create dropdown, only expose it from Smart Create intent routing.** Rejected — the podcast builder is a real product with paying users; hiding its discovery surface is a regression, not a fix.

## Risks

1. **Navbar section-group ordering.** `CREATE_ITEMS` section headers render with a `border-t` above when `idx > 0`. Inserting three items in a new "Interview Prep" section will push existing items down; visually verify the Create dropdown still looks balanced at 10-11 items. If it feels long, consider whether "Blog & Newsletter" still belongs in Create or should move under a "Writing" submenu (out of scope for Fix #3 but flag to user).

2. **Landing page grid.** Going from `md:grid-cols-2` to `md:grid-cols-3` reduces each card's width; the primary text-mode card's `shadow-lg` + gradient may feel less prominent. Audit visually before merge. Fallback: keep `md:grid-cols-2` for the primary card as a full row and drop the two secondary cards on the row below.

3. **Smart Create regression**: changing `'interview-prep'` label from "Interview Prep" to "Custom Interview Series" assumes every code path that reads `CONTENT_TYPE_LABELS['interview-prep'].label` is showing it in the Smart Create plan context specifically. Grep confirms `PlanSection.tsx` is the only consumer, but Codex audit should double-check for other imports of `CONTENT_TYPE_LABELS`.

## Open questions for audit

1. Is a 3-card landing row visually acceptable, or should the primary card be kept full-width with secondary cards below?
2. Does the Help Center or any Mintlify page reference "Interview Prep" in a way that will now be ambiguous?
3. Any analytics events that read a `label` string (vs route) for interview-prep products that would be broken by relabeling?

## Triangulation

STOP here. This proposal does not authorize code changes. Awaiting Codex audit before implementation begins.
