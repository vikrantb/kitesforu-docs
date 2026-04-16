# P3 — Persona Settings Page IA Surfacing

**Status**: PROPOSED
**Priority**: P3
**Affected repos**: kitesforu-frontend
**Maps to**: UI Excellence Sweep S17

## Problem

The persona/settings page at /settings/persona exists and works — switching personas reorders the Navbar Create dropdown. But the page has no link from anywhere: not in the UserButton menu, not in the Navbar, not in the footer, not in the home page. Users who picked the wrong persona during onboarding have no way to change it.

## Scope

- Add a "Change your role" link to the Clerk UserButton dropdown (if Clerk v4 supports custom menu items)
- OR add a link in the footer under "Product"
- OR add a link in the Activity page sidebar

## Acceptance criteria

- [ ] /settings/persona is reachable from at least one navigation surface
- [ ] Users can switch personas and see the Navbar reorder immediately

## Effort estimate

2 hours, 1 PR.
