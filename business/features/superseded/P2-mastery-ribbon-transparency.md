# P2 — Mastery Ribbon Transparency

**Status**: PROPOSED
**Priority**: P2
**Affected repos**: kitesforu-frontend
**Maps to**: UI Excellence Sweep S2

## Problem

The Master State Machine's hidden tuple (pressure, support, mastery) drives narrative ribbon transitions (Teach → Practice → Stress-Test → Review), but users only see the label change — never why it changed. The `pendingLabel` field exists in state but isn't surfaced. Transitions feel arbitrary.

## User story

As a user in a mock interview, I want to understand why the system shifted from "Teach" to "Practice" mode — was it because I got three strong answers in a row? Because the Elo passed a threshold? The system should show me.

## Scope

- Surface `pendingLabel` as a preview chip on the ribbon: "Practice mode in 1 strong answer"
- Or show a micro-sparkline of the hidden tuple trajectory
- Or a simple tooltip on the ribbon: "Switched because your mastery reached 0.65"

## Design decision needed

The hidden tuple is explicitly designed as "hidden" — the design claim is "narrative labels are theater, hidden tuple is the product." Surfacing the tuple directly contradicts this philosophy. Options:
1. Surface the pending label (partial transparency without exposing the tuple)
2. Surface the tuple (full transparency, breaks the theater)
3. Show a qualitative explanation ("You've been strong on the last few answers")

Recommend option 1 or 3. Needs a design sync before implementation.

## Effort estimate

8-12 hours, 1-2 PRs (depends on design decision).
