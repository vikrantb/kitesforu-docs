# Bugs

Live-observed bugs tracked as markdown files — **not GitHub Issues** (GitHub Issues on a private repo burns seat minutes and fragments the knowledge base; markdown travels with the code and stays cheap).

## Folder layout

```
business/bugs/
├── README.md            ← you are here
├── open/                ← reported, analyzed, fix not yet shipped+verified
└── closed/              ← fix shipped AND verified on beta (or WONT_FIX with reason)
```

A bug lives in `open/` the moment it's reported with enough context to act on. It moves to `closed/` only after the fix is merged **and** the symptom is gone on beta.kitesforu.com. A fix that is merged but awaiting deploy/verify stays in `open/` — the status field tells you which phase it's in.

## Filename convention

```
{hex-unix-timestamp}-{kebab-slug}.md
```

Generate the prefix with `printf '%X' $(date +%s)` at report time — gives each bug a unique sortable id without a central numbering authority.

Example: `69E54B51-car-mode-audio-concurrency.md`

## Status key (inside the file frontmatter)

- `OPEN` — reproduced, analyzed, fix pending
- `FIX_IN_FLIGHT` — PR opened, not yet merged
- `DEPLOYED_NOT_VERIFIED` — merged + deployed, awaiting live beta confirmation (still in `open/`)
- `FIXED` — shipped to beta and verified (lives in `closed/`)
- `WONT_FIX` — analyzed and intentionally deferred (must include reason; lives in `closed/`)

## How to file a new bug

1. Reproduce the symptom (live on beta, in a dev run, in a test). Capture the verbatim user report if it came from the product owner.
2. Pick a hex timestamp: `printf '%X' $(date +%s)`.
3. Create `business/bugs/open/{hex}-{slug}.md` with sections:
   - **Reported symptom (verbatim)** — quote the original report, do not paraphrase
   - **Correct mental model** — the invariant the system should honor
   - **Root cause** — evidence-based, cite `file:line`
   - **Proposed change set** — bounded list, grouped by repo
   - **Risks + mitigations**
   - **Test plan**
   - **Files referenced**
4. Open a PR with the bug file under `open/`. A separate PR can start the fix.

A concrete example lives in the `closed/` folder — `69E54B51` is a good template.

## How to fix and close a bug

1. Open a fix PR in the relevant repo(s). Reference the bug id in the PR title: `fix(...): ... (bug 69E54B51)`.
2. Keep the bug file in `open/`. Update `Status` to `FIX_IN_FLIGHT`.
3. When all fix PRs merge and the change deploys, update `Status` to `DEPLOYED_NOT_VERIFIED`.
4. Verify on beta per the Test plan section.
5. Move the file: `git mv business/bugs/open/{hex}-{slug}.md business/bugs/closed/`. Update `Status: FIXED`. Include the **fix PR links** and a one-line **verification note** at the top of the file so future readers can trace what landed.

Do both moves (to `FIX_IN_FLIGHT`, then to `closed/`) as part of PRs — the bug folder is part of the source of truth, not a scratchpad.

## Why not GitHub Issues

- Private-repo Issues cost seat-minutes on multi-team setups.
- Markdown files live next to code, so grep works, git log works, and the fix PR can edit the bug file in the same commit.
- Closed bugs are pruned only in review — the `closed/` folder is a durable post-mortem archive.

If a bug needs external reporter visibility (customer support, compliance), link from the markdown to whatever external tracker is used. The markdown is still the canonical engineering record.
