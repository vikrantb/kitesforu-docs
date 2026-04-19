# Bugs

Live-observed production bugs with enough analysis to fix.

**Filename convention**: `{hex-unix-timestamp}-{slug}.md` where the hex prefix is `printf '%X' $(date +%s)` at report time. Gives each bug a unique sortable id without inventing a numbering scheme.

## Status key
- `OPEN` — reproduced, analyzed, fix pending
- `FIX_IN_FLIGHT` — PR opened, not yet merged
- `FIXED` — shipped to beta and verified
- `WONT_FIX` — analyzed and intentionally deferred (include reason)
