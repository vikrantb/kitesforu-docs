# Local CI Replacement And GitHub CI Simplification

## Why this exists

GitHub Actions is currently carrying too much of the real quality burden for KitesForU. That is expensive, slow, and late in the feedback loop. The goal is to move the real safety work locally so the developer or Claude session catches problems before push, and GitHub CI becomes intentionally lightweight.

This is not just "run lint before pushing." The local process should do more than current GitHub CI in the places that matter:

- run immediately after edits, not minutes later in the cloud
- run a broader set of deterministic quality checks before commit/push
- protect branch discipline and file organization
- make it easy to reproduce the exact safety checks repo by repo
- leave GitHub CI as a small deploy/publish path instead of an expensive debugging environment

## What GitHub CI currently does

### Frontend

- `pnpm install --frozen-lockfile`
- `pnpm run lint`
- `pnpm run type-check`
- `pnpm run test`
- `pnpm run build`
- on `main` push: Docker build/push + Cloud Run deploy + health check

### API

- Python setup + Artifact Registry auth
- install `.[dev]`
- `ruff check src/` (currently non-blocking)
- `pytest tests/ -v --cov=src/api` (currently non-blocking)
- `mypy src/api --ignore-missing-imports` (currently non-blocking)
- on `main` push: Docker build/push + Cloud Run deploy + `/docs` verification

### Workers

- Python setup + Artifact Registry auth
- install `.[dev]`
- `ruff check src/`
- `pytest tests/ -v --cov=src/workers`
- `mypy src/workers --ignore-missing-imports` (non-blocking because of existing debt)
- on `main` push: Docker build/push + multi-service Cloud Run deploy + readiness checks

### Course Workers

- Python setup
- `ruff check src/` (non-blocking)
- `pytest tests/ -v --cov=src` (non-blocking)
- on `main` push: Docker build/push + multi-service Cloud Run deploy

### Schemas

- Python setup
- `pytest tests/ -v`
- `mypy src/kitesforu_schemas --ignore-missing-imports` (non-blocking)
- on `main` push: package build + version check + publish to Artifact Registry

### Infrastructure

- `terraform fmt -check -recursive`
- `terraform init -backend=false`
- `terraform validate`
- `terraform plan`
- on `main` push: `terraform apply -auto-approve`

## The replacement model

### Layer 1: Post-edit hooks

Run immediately after Claude edits a file.

Purpose:

- auto-format where safe
- run targeted lint or syntax checks on the edited file
- catch local mistakes while the edit context is still hot

Examples:

- frontend: lint the touched file, optionally run related Jest tests
- python repos: `ruff format`, `ruff check --fix`, `python -m py_compile`
- terraform: `terraform fmt` on touched files

### Layer 2: Stop hook / local CI runner

Run when Claude reaches a natural stop.

Purpose:

- execute a fast but meaningful repo-level quality gate
- simulate the important parts of CI locally before commit/push
- make Claude self-correct before code leaves the machine

Examples:

- frontend: lint + type-check + tests + build
- api/workers/schemas: lint + tests + compile/type checks
- infrastructure: fmt + validate

### Layer 3: Git hooks

Versioned git hooks should run even if a human developer, not Claude, is making the change.

Recommended split:

- `pre-commit`: fast guardrails
- `pre-push`: stronger repo-local CI run

This is the lowest-cost way to stop bad pushes before GitHub is involved.

## Desired repo-local process

### Frontend

Fast gate:

- `pnpm run lint`
- `pnpm run type-check`
- `pnpm run test`

Strong gate:

- all of the above
- `pnpm run build`

### API

Fast gate:

- changed-file `ruff check`
- changed-file `python -m py_compile`
- changed shell/json validation for hooks and Claude settings

Strong gate:

- all of the above
- `pytest tests/ -x -q` on the broad suite, with any pre-existing baseline failures explicitly carved out and tracked
- `mypy src/api --ignore-missing-imports`
- because the broad suite is still red today, `pre-push` should enforce the fast gate and leave `./scripts/local-ci.sh full` as an explicit audit command until the baseline debt is fixed

### Workers

Fast gate:

- changed-file `ruff check`
- changed-file `python -m py_compile`
- changed shell/json validation for hooks and Claude settings

Strong gate:

- all of the above
- `pytest tests/ -x -q`
- `mypy src/workers --ignore-missing-imports` as a warning gate until type debt is burned down

### Course Workers

Fast gate:

- changed-file `ruff check`
- changed-file `python -m py_compile`
- changed shell/json validation for hooks and Claude settings

Strong gate:

- all of the above
- `pytest tests/ -x -q` on the broad suite, with any environment-bound failures explicitly carved out and tracked
- `python -m compileall src`
- because the broad suite is still red today, `pre-push` should enforce the fast gate and leave `./scripts/local-ci.sh full` as an explicit audit command until the baseline debt is fixed

### Schemas

Fast gate:

- `pytest tests/ -x -q`
- `python -m compileall src`

Strong gate:

- all of the above
- `mypy src/kitesforu_schemas --ignore-missing-imports`
- fail locally if `src/` changes without a version bump

### Infrastructure

Fast gate:

- `terraform fmt -check -recursive`
- `terraform init -backend=false`
- `terraform validate`

Strong gate:

- same as fast gate

## What GitHub CI should become

Once local CI and hooks are active, GitHub CI should become intentionally dumb:

- PR jobs should become lightweight acknowledgement jobs, not full validation jobs
- `main` push should still deploy/publish/apply
- heavy quality checks should be disabled in the workflow files, but preserved in comments or easy-to-reenable structure

Practical rule:

- local hooks and local CI own quality
- GitHub Actions own deployment/publish/apply

## Required safety additions beyond current CI

The local process should do more than current CI in these ways:

- block edits on `main` / `master`
- keep a stable repo-local command for "run what CI would have run here"
- catch changed-file issues immediately after edits
- keep repo-specific safety rules close to the repo
- make version bump rules local for publishable packages
- keep escape hatches explicit: `git commit --no-verify` or `git push --no-verify`

## Brainstorm / Claude follow-up still needed

This proposal is intentionally implementation-oriented, but it still needs a stronger second pass with deeper research and critique before the final rollout is declared complete.

That follow-up should answer:

- which local checks are worth running on every edit vs only on stop vs only on pre-push
- where local CI should be stricter than current GitHub CI
- how to avoid local hooks becoming so slow that people bypass them
- how to simplify frontend CI without stomping the active feature branch
- whether deploy verification should remain in GitHub or partially move local

## Baseline issues discovered during rollout

- `kitesforu-api`: the broad test suite currently has multiple red tests, including `tests/test_activity.py::TestFetchPodcasts::test_returns_items_with_inputs` (`/progress/pod1` vs `/studio/pod1`) and `tests/test_api_allow_premium.py::TestAPIAllowPremium::test_minimum_duration_validation`. The local full gate remains an audit command; `pre-push` enforces the fast deterministic gate until the suite is repaired.
- `kitesforu-course-workers`: the broad suite currently has multiple red tests, including the OpenAI-key path in `tests/integration/test_course_pipeline.py::TestCoursePipelineIntegration::test_syllabus_generates_curriculum` and an expectation mismatch in `tests/unit/test_class_workers.py::TestClassInitiatorWorker::test_validate_input_missing_grade_level`. The local full gate remains an audit command; `pre-push` enforces the fast deterministic gate until the suite is repaired.
- `kitesforu-workers`, `kitesforu-schemas`, and `kitesforu-infrastructure`: the current local full gate ran successfully in this rollout.

These carve-outs are not meant to become permanent. They exist so the local-CI replacement can go live without pretending the baseline is cleaner than it is.

## Immediate implementation direction

1. Add a reusable local CI runner and hook scripts
2. Add versioned git hooks per repo
3. Wire Claude hooks so the same guardrails apply during autonomous coding
4. Simplify GitHub CI in the clean repos first
5. Let the active frontend worktree finish, then apply the same pattern there
