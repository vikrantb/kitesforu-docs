# P1 — Latency & Cold Start Optimization

**Status**: PROPOSED
**Priority**: P1
**Affected repos**: kitesforu-workers, kitesforu-infrastructure
**Related**: future/P1-cold-start-latency.md (earlier doc)

## Problem

Cloud Run cold starts add 5-15s to the first request after scale-to-zero. For a voice-first platform where the promise is "talk and get immediate feedback," cold start latency is directly hostile to the UX.

## Current state

- 11 Cloud Run services deployed
- Minimum instances = 0 (scale to zero)
- First request after idle hits a cold start
- Pipeline total: 53s average (down from 211s after Intelligent Dynamic Pipeline)
- Cold start can add 5-15s on top of that

## Scope

### Infrastructure
- Set **minimum instances = 1** for the most latency-sensitive services (initiator, audio worker, script worker)
- Evaluate Cloud Run "startup CPU boost" feature
- Measure and document the actual cold start penalty per service

### Workers
- **Preload models on startup** — move heavy imports (TTS clients, LLM clients) to module level
- **Connection pooling** — keep Firestore and Pub/Sub connections warm
- **Reduce Docker image size** — audit dependencies, multi-stage builds where missing

### Frontend
- **Optimistic UI during cold start** — show "Setting up your session..." with a progress animation rather than a spinner
- **Prefetch API health** — ping the API on page load so the cold start happens before the user clicks "Create"

## Acceptance criteria

- [ ] Latency-sensitive services have min_instances >= 1
- [ ] Cold start penalty measured and documented per service
- [ ] Frontend shows meaningful progress during first-request latency
- [ ] API prefetch fires on relevant page loads

## Effort estimate

8-12 hours infrastructure + 4-6 hours frontend.
