# Homepage simplification brainstorm

> **Status: CLOSED 2026-04-24.** All three implementation PRs (iteration-5 PR #1 / #2 / #3) shipped live to prod. Both feature flags (`feature_home_signed_out_v2` + `feature_home_signed_in_v2`) were flipped ON and then removed; legacy components (`HomeDashboard`, `SignedOutHero` v1, `PersonaSection`, `TemplateChip`, `SignedInHome` wrapper) deleted; `CreatorStatsStrip` ported to `/dashboard`; `BottomNav` 4th slot swapped to Car Mode (signed-in) / Search (signed-out); `/?view=X` legacy redirects in place. See `iteration-5-implementation.md` for the full closing summary with per-PR tracing.

**Problem**: The KitesForU homepage tries to do too many things at once. New visitors don't know what the product is or what action they should take. The goal is to find a simpler, clearer shape.

**Not this**: a quick polish pass or CSS cleanup.
**This**: a serious product/UX rethink that converges on the most elegant path — beauty in simplicity.

## Method

1. **Stage 1 — divergent brainstorm** (4-5 iterations, broad → narrow)
2. **Stage 2 — converge + prioritize** (one honest recommendation)

Four deep-research agents produce grounded input before synthesis:

| Agent | Role | Output |
|---|---|---|
| 1 | current-homepage-friction audit | `agent-1-friction-audit.md` |
| 2 | onboarding / first-time-user patterns | `agent-2-onboarding-patterns.md` |
| 3 | information architecture + action hierarchy | `agent-3-information-architecture.md` |
| 4 | KitesForU-specific user intent + value | `agent-4-kitesforu-value.md` |

The agents ran in parallel (no cross-reading) to avoid convergence pressure before synthesis.

## Iteration sequence

- **iteration-0** — framing + evaluation criteria (this doc + the agent briefs)
- **iteration-1** — broad divergence, 6-8 homepage shapes, no judgment yet
- **iteration-2** — narrow to the 3-4 that survive first-order critique
- **iteration-3** — stress-test survivors against edge cases (signed-out vs signed-in, mobile vs desktop, first-time vs returning)
- **iteration-4** — pick the winner, sketch the actual layout, define what gets cut
- **iteration-5** — implementation-aware prioritization: what ships first, what waits

## Evaluation criteria

Each iteration judges ideas against these, in priority order:

1. **Can a first-time visitor answer "what is this and what am I supposed to do" in under 5 seconds?** If not, the idea fails regardless of other merits.
2. **Does the homepage have exactly ONE above-the-fold primary action?** Competing CTAs are the most common failure mode of the current page; a candidate that has 2+ co-equal actions is not simpler, just reshuffled.
3. **Does it honor the "if it's in the shell, don't put it on the homepage" rule?** Navbar + bottom-nav already carry navigation; duplicating them on the homepage is double-entry without information gain.
4. **Is the language honest?** Marketing copy that promises the product does N things when N > 2 is a lie that the product must pay for later. Pick the real wedge.
5. **Is the first flow legible without documentation?** A visitor should be able to complete their first meaningful action without reading a help doc or watching a demo video.
6. **Is it beautiful?** Beauty here means: few elements, each with a clear job, nothing present merely because someone's KPI depended on it.

## Anti-patterns to avoid

- 3+ above-the-fold CTAs
- "Pick your persona" selectors on the homepage (belongs in creation flows, not positioning)
- Feature-list heroes
- Testimonial walls before the visitor knows what the product is
- Duplicate entry points that already exist in the navbar

## Hard constraints

- Preserve existing architecture (the player, the persistent audio bridge, Clerk auth, the layout shell).
- Any recommendation must be implementable as an incremental PR, not a rewrite.
- Must not regress already-shipped features (speaker viz, cross-device listen-progress, scenario-guidance v2, library search).

## What success looks like

A single iteration-5 doc that names:
- The one homepage shape we're adopting
- What copy goes above the fold
- What UI blocks stay / move / get cut
- A 3-PR implementation sequence (code, not design theater)
- The metric we'll watch post-ship to confirm the simplification landed
