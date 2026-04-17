# Fix A — Creation Flow Prop Alignment (S5)

**Status**: Draft — awaiting audit
**Date**: 2026-04-14
**Part of**: UI Excellence Sweep, Wave 1, Fix A
**Maps to**: Issue S5 in `ui-excellence-sweep.md`
**Severity**: Critical — shipping bug, not a polish item
**Depends on**: nothing
**Blocks**: nothing in the sweep — pure bug fix, ships first

---

## Problem

The smart create page (`app/create-smart/page.tsx`) reads `?template=` and `?text=` from the URL and tries to pass them into three different sub-components depending on mode and state. All three hand-offs are broken because the page's call sites drifted from the component interfaces. Templated creation URLs silently fall through to blank flows, and the clarifying-questions skip path is dead.

This affects the Navbar Create dropdown today. Eight entries in `CREATE_ITEMS` (`Navbar.tsx:14-34`) carry a `?template=...` query param — every single one lands on a blank create flow with no hydrated intent.

This is the highest-leverage shipping bug in the sweep. Wave 1 Fix A.

---

## Three concrete mismatches

### Mismatch 1 — `ChatSection`

**Parent call site** (`app/create-smart/page.tsx:113-123`):

```tsx
<ChatSection
  state={chat.state}
  messages={chat.messages}
  error={chat.error}
  isStreaming={chat.isStreaming}
  onSendMessage={chat.sendMessage}
  onExecute={chat.execute}
  onReset={chat.reset}
  initialText={initialText}
  templateId={templateId}
/>
```

**Component interface** (`components/smart-create/chat/ChatSection.tsx:15-29`):

```ts
interface ChatSectionProps {
  state: ChatState
  messages: ChatMessage[]
  plan: ContentPlan                       // ← parent never passes
  error: string | null
  isStreaming: boolean
  onSendMessage: (text: string) => void
  onExecute: (editedPlan?: ContentPlan) => void
  onDriveExecute?: () => void
  onUpdatePlan: (updates: Partial<ContentPlan>) => void  // ← parent never passes
  onRetry: () => void                                    // ← parent never passes
  onReset: () => void
  economyMode?: boolean
  onEconomyModeChange?: (v: boolean) => void
}
```

**Delta**:
- Parent passes `initialText`, `templateId` — not in the interface, dropped at runtime.
- Parent omits `plan`, `onUpdatePlan`, `onRetry` — required by the interface. These are undefined at runtime, so any internal call to `onUpdatePlan(...)` or `onRetry()` crashes, and any render that reads `plan.*` throws on the dot access.
- CI `tsc --noEmit` should flag this as a TypeScript error, but per the frontend CLAUDE.md the test-job type check runs with `continue-on-error: true`, so it's shipping silently.

### Mismatch 2 — `IntentSection`

**Parent call site** (`app/create-smart/page.tsx:127-131`):

```tsx
<IntentSection
  onSubmit={batch.submitIntent}
  initialText={initialText}
  templateId={templateId}
/>
```

**Component interface** (`components/smart-create/IntentSection.tsx:17-26`):

```ts
interface IntentSectionProps {
  onSubmit: (input: {
    template_id?: string | null
    initial_text?: string | null
    pasted_content?: string | null
  }) => void
  disabled?: boolean
  defaultTemplateId?: string | null  // ← parent passes `templateId` instead
  defaultText?: string | null        // ← parent passes `initialText` instead
}
```

**Delta**: Name drift. `initialText`/`templateId` (parent) vs `defaultText`/`defaultTemplateId` (component). Silent drop; batch mode never hydrates from query params.

### Mismatch 3 — `QuestionSection`

**Parent call site** (`app/create-smart/page.tsx:141-146`):

```tsx
<QuestionSection
  questions={batch.questions}
  onSubmit={batch.submitAnswers}
  onSkip={() => router.push(getSkipUrl(batch.questions))}
/>
```

**Component interface** (`components/smart-create/QuestionSection.tsx:9-13`):

```ts
interface QuestionSectionProps {
  questions: QuestionItem[]
  onSubmit: (answers: Record<string, string>) => void
  disabled?: boolean
  // no onSkip
}
```

**Delta**:
- Parent passes `onSkip` — not in interface, silently dropped. The skip button (if the child renders one) has no handler; if the child doesn't render one, the skip path is entirely unreachable.
- `getSkipUrl(batch.questions)` is called with `QuestionItem[]`. Per the agent report, `getSkipUrl` expects a plan object; calling it with a `QuestionItem[]` is also a type bug. Need to verify `getSkipUrl`'s actual signature during implementation — the fix may extend further than this proposal.

---

## Goal

Every templated creation URL hydrates the correct mode with the correct intent. Skip button in clarifying questions actually skips. Zero runtime crashes. Zero silent prop drops.

---

## Non-goals

- **No refactor of the creation hooks** (`useSmartCreate`, `useSmartCreateChat`). Scope is prop alignment only.
- **No changes to the intake backend** (`/v1/smart-create/intake`, `/v1/smart-create/chat`). The DTOs stay the same.
- **No design changes** to the creation flow UI. If a prop is renamed, internal UI stays identical.
- **No new template query params.** Use what's there today.
- **No fix for other prop drift** outside these three call sites. If `ThinkingIndicator.label` (S5-adjacent, reported by agent) is broken, it's a separate follow-up.

---

## Design

Three surgical changes plus one investigation.

### Change 1 — `ChatSection` must actually support templated init

Two options here; Option A is the recommended one.

**Option A (recommended) — hydrate through the chat hook, not through props.**
`useSmartCreateChat` already owns state. Add two optional params to the hook's initializer or expose a `hydrate({ initialText, templateId })` method. Parent calls `hydrate` once on mount when the query params are present. The `ChatSection` interface stays clean (no new props).

**Option B — accept `initialText`/`templateId` on the interface.**
Add both as optional props on `ChatSectionProps`. Inside, push them into the hook on mount via a `useEffect`. Less clean because it mixes URL plumbing into a UI component, but lower-blast-radius.

**Additional required fix regardless of option**: the parent must also pass the three missing required props — `plan`, `onUpdatePlan`, `onRetry`. These live on the chat hook today (`useSmartCreateChat`). If they don't, this proposal's scope extends by one commit: expose them on the hook. Verify during implementation, not during proposal.

### Change 2 — `IntentSection` prop-name alignment

Rename at the call site. Minimal change.

```diff
 <IntentSection
   onSubmit={batch.submitIntent}
-  initialText={initialText}
-  templateId={templateId}
+  defaultText={initialText}
+  defaultTemplateId={templateId}
 />
```

No component change. Two lines.

### Change 3 — `QuestionSection` skip path

Two sub-parts:

**3a.** Add `onSkip` to the interface:

```diff
 interface QuestionSectionProps {
   questions: QuestionItem[]
   onSubmit: (answers: Record<string, string>) => void
   disabled?: boolean
+  onSkip?: () => void
 }
```

Render a "Skip" button in the component if `onSkip` is provided. Ghost button, bottom of card, `text-gray-500 hover:text-gray-700`. Do not render if `onSkip` is absent.

**3b.** Fix the `getSkipUrl` call. The current call passes `batch.questions` (a `QuestionItem[]`). **Investigation needed**: read `getSkipUrl`'s actual signature in `components/smart-create/utils.ts`. Three likely outcomes:
- It expects a plan — pass `batch.plan` instead (if available at question state; may not be).
- It expects `{ template_id }` — pass `{ template_id: templateId }`.
- It's dead code — rip it out and have `onSkip` just call `batch.reset()` plus `router.push('/')`.

This proposal does not pre-commit to a fix; the implementer reads `utils.ts` first and picks the right approach. Flag in the code PR description which branch was taken.

### Change 4 — CI type-check hardening (scope extension check)

The agent report flagged that `tsc --noEmit` in CI is `continue-on-error: true`, which is how three interface mismatches shipped. Two options:

**Option A**: fix only these three call sites in this PR; leave CI lenient. Fast, surgical, accepts the debt.

**Option B**: in the same PR, flip the type-check step from `continue-on-error: true` to hard gate, then fix every other type error the repo surfaces. High risk — may find 20+ additional errors, blows up scope.

**Recommendation**: Option A for this fix. File a separate issue "CI type-check is lenient — harden". Out of scope here.

---

## Files changed

| File | Change | Est. lines |
|---|---|---|
| `kitesforu-frontend/app/create-smart/page.tsx` | Rename 2 props on IntentSection; call `ChatSection` with correct props (Option A hydration or Option B extension); verify `QuestionSection` onSkip path | ~10 |
| `kitesforu-frontend/components/smart-create/chat/ChatSection.tsx` | Option B: add `initialText`/`templateId` props; or Option A: no change | 0 or ~6 |
| `kitesforu-frontend/hooks/useSmartCreateChat.ts` | Option A: expose `hydrate({ initialText, templateId })`; verify `plan`/`onUpdatePlan`/`onRetry` exist | ~15 |
| `kitesforu-frontend/components/smart-create/QuestionSection.tsx` | Add `onSkip?` prop, render skip button when provided | ~12 |
| `kitesforu-frontend/components/smart-create/utils.ts` | Fix `getSkipUrl` signature mismatch (investigation-dependent) | ~5 |
| `kitesforu-frontend/tests/smart-create/*.test.tsx` | Unit test for each of the three call sites + a template-URL regression test | ~40 |

Total: ~80 lines across 5-6 files. Single PR.

---

## Test plan

### Pre-flight
- [ ] `pnpm type-check` — must be clean for the files touched. Zero tolerance on this PR.
- [ ] `pnpm lint` — clean.
- [ ] `pnpm build` — clean. Note existing CI is lenient; we're not.

### Unit tests (Jest)
- [ ] `ChatSection` rendering test: mounts with a templated hook state, asserts the chat area seeds with `initialText` or acknowledges `templateId`.
- [ ] `IntentSection` rendering test: mounted with `defaultText='hello'` and `defaultTemplateId='horror-series'`, both appear in the rendered form.
- [ ] `QuestionSection` rendering test: mounted with `onSkip`, skip button appears and calls the handler on click.
- [ ] `QuestionSection` rendering test: mounted without `onSkip`, skip button does NOT appear.
- [ ] `getSkipUrl` test: exercises whatever input shape the investigation settled on.

### Playwright E2E (`beta.kitesforu.com` after deploy)
For each of the 8 templated Create dropdown entries:
- [ ] Open Create dropdown → click template entry → verify URL carries `?template=<slug>` → verify the landing create page shows the template context (either chat mode seeds or batch mode hydrates) → verify no runtime error in console.

For clarifying questions:
- [ ] Submit a raw prompt that triggers question mode → click Skip → verify navigation happens (not a dead button) → verify we land somewhere sane, not a 404.

Record each Playwright result in the PR description. Screenshot the 8 template landings + the skip path to `.vision_vault/fix-a/`.

### Regression checks
- [ ] Home page quick-chips still work (`HeroSection.tsx` chips link into create flow; make sure none of them break).
- [ ] Smart Create mode toggle (`Chat` ↔ `Batch`) still works — `handleModeSwitch` in `create-smart/page.tsx:70-79` resets both hooks; confirm nothing regresses.
- [ ] No regression on the non-templated entry (`/create-smart` with no query params) — must still land on the blank chat intake.

---

## Rollout

Per shipping-playbook memory, this is one atomic unit. For a pure bug fix:

1. **Code PR** — `kitesforu-frontend` branch `fix/a-creation-prop-alignment`. Single focused PR.
2. **Docs PR** — not required. No user-visible UI change. (If the skip button is genuinely new, add one line to the Help Center's creation-flow section.)
3. **Tooltip flag PR** — not required.

Merge → verify Cloud Run revision → Playwright → move to Fix B.

---

## Alternatives considered

1. **"Just delete the dead props and call it done."** Rejected — that regresses the templated URL feature. The Navbar Create dropdown depends on template entries hydrating correctly; silently removing the prop plumbing without fixing it ships a different bug.

2. **Refactor the entire create-smart page to take a single `CreateInitialState` object instead of six separate props.** Rejected — out of scope. This is a polish opportunity, not a bug fix. File a separate `refactor/create-smart-state-container` issue and defer.

3. **Flip `tsc` to hard gate in the same PR.** Rejected — scope explosion. Separate issue, separate PR.

4. **Switch `ChatSection` to Option B (accept URL props directly).** Viable alternative; implementer's call. Option A (hydrate through the hook) is cleaner but requires exposing one more hook method. If the hook turns out to be hard to extend, fall back to B. Document which path the PR takes.

---

## Risks

1. **Runtime crashes already in production.** `ChatSection` receives `plan={undefined}` today and reads `plan.*` somewhere in its body. Either this path is never hit (no templated URL triggers chat mode with a plan render), or it crashes silently and users land in an error boundary. Before implementing, do a quick `console.log`/`console.error` check of production for "`Cannot read properties of undefined`" errors on `/create-smart`. If crashes are happening, flip this to **P0** and escalate the ship.

2. **`getSkipUrl` may be dead code.** If it hasn't been called with the right type in months, the skip path may have been silently broken since inception. The fix might be "remove `getSkipUrl`, inline the navigation" — simpler than expected. Verify via `git log -p components/smart-create/utils.ts` during implementation.

3. **Option A (hook hydration) vs Option B (prop extension) decision is irreversible without noise.** Pick once. If in doubt, default to Option B (smaller blast radius, same behavior).

4. **Type-check in CI is lenient.** This PR must land with zero local type errors even though CI won't block on them. If we ship type errors under the cover of lenient CI, the next PR will inherit the debt and we'll pretend the fix was complete.

5. **Playwright coverage gap.** I have no evidence there's an existing E2E test for the create-smart flow. If there isn't, this PR creates a new test file. Confirm during implementation.

---

## Open questions for audit

1. **Option A vs Option B** for `ChatSection` — does the reviewer have a preference? Default Option A, fall back B.
2. **CI type-check hardening** — file separate issue now or defer indefinitely?
3. **Does `useSmartCreateChat` already expose `plan` / `onUpdatePlan` / `onRetry`?** Implementer confirms; if not, hook extension is part of this PR's scope.
4. **Is there a production error signal** indicating `ChatSection`'s missing `plan` prop is crashing real users? If so, flip priority and skip the audit gate with explicit reviewer approval.

---

## Triangulation

**STOP here.** This proposal does not authorize any code changes. Awaiting audit.

Next in the Wave 1 sequence after Fix A merges: **Fix B (S1 — voice replay card reachability)**, proposal to be drafted after this one ships.
