# Feature Tracking — KitesForU

All features flow through this system:

```
business/features/proposed/   ← Design + scope + acceptance criteria
business/features/done/       ← Shipped, verified, deployed
```

## Workflow

1. **Propose**: write a markdown in `proposed/` with title, user story, scope, acceptance criteria, and affected repos.
2. **Implement**: reference the feature file in PRs. Update status field as work progresses.
3. **Ship**: once deployed + verified, move the file to `done/` with a completion date and deploy artifacts.

## Naming convention

```
{priority}-{slug}.md
```

Priority: P0 (ship-blocker), P1 (high), P2 (medium), P3 (low/nice-to-have)

## Current state (April 2026)

See `proposed/` for the active backlog and `done/` for shipped features.
