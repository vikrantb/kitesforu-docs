# Autonomous `/loop` Playbook

**Status**: Living document
**Owner**: whoever is running the next autonomous session
**Last updated**: 2026-04-18 (from session that produced 38 PRs + this doc)

---

## What this is

KitesForU uses the Claude Code `/loop` primitive to run autonomous shipping passes. A single prompt re-fires on a cadence; Claude responds each firing. Good loops ship real work unattended. Bad loops ship garbage at volume.

This doc is the bad-outcome-prevention layer. It's here because the 2026-04-17 session shipped 38 PRs with 14 still open, most Playwright-unverified, some stacking conflicts. That's a loop failing loudly.

---

## The prompt that fired 22+ times that session

```
Review your work, identify at least 5 issues or improvements,
and then proceed with the next step in the roadmap
(R1->R3 then Phase B audit). Use --ultrathink --seq.
```

It's open-ended by design. That's its superpower (Claude picks the tightest next slice) and its failure mode (Claude always picks *something*, even when the right answer is "stop and merge").

---

## Failure modes observed

| Mode | Symptom | Signal to catch it |
|---|---|---|
| **PR pile-up** | Open PR count climbs past 5 | Any loop iteration where the PR number increments but no PR merges |
| **Diminishing audits** | 5 "issues" list becomes 4th-order polish items | Issue list repeats types across iterations; no new architectural findings |
| **Unverified stacking** | PRs reference each other but no Playwright ran | Active Cloud Run rev doesn't match the latest merged commit |
| **Ghost-typed correction** | Fragmented messages interrupting the loop | User types short phrases like "think properly", "no hanky fixes", a bare digit rating |
| **Scope drift** | Started as R1-R3 roadmap, became "find another dark-polish file" | Recent PR titles are all `feat: dark polish on X` for successively less-used files |

---

## Hard rules for future autonomous runs

### Rule 1: Max 3 open PRs before forced pause

If `gh pr list --state=open --author=@me` returns more than 3 from the current session, the next loop iteration **must not ship a new PR**. Instead:
- List the open PRs with a coherence rating (A/B/C)
- Merge the ones that are green + coherent
- Report the wait, then stop until human unblocks

### Rule 2: No new PR until last one has a Cloud Run rev

Every `feat:` PR in this repo triggers a Cloud Run deploy. Don't queue a new one until the previous rev is live *and* smoke-tested (curl or Playwright). Rapid-fire merging serializes the deploy queue and makes rollbacks expensive.

### Rule 3: Playwright rejection means stop shipping the surface

If the user rejects `pnpm exec playwright test ...` (interrupts or says no), the feature being tested is **MANUAL-VERIFY**. Don't ship further code on that surface; it's code-reviewed-only until human verifies.

### Rule 4: A bare digit from the user is a quality rating, not input

`0` = "that last response was bad". `10` = "ship more of that". See `~/.claude/rules/velocity-preference.md`. Don't ask clarifying questions when you get a digit — *act* on the rating.

### Rule 5: Fragmented / ghost-typed messages are a signal to reason, not parse

The user may be typing via voice / ghost-typist / dictation. Messages arrive as word salads. Don't respond fragment-by-fragment — reason about the intent behind the cluster and propose ONE concrete action, not a menu.

### Rule 6: Each loop iteration must improve a verifiable metric

Good metrics: "tests pass on beta", "PR count decreases", "a previously-failing path now works". Bad metric: "3 more files got `dark:` classes". If the iteration can't name a verifiable improvement, skip code and do ops work (merge, verify, document).

### Rule 7: The memory file is part of the deliverable

Sessions that produce durable improvements must also update `human/operations/autonomous-sessions/session-YYYY-MM-DD-*.md` at close. Ephemeral chat state loses on context compaction; committed docs don't.

---

## Recommended loop harness

Next autonomous run should start from this checklist, not the open-ended prompt:

```
1. Read human/operations/autonomous-loop-playbook.md (this file)
2. Read the latest session log in human/operations/autonomous-sessions/
3. `gh pr list --state=open --author=@me` — if > 3, go to merge/verify, not new work
4. Pick the SINGLE smallest-scope next-step deliverable from proposed/
5. Before coding, state: verification metric you'll hit, rollback plan, expected open PR count
6. Ship. Run Playwright. Merge. Verify rev.
7. Update session log.
8. Only then: loop again.
```

---

## Session logs

Committed under `autonomous-sessions/`:
- `session-2026-04-17-r1-shipping.md` — first pass (R1 foundation, backfill, persistent player)
- `session-2026-04-17-r1-shipping-addendum.md` — full session log (38 PRs, meta-observation)

---

## Notes on the "ghost typist" workflow

The 2026-04-17 session user identified themselves as "Ghost Typist" — a voice/automation workflow where prompts are spoken or pasted in fragments. Characteristics:
- Messages may be typo-dense ("hanky" for "hacky", "adhic" for "adhoc")
- Same prompt re-fires many times (it's the `/loop` primitive, not the user repeating)
- Brief corrections arrive as terse ratings (`0`, `10`) or single-word nudges

**How to respond**: treat a fragment cluster as a single intent. Respond with action, not with a menu of options. One decisive move > three choices.
