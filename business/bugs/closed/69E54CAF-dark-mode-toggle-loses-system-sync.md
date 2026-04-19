# 69E54CAF — Dark-mode toggle loses OS sync permanently after first click

**Status**: FIXED
**Reported**: 2026-04-19 by product owner
**Priority**: P2 — degrades user experience but not blocking
**Surface**: Navbar theme toggle, all pages
**Fix PRs**: kitesforu-frontend #457 (3-state cycle: light → dark → system → light) — merged 2026-04-19
**Verification**: moved to `closed/` once beta confirms toggle cycles through an "auto" state and the OS `prefers-color-scheme` change is tracked live.

## Reported symptom

> The light mode/dark mode is not synced with the OS.

## Root cause

`hooks/useTheme.tsx:82` — the provider defaults to `mode='system'` on a fresh install and correctly listens to `prefers-color-scheme` changes (`:95-110`). But the toggle button is a two-state cycle:

```ts
const toggle = useCallback(() => {
  // Comment says: "Toggling cycles: current resolved -> its opposite -> 'system' on third click."
  setMode(resolved === 'dark' ? 'light' : 'dark')
}, [resolved, setMode])
```

The comment promises a 3-state cycle (light → dark → system → light). The code implements a 2-state cycle (light ↔ dark). Once the user clicks the toggle even once, `mode` is stamped to `'light'` or `'dark'` in localStorage and OS sync is lost forever — the `prefers-color-scheme` listener early-returns because `mode !== 'system'`.

## Proposed fix (≤3 file edits)

1. `hooks/useTheme.tsx:124-127` — make `toggle` actually a 3-state cycle matching the comment:
   - current resolved `light` → set mode `dark`
   - current mode `dark` → set mode `system`
   - current mode `system` → set mode `light`
2. `components/Navbar.tsx` (or wherever the toggle button lives) — update the `aria-label` + icon to reflect the 3rd state: `Auto (system)` with a half-moon / half-sun icon.
3. Optional: on first visit when no stored preference, make the toggle tooltip say "Auto — follows your OS" so the system default is discoverable.

## Risks + mitigations

- Users who already clicked the toggle have `'light'` or `'dark'` stamped in localStorage; they won't get OS sync back until they cycle through to `'system'`. Acceptable — the button now has a path back.
- Migration path: no action needed, localStorage retains the user's most recent choice.

## Test plan

- Fresh browser profile: load `/`, OS in dark mode → page renders dark. Switch OS to light → page flips to light live.
- Click toggle once → mode becomes `dark`, localStorage `kforu_theme='dark'`, icon shows sun.
- Click toggle again → mode becomes `system`, localStorage `'system'`, icon shows auto/half.
- OS switches light ↔ dark while in `system` mode → page tracks.

## Files referenced

- `kitesforu-frontend/hooks/useTheme.tsx`
- `kitesforu-frontend/components/Navbar.tsx` (toggle button host)
