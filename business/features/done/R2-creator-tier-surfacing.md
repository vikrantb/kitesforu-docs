# R2 — Creator Tier Surfacing (A/B Audio Samples + Tier Toggle)

**Status**: SHIPPED SCAFFOLD (2026-04-20) — code in place, manifest upload is the remaining human-approved step. See Implementation Summary at bottom.
**Priority**: P2
**Effort**: ~1 week (sample generation + PlanSection UI + pricing wiring)
**Origin**: 2026-04-19 strategy pass — option #7 (make the quality tier legible before purchase)

## Problem

The voice persona system ships four tiers with 30× cost spread per episode:

- Inworld ($0.15/ep)
- Google ($0.24/ep)
- OpenAI ($0.18/ep)
- ElevenLabs ($4.50/ep)

Creators pay for premium voices on trust — they cannot hear the difference before committing a credit. Pricing copy talks in adjectives ("studio quality", "premium"), not sound. On-platform conversion to paid plans is limited by this: the listener has to imagine what they are missing.

Shipping a tier comparison with real audio samples directly inside the creation surface converts imagination into an A/B test, on demand, at the moment of intent.

## Proposal

Pre-generate a small library of 6–10 second persona clips across the four provider tiers using identical scripts — one tight sample per persona-archetype × tier combination. Render a "Hear the difference" card inside `PlanSection` (and the pricing page) where the creator can toggle between tiers and hear the same line delivered at each quality level.

## Why this option

- **Converts a purchase on trust into a purchase on evidence.**
- **Works both directions**: reinforces free tier value AND makes premium upside hearable.
- **Low incremental cost**: samples are pre-generated once, then served statically from Cloud Storage. Zero per-session TTS cost.
- **Survives locale**: generate the same set in every supported language to keep the non-English experience first-class.

## Sample catalog (v1)

Four archetypes × four tiers = 16 samples per language. Script chosen to exercise the areas of biggest differentiation: prosody on a statement, laugh or emphasis on a reaction, and a soft delivery on an introspective line. Same script across all tiers so the delta is attributable to the voice only.

Suggested archetypes:
- Warm narrator (documentary / storytelling)
- Energetic host (news / explainer)
- Intimate reader (bedtime / meditation)
- Comedic duo (dialogue format)

## Implementation

### Sample generation pipeline (one-time job)
- `kitesforu-workers/scripts/generate_tier_samples.py` — orchestrates per-archetype × per-tier generation, writes MP3 + metadata JSON to `gs://kitesforu-public-assets/tier-samples/{lang}/{archetype}/{tier}.mp3`
- Metadata: `{ archetype, tier, persona_id, voice_id, lang, duration_ms, script, generated_at }`
- Rerun triggered when a tier swaps out a persona (rare) — CI job that verifies all samples still exist.

### API — public manifest
`GET /v1/tier-samples?lang=en` → list of archetype + tier + audio URL + metadata. Public (no auth) so the signed-out pricing page can play it.

### Frontend — new component
`kitesforu-frontend/components/audio/TierSampleCard.tsx`:
- Four-tab row (Free / Standard / Pro / Studio) matching the pricing tiers
- Inline `<audio>` element
- Single archetype toggle at the top of the card (warm narrator default; dropdown for the other three)
- Switching tiers pauses the current sample and primes the new one — same archetype, different voice — so the A/B is hearable in one tap

Wire into:
- `app/create-smart/_components/PlanSection.tsx` — below the plan preview, above the big "Generate" button
- `app/pricing/page.tsx` — in the tier comparison table as the signature row

## Why pre-generate (not generate on the fly)

- Per-session TTS costs for a sample are 100% waste (no creator content is produced).
- Latency: pre-generated samples play instantly. Live TTS would add 2–6s spinner per tier toggle.
- Consistency: every creator hears the exact same reference clip, so the tier comparison is apples-to-apples.
- Cache-friendly: Cloud Storage + CDN edges make this essentially free at scale.

## Guardrails

- Sample clips must be labelled clearly ("Sample · Warm narrator archetype") so a listener cannot mistake them for personal output.
- Never use a creator's uploaded content as sample seed — use fixed public-domain copy.
- Every sample metadata JSON captures `persona_id` + `voice_id` — if a persona is deprecated the regeneration job must swap to the current one.

## Acceptance criteria

- [ ] Sample generation script produces 16 samples per supported language with metadata JSON
- [ ] Public assets bucket hosts MP3s + manifest at stable URLs
- [ ] `GET /v1/tier-samples?lang=` returns manifest; caching headers set for 24h
- [ ] `TierSampleCard` renders in `PlanSection` and on `/pricing`
- [ ] Switching tiers pauses any playing sample, loads new one, keeps archetype constant
- [ ] Accessible: keyboard nav between tier tabs, captions available for each sample, `aria-label` on audio element
- [ ] Playwright test: click each tier tab in turn, assert MP3 src changes and play button is pressable
- [ ] Beta-verify: pricing page and PlanSection both play a tier switch in a test browser

## Out of scope

- User-personalised samples (generated from their own draft) — much higher cost, defer
- More than four archetypes (keeps the catalog small and manageable for v1)
- Language auto-switch (v1 serves the session's UI language; user can toggle)
- Mid-generation tier swap (a creator choosing Pro during generation) — separate pricing-flow proposal

## Key file references

- Tier map + pricing: `kitesforu-frontend/lib/pricing.ts` (existing tier definitions)
- Plan preview: `kitesforu-frontend/app/create-smart/_components/PlanSection.tsx`
- Persona catalog: `kitesforu-workers/personas/` (YAML source of truth)
- TTS providers: `kitesforu-workers/src/workers/stages/audio/providers/`

---

## Implementation Summary (2026-04-20) — scaffold shipped across 3 repos, manifest upload pending

**API — kitesforu-api PR #245 (MERGED)**: `GET /v1/tier-samples?lang=` public endpoint. Reads `manifest.json` from the public GCS bucket (`KITESFORU_AUDIO_ASSETS_BUCKET`, defaults to `kitesforu-dev-audio-assets`) under `tier-samples/{lang}/`. Returns `{samples: [], lang, manifest_version}`. Graceful-null contract: a missing blob, GCS exception, or JSON error all return an empty list with 200 — the public signed-out pricing page can never 500 because of this aux asset. `Cache-Control: public, max-age=86400`. Per-process in-memory manifest cache per lang. 6 unit tests pinning the never-500 / cache / happy-path / skip-malformed contract.

**Frontend — kitesforu-frontend PR #475 (MERGED)**: `components/audio/TierSampleCard.tsx`. Fetches the API once on mount. Groups samples by `archetype` → archetype dropdown + tier tabs (stable order: free / creator / pro / business; missing tiers skip without shifting siblings). Inline `<audio preload="none">` with explicit pause + reset on tier switch so clips never overlap. **Ship-safe empty state**: returns `null` while loading, on fetch error, and when the manifest is empty — lets both mount surfaces (`PlanSection`, `/pricing`) render day one before any MP3 exists; the card lights up automatically once the manifest is uploaded. Dark-mode aware; labelled heading + `role=tablist` + aria-label on play button + "Sample" copy so clips cannot be mistaken for personal output. 4 Jest tests pin the contract.

**Workers — kitesforu-workers PR #297 (MERGED)**: `scripts/generate_tier_samples.py`. Produces 4 archetypes × 4 providers = 16 MP3s + `manifest.json`. Same script across tiers inside an archetype so the audible delta is attributable to voice quality alone. Fixed public-domain-safe copy (creator content never sampled). Uploads to `gs://${bucket}/tier-samples/{lang}/{archetype}/{provider}.mp3` + `.../manifest.json`. Guardrails: `--dry-run` (verified locally, prints all 16 rows), `--archetype` / `--tier` for targeted regeneration, manifest write suppressed if zero samples succeeded (never clobbers a healthy catalog), per-provider errors logged and skipped.

**Remaining human-approved step**: run `python scripts/generate_tier_samples.py --lang en-US` in an environment with OPENAI_API_KEY + GOOGLE_APPLICATION_CREDENTIALS + INWORLD_API_KEY + ELEVENLABS_API_KEY configured. Cost ~$0.64 per language. Once the manifest lands in GCS, the frontend card lights up on `/pricing` and every `PlanSection` — no additional deploy needed.

**Deviations from the proposal**:
- Tier labels match current pricing tiers (Free / Creator / Pro / Business) rather than the proposal's older Free/Standard/Pro/Studio naming — the frontend component lives on the live pricing page, so it uses that taxonomy.
- Spec acceptance-criterion "Playwright test: click each tier tab in turn, assert MP3 src changes" — deferred to post-manifest-upload beta smoke (the Jest tests pin the rendering contract synthetically; the live Playwright run needs real MP3 URLs to assert).
- Language auto-switch (AC out of scope) — the component accepts a `lang` prop defaulting to `en-US`; wiring it to the app's i18n context is a future follow-up once additional-language manifests exist.

**Deferred follow-ups**:
- Run the script in production environment to populate `en-US` manifest; subsequent language runs as we add locales.
- Playwright E2E on beta post-upload.
- Analytics hook points on surface click-through (stubbed as `data-surface="plan-section|pricing"` on the root div; instrument with the app's analytics helper once usage is worth measuring).
