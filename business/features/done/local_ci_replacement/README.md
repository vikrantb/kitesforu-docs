# Local CI Replacement — Trust Local, Ship Fast

**Status**: SHIPPED (Phase 1-3 merged across 6 repos 2026-04-22/23) — Phase 4 (AST-invariant hooks) deferred
**Priority**: P0 (infra)
**Affected repos**: all 6 (kitesforu-frontend, kitesforu-api, kitesforu-workers, kitesforu-course-workers, kitesforu-schemas, kitesforu-infrastructure)
**Owner directive**: "local CI should cover everything GitHub CI covers and do more where it matters… once the local path is strong enough, GitHub CI should become lightweight and mostly just push to prod/publish/apply."
**Grounding**: 4 deep-research passes (GitHub-Actions coverage audit / Claude-hooks strategy / git-hooks workflow / CI simplification). This doc is the synthesis, not a stitch.

---

## 1. One-paragraph thesis

Three of six repos (course-workers, schemas, infrastructure) already use a `LOCAL_CI_OWNS_QUALITY_GATES='true'` env flag with per-step `if:` guards — the pattern exists and works. The three repos still paying full CI minutes on every PR — frontend, api, workers — consume ~28-38 min per PR combined and catch problems locally solvable. This proposal extends the proven pattern to those three, introduces a structured Claude-hooks + git-hooks quality gate on the dev machine, and narrows GitHub Actions to one load-bearing job per repo (does it build? does it deploy?) with a one-flag rollback to the full suite. Saves ~18.7 CI-minutes per PR across repos; shrinks PR feedback from minutes to near-instant; catches the architectural-invariant bugs (Pydantic/Firestore, Literal/cast, duration-int) that no CI runs today.

## 2. State today (honest audit)

From the coverage-audit pass:

| Repo | Status | PR-path CI-min | Cut candidate? |
|---|---|---|---|
| kitesforu-frontend | Full gate (lint + tc + test + build) | 5-9 | **Yes** |
| kitesforu-api | Full gate (ruff + pytest + mypy + dep-sync) | 5-8 | **Yes** (keep dep-sync) |
| kitesforu-workers | Full gate (ruff + pytest + mypy) | 6-9 | **Yes** |
| kitesforu-course-workers | Already local-first (`LOCAL_CI_OWNS_QUALITY_GATES`) | ~0 | Already done |
| kitesforu-schemas | Already local-first | ~0 | Already done |
| kitesforu-infrastructure | Already local-first | ~0 | Already done |

Genuinely CI-locked steps (cannot move): all Artifact Registry Docker pushes, all `gcloud run services update` calls, parallel multi-service deploys (workers 11 / course-workers 8), `twine upload` for schemas, WIF-based `terraform apply`. These stay untouched.

## 3. Three-layer local quality gate

### Layer 1 — Claude hooks (`.claude/hooks/`)

Self-correcting loop that fires at edit time and at `Stop`. Not "run pytest on save" — AST-enforced architectural invariants plus file-scoped type/lint.

**Hook philosophy (5 principles)**:
1. Fast feedback beats thorough feedback at edit time. `<3s` post-edit ceiling; anything slower trains Claude to batch edits and lose the self-correction window.
2. Hooks write **structured output back into Claude's context**, not just stdout. Every failure emits file + line + rule + suggested-fix in a parseable block.
3. Auto-fix is a privilege, not a default. Every mutation surfaces the diff. Silent rewrites hide bugs (import reordering that masks circular deps; prettier collapsing comment-bearing multi-line conditionals).
4. Architectural invariants are **ast-grep / libcst patterns**, not docs. `ConfigDict(extra='ignore')` on Firestore read models, `cast()` before `SupportedLanguage`, no `max_length` on `List[str]` for LLM content, Dockerfile-COPY for new top-level dirs — all enforceable at edit time.
5. Stop-hooks are a contract with a **two-iteration ceiling**. If the hook fails twice, Claude halts and reports — never loops indefinitely.

**Priority hooks (ship first)** — the ones that earn their keep on day one:

| Hook | Trigger | Cost | Why |
|---|---|---|---|
| `post-edit-py-ruff` | PostToolUse on `*.py` | ~200ms | Auto-fix lint + format; `--exit-non-zero-on-fix` surfaces diff |
| `post-edit-py-pyright-file` | PostToolUse on `*.py` | ~1s | File-scoped type-check, not full repo |
| `post-edit-ts-tsc-incremental` | PostToolUse on `*.ts` `*.tsx` | ~800ms | `--incremental` with persistent `tsBuildInfoFile` |
| `post-edit-ts-eslint-file` | PostToolUse on `*.ts` `*.tsx` | ~300ms | `--fix --cache`, structured JSON output |
| `post-edit-pydantic-firestore` | PostToolUse on `**/models/*.py` | ~50ms | AST scan for `BaseModel` without `ConfigDict(extra='ignore')` in the same file. Prevented three prior outages per MEMORY.md "Pydantic kills reads". |
| `post-edit-npm-forbidden` | PostToolUse on `package.json` / `package-lock.json` | ~10ms | Blocks `package-lock.json`. Docker uses `pnpm install --frozen-lockfile`. |
| `post-edit-dockerfile-copy` | PostToolUse when new top-level dir appears | ~50ms | Catches the persona-bug pattern: new dir at repo root without matching `COPY` in Dockerfile silently fails to ship. |
| `stop-changed-typecheck` | Stop | scope-dep | `tsc`/`pyright` only on `git diff --name-only $(git merge-base HEAD origin/main)` |
| `stop-changed-tests` | Stop | scope-dep | `jest --findRelatedTests` or `pytest --testmon` — tests whose dependencies changed |
| `session-start-git-status` | SessionStart | ~100ms | Shows WIP at session open; prevents the branch-cut-stomps-your-WIP pattern |

**Backlog hooks** (ship after priority hooks stabilize): suppression drift, TODO drift, test-pair check, coverage delta, cross-file schema-consumer grep. These accumulate warnings, don't block.

**Anti-patterns** (explicit no):
- Full-suite pytest/jest in a PostToolUse hook. Batches kill the loop.
- Auto-fix hooks that mutate silently.
- Hooks that block on warnings.
- Stop-hooks that never let Claude stop (`any` flagging on legacy code → infinite correction).

### Layer 2 — Git hooks (`.husky/` for TS, `.pre-commit-config.yaml` for Python)

Pre-commit: `<2s` typical, `<5s` p99. Pre-push: `<60s` typical, `<120s` p99.

**Pre-commit spec** — staged-files only, no test runs, auto-fix + re-stage:

```yaml
# Python repos (.pre-commit-config.yaml)
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.2
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
  - repo: local
    hooks:
      - id: pydantic-firestore-check
        entry: scripts/ci/check-pydantic-configs.sh
        language: script
        files: ^src/.*\.py$
```

```json
// Frontend (package.json lint-staged config)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix --cache --max-warnings=0", "prettier --write"],
    "*.{md,json,yml}": ["prettier --write"]
  }
}
```

No tsc in pre-commit (too slow on staged-only). Deferred to pre-push.

**Pre-push spec** — changed-since-merge-base scope, heavier checks:

```bash
# scripts/ci/pre-push-frontend.sh
#!/usr/bin/env bash
set -euo pipefail
CHANGED=$(git diff --name-only "$(git merge-base HEAD origin/main)"...HEAD)
if echo "$CHANGED" | grep -qE '\.(ts|tsx)$'; then
  pnpm run type-check    # tsc --noEmit, incremental
  pnpm run lint          # full eslint with cache
fi
if echo "$CHANGED" | grep -qE '^(package\.json|pnpm-lock\.yaml)$'; then
  pnpm install --frozen-lockfile --prefer-offline
fi
```

```bash
# scripts/ci/pre-push-python.sh
#!/usr/bin/env bash
set -euo pipefail
CHANGED=$(git diff --name-only "$(git merge-base HEAD origin/main)"...HEAD | grep -E '\.py$' || true)
[ -z "$CHANGED" ] && exit 0
ruff check .
mypy --install-types --non-interactive src/
pytest --testmon -x -q tests/ || pytest -x -q tests/unit/
```

**`pnpm build` is intentionally NOT in pre-push.** Next.js build is 60-120s and GitHub runs it anyway as the load-bearing safety-net. `tsc --noEmit` catches ~95% of build failures at 20s.

**Tool choice, decided**:
- kitesforu-frontend → **husky + lint-staged** (already in node ecosystem)
- kitesforu-api / workers / course-workers / schemas → **`pre-commit` (Python framework)** (repos already have Python; framework gives staged file scoping for free)
- kitesforu-infrastructure → **`pre-commit` (Python framework)** with `pre-commit-terraform` (community standard)

**Rejected**:
- Raw `.git/hooks/` — doesn't sync across machines; new hires miss them.
- husky in Python repos — forces pnpm into a pure-Python toolchain; adds node to Cloud Run base-image risk surface.
- Shared hook logic across repos via submodule or tarball — version skew, debugging hell. **Duplicate 30 lines; move on.**

### Layer 3 — GitHub Actions (lightweight)

Every repo gets the `LOCAL_CI_OWNS_QUALITY_GATES='true'` pattern already proven in 3 repos, PLUS the cut jobs preserved behind a `quality_gates_rollback` gate. Re-arming the full suite is one env flip or one `workflow_dispatch` click.

**What stays**:
- `docker build` + `docker/build-push-action@v6` + registry cache + `gcloud run services update`.
- Parallel multi-service deploys (workers 11, course-workers 8) — these are real CI value.
- WIF-based `terraform apply` on main.
- `twine upload` for schemas + smart version-check.
- `dep_sync` for api (catches the PR #265/#266 `requirements.txt ↔ pyproject.toml` drift class).
- Curl verify after deploy.

**What leaves the happy path**:
- `pnpm lint` / `ruff check` (soft gates were noise; dev runs locally)
- `pnpm type-check` / `mypy` (local pre-push covers; full run preserved in rollback job)
- `pnpm test` / `pytest` (same)
- `pnpm build` on PR (build happens on merge; local `tsc --noEmit` is ~95% coverage)

**One mandatory GitHub-side check stays: the build step itself.** If `docker build` or `python -m build` or `terraform validate` won't run on clean ubuntu-latest, nothing else matters. No attestation file, no signed-SHA theater — trust the dev, rely on the build as the final safety net.

## 4. Critique of the agent pass (what I'm sharpening)

The 4-agent pass was strong but has tensions to resolve:

1. **Agent 2 proposed 20 hooks; Agent 3 demanded `<5s` total pre-commit.** Resolution: ship Agent 2's **top 10 priority hooks** in Layer 1 (fired at edit time by Claude, not pre-commit) and **3 checks** in Layer 2 (pre-commit — ruff + prettier + pydantic-firestore script). Claude hooks can spend more compute because they run during edit, not per-keystroke.

2. **Agent 4 left the api `dep_sync` job in.** Correct. The `check_deps_in_sync.py` script is stdlib-only, runs in <10s, and catches the class of bug (`requirements.txt` vs `pyproject.toml` drift) that crashed prod twice recently. Extend it to **workers and course-workers** too — trivial copy, closes the biggest post-cut residual risk.

3. **Agent 2 wants cross-repo schema-consumer grep.** Defer. Nice-to-have but a rabbit hole — requires careful scoping of what "consumer" means when kitesforu-schemas publishes Python AND the frontend has hand-mirrored TS types. Phase 2.

4. **Agent 3 said "Duplication > coupling for hook scripts."** Accepted. Every repo gets its own `scripts/ci/` directory with a ~30-line pre-push script. Shared framework (husky config, pre-commit versions) is fine; shared script bodies are not.

5. **Agent 4's "trust the dev" regression guard.** Accepted. No attestation file. If broken code lands, `workflow_dispatch.run_full_suite=true` on the fix PR surfaces what's red. Rollback is one click away.

6. **Playwright post-deploy verification (CLAUDE.md rule 11) stays mandatory.** Runtime features (Audio, SSE, SpeechSynthesis, MediaSession) can't be validated by tsc/jest regardless. The local CI shift doesn't touch this.

## 5. Acceptance criteria

- [ ] All 6 repos have a `LOCAL_CI_OWNS_QUALITY_GATES='true'` env flag and `workflow_dispatch.run_full_suite` input
- [ ] Removed PR-path quality-gate jobs are preserved behind `quality_gates_rollback` gate (commented re-enable recipe in body)
- [ ] kitesforu-frontend has `.husky/pre-commit` + `.husky/pre-push` + `lint-staged` config + `scripts/ci/pre-push-frontend.sh`
- [ ] All 4 Python repos have `.pre-commit-config.yaml` pinned to the matching ruff version + `scripts/ci/pre-push-python.sh`
- [ ] Every repo has `.claude/hooks/settings.json` wiring the 10 priority hooks at minimum
- [ ] AST-invariant hook scripts live in `.claude/scripts/` (Python) or `.claude/hooks/` (shell)
- [ ] api / workers / course-workers each have a `dep_sync` GitHub job (extend `check_deps_in_sync.py` to workers + course-workers)
- [ ] Frontend CI PR-job shrinks from ~5.5 min to <1 min (just dep-sync-equivalent if any)
- [ ] Pre-commit runs in <2s on a 10-file commit; pre-push in <60s on a 50-file diff
- [ ] A dev who clones the repo cleanly gets the hooks installed automatically (`pnpm install` triggers husky via `"prepare"`; `pre-commit install` step in README for Python repos)
- [ ] Playwright post-deploy verification (CLAUDE.md rule 11) is explicitly called out as unchanged
- [ ] **Rollback test**: flip `LOCAL_CI_OWNS_QUALITY_GATES='false'` on one repo, confirm full legacy suite runs on next PR

## 6. Phased shipping

**Phase 1 — frontend** (this PR sequence): implement husky + lint-staged + pre-push script + .claude/hooks/ priority set + simplified `.github/workflows/ci.yml`. Highest-leverage repo, highest CI-minute savings, zero risk to api/workers.

**Phase 2 — api + workers**: port the Python pre-commit + Claude hooks pattern. Extend `check_deps_in_sync.py` to workers + course-workers. Shrink PR-path CI jobs to dep_sync + rollback.

**Phase 3 — course-workers + schemas + infrastructure polish**: these are already partially done; add `workflow_dispatch.run_full_suite` input for parity and wire `.claude/hooks/` priority set. Small scope.

**Phase 4 — architectural-invariant hooks**: the high-leverage AST checks (pydantic-firestore, literal-cast, duration-float, dockerfile-copy-verify). These catch bugs that have each cost production outages per MEMORY.md. Ship after the Phase 1-3 plumbing is stable.

## 7. Risk / rollback

**Risk 1: Broken code lands on main because a dev bypassed local checks.**
- Mitigation: the `docker build` / `python -m build` / `terraform validate` step on every push is the real guard. Dep-sync catches import errors before deploy. Playwright post-deploy catches runtime breaks.
- Rollback: flip `LOCAL_CI_OWNS_QUALITY_GATES='false'` repo-by-repo to re-arm the full suite. One env var change.

**Risk 2: Hooks slow enough that devs start using `--no-verify`.**
- Mitigation: hard timing budgets in section 3 (<2s pre-commit, <60s pre-push). Budgets are enforced by the script — if a check exceeds budget, it moves to pre-push or CI, not the other way.
- Escape hatch: `SKIP=test-hook-name git commit` skips one hook without nuking all. Pressure-release valve that keeps devs from bypassing everything.

**Risk 3: Architectural invariant hooks have false positives on legitimate edge cases.**
- Mitigation: every invariant hook honors an explicit opt-out comment (e.g. `# noqa: firestore-configdict  # reason: write-only path`). The opt-out is the escape hatch; false-positive stories drive hook refinement, not disabling.

**Risk 4: Hook configs drift across repos.**
- Mitigation: accepted. Per Agent 3 — "duplicate 30 lines, move on." A shared library of hook scripts is more fragile than duplication.

## 8. What NOT to build

- **Attestation files / signed SHAs proving local CI ran.** Theater. Trust the dev + build-step safety net.
- **Shared cross-repo hook library.** Version skew nightmare. Duplicate.
- **Meta-hooks that run across repos.** Each repo owns its checks. If a schema change breaks api, that's api's CI problem.
- **Pre-commit hooks that run full test suites.** Devs bypass within a week.
- **`pnpm build` in pre-push.** CI's job. 90s for ~0 additional coverage over `tsc --noEmit`.
- **Schema-consumer cross-repo grep in Phase 1.** Rabbit hole (Python vs TS mirror). Defer to Phase 2 if at all.
- **Auto-fix hooks that mutate silently.** Always surface the diff; let Claude see what changed.
- **Hooks that block on warnings.** Warnings accumulate, don't block.

## 9. Ship log

- **2026-04-22 docs this proposal** — synthesis of 4-agent deep-research pass
- **2026-04-22 frontend Phase 1 SHIPPED** (frontend PR #581) — `.claude/settings.json` canonical hook path, husky + lint-staged, `scripts/ci/pre-push-frontend.sh` + `local-ci.sh` + `post-edit-check.sh`, `quality_gates_rollback` gate with `workflow_dispatch.run_full_suite` input. Live pre-push on the shipping commit itself ran in 1s.
- **2026-04-22/23 Phase 2 + 3 SHIPPED across 5 Python/Terraform repos** — api PR #282 (incl. dep_sync guard that stays CI-mandatory for `requirements.txt↔pyproject.toml` drift), workers PR #312, course-workers PR #59, schemas PR #78, infrastructure PR #40. Each repo has `.claude/settings.json` (PreToolUse branch guard, PostToolUse post-edit-check, Stop local-ci, SessionStart git-status), `.githooks/pre-commit` + `.githooks/pre-push`, `scripts/local-ci.sh` + `scripts/post-edit-check.sh` + `scripts/setup-local-ci.sh`. CI tests/lint/typecheck gated behind `LOCAL_CI_OWNS_QUALITY_GATES='true'`; build-and-deploy path preserved on main.
- *(next, deferred)* Phase 4 AST-invariant hooks — pydantic-firestore ConfigDict check, SupportedLanguage cast enforcement, duration-float check, Dockerfile-COPY verify for new top-level dirs. Ship after Phase 1-3 plumbing is observed stable in practice; the MEMORY.md outage classes these prevent are the real motivator.

## 10. Sources

Agent outputs (condensed in sections above):
- `github-actions-coverage-audit-2026-04-22.md` — per-repo inventory + classification tables
- `claude-hooks-strategy-2026-04-22.md` — 20-hook inventory + 4-state self-correcting loop protocol
- `git-hooks-workflow-2026-04-22.md` — timing budgets + pre-commit/pre-push specs per repo type
- `ci-simplification-2026-04-22.md` — per-repo simplified YAML + deploy-path preservation audit + failure-mode analysis

External grounding:
- MEMORY.md `pipeline_integration_lessons.md` — architectural invariants that have cost outages (Pydantic/Firestore, voice-selected-but-not-threaded, Dockerfile-COPY for new directories, duration-as-float)
- CLAUDE.md (per-repo) — pnpm-only rule, permissive Pydantic read models, SupportedLanguage cast requirement, Playwright post-deploy rule
- Existing `LOCAL_CI_OWNS_QUALITY_GATES` pattern in course-workers / schemas / infrastructure — the proven shape this proposal extends
