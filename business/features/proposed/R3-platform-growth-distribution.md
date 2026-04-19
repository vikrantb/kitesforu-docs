# R3 — Platform, Growth & Distribution

**Status**: PROPOSED
**Priority**: P2
**Effort**: 4 weeks
**Affected repos**: kitesforu-frontend, kitesforu-api, kitesforu-infrastructure
**Depends on**: R1 (UX), R2 (content quality)

---

## Why This Is Release 3

R1 fixed how users find and navigate. R2 fixed content quality. R3 fixes how the platform grows — distribution for creators, analytics for measurement, internationalization for global reach, and team features for B2B.

---

## Deliverable 1: Distribution for Solo Podcasters

**Persona**: Solo Podcaster (#16) — needs RSS to publish on Spotify/Apple Podcasts

### Features

1. **RSS Feed Generation**: For any multi-episode course/series, generate an RSS 2.0 feed with iTunes-compatible tags
2. **Episode export**: Download individual episodes as MP3 (already partially built — download button on detail page)
3. **Show notes**: Auto-generate text show notes alongside audio (summary, key points, timestamps)
4. **Podcast hosting page**: Public page at `/listen/{series-id}` with embedded player, episode list, RSS subscribe link

### API Changes

- `GET /v1/courses/{id}/rss` — returns RSS XML
- `GET /v1/courses/{id}/show-notes` — returns generated show notes
- Public access for RSS feed (no auth, but rate-limited)

### Acceptance Criteria
- [ ] RSS feed validates against Apple Podcasts validator
- [ ] Spotify can ingest the RSS feed
- [ ] Each episode has show notes (auto-generated)
- [ ] Public podcast page works without sign-in
- [ ] Creator can customize podcast title, description, artwork for the feed

---

## Deliverable 2: Sharing & Collaboration

### Shareable Links (all content types)

1. **Share button** visible on all detail page headers (currently hidden for most types)
2. Share modal with: Copy Link, Twitter, LinkedIn, WhatsApp, Email
3. Public view page at `/shared/{token}` — audio player, no account required
4. Share analytics: view count, play count (visible to creator)

### Classroom Sharing

- Shareable link for students (already exists for classes)
- Extension to courses: share a course with a colleague via link
- QR code generation for in-person sharing

### Acceptance Criteria
- [x] Share button visible on ALL content detail pages — frontend PR #425 (plus prior PRs): classes have full ShareContentModal + ShareButton; writeups have ShareButton; courses now have both (PR #425 wired the missing Share+Distribute trigger). Studio/podcast detail still uses generation-surface patterns; full-modal share is N/A because ShareContentModal doesn't support the podcast variant yet.
- [x] Public share page works without authentication — `/courses/shared/` and `/classes/join/[shareToken]` routes render without login
- [x] Share modal includes social platforms + copy link — `components/sharing/ShareModal.tsx` + `components/ShareContentModal.tsx` (distribution tab adds Apple/Spotify/YouTube/Amazon)
- [x] View/play counts tracked for shared content — frontend PR #426: `shared_content_view` fires on public share page load; `shared_content_play` fires on each episode play in a shared course. Events carry `share_token` + `content_id` so the owning user can be attributed without the viewer signing in.

---

## Deliverable 3: Analytics Instrumentation

**Problem**: We listed success metrics but can't measure them. No event tracking exists.

### Events to Track

| Event | Properties | Purpose |
|-------|-----------|---------|
| `content_created` | type, template, duration, content_purpose | Creation funnel |
| `content_played` | content_id, type, duration, completion_pct | Engagement |
| `content_completed` | content_id, listen_duration | Completion rate |
| `player_action` | action (play/pause/seek/speed), context | Player usage |
| `library_filter` | filter_type, sort | Discovery patterns |
| `search_query` | query, results_count | Search effectiveness |
| `share_action` | platform, content_type | Virality |
| `create_abandoned` | step, template | Funnel drop-off |
| `mock_completed` | score, questions_answered | Interview prep |

### Implementation

Use a lightweight event tracking hook:
```typescript
// hooks/useAnalytics.ts
function useAnalytics() {
  return {
    track: (event: string, props: Record<string, unknown>) => {
      // Send to analytics backend (BigQuery, Mixpanel, or custom)
    }
  }
}
```

Add `track()` calls at key interaction points across the app.

### Dashboard

Simple admin page at `/admin/analytics` showing:
- DAU/MAU ratio
- Content creation funnel (start → complete)
- Audio completion rates by type
- Most popular templates
- Search queries (to inform template creation)

### Acceptance Criteria
- [x] All key events tracked — frontend PRs #429–#435. Unified funnel includes `content_created`, `create_abandoned`, `content_completed`, `mock_completed`, `player_action`, `library_filter`, `library_search`, `shared_content_view`, `shared_content_play`, plus navigation events `template_selected` / `persona_selected`. All dispatched from `lib/analytics.ts`.
- [ ] Analytics dashboard shows daily metrics
- [x] Creation funnel visible (start → generated → played) — `create_abandoned` (drop-off at intent/plan/question), `content_created` (successful creation), and `content_completed` (natural playback end) together cover the funnel per content type.
- [x] Audio completion rate measurable by content type — `content_completed` carries `content_type`; `content_created` pairs with the same `content_type`, so completion rate is derivable per type.
- [x] No PII in event data — trackers in `lib/analytics.ts` carry only IDs, content types, share tokens, enum fields (persona, template, action), and numeric metrics. No email, name, or free-text user input beyond a truncated search query.

---

## Deliverable 4: Internationalization Foundation

**Persona**: Non-Native Speaker (#20) — the #2 business value persona

### UI Localization (Phase 1 — structure only)

1. Extract all UI strings into a messages file using `next-intl` or similar
2. Support English (default) + Spanish + Portuguese + Hindi + Japanese as initial set
3. Language selector in settings
4. UI renders in selected language

### Content Language Improvements

1. Audio content already supports 5+ languages
2. Add language badge to library cards
3. Filter library by language
4. Default content language from user profile

### Right-to-Left (RTL) Foundation

1. Add `dir="rtl"` support for Arabic/Hebrew (future)
2. Use logical CSS properties (`margin-inline-start` vs `margin-left`)
3. Not full RTL implementation — just the foundation for future

### Acceptance Criteria
- [ ] All UI strings externalized to messages files
- [ ] Language selector in user settings
- [x] Library shows language badges on cards — frontend PR #427: `AudioContentCard` renders a flag + language code pill for non-English items (English suppressed as the 90%+ default). `data-language` attribute also exposed on the card root for future filtering wiring.
- [x] Library filterable by language — frontend PR #428: `/library?lang=<bcp47>` adds a filter dropdown populated with the languages actually present on the user's items. Initial value comes from `?lang=`, so the filter is deep-linkable. Paired with PR #427's language badge pill.
- [ ] RTL CSS foundations in place (logical properties)

---

## Deliverable 5: Team Features Foundation (B2B)

**Persona**: Corporate Trainer (#10) — needs team dashboards and completion tracking

### Team Dashboard (basic)

1. Team owner can invite members via email
2. Team library: shared content visible to all team members
3. Completion tracking: who listened to what, when
4. Simple analytics: completion rate per module, per team member

### SCORM Export Improvements

1. SCORM 1.2 export for classes (already exists)
2. Extend to courses (multi-episode → multi-SCO package)
3. Include completion tracking data in SCORM output
4. Test with 3 major LMS platforms (Docebo, Moodle, Canvas)

### Acceptance Criteria
- [ ] Team owner can invite 5+ members
- [ ] Team members see shared content in their library
- [ ] Completion tracking visible to team owner
- [ ] SCORM export works for courses (not just classes)
- [ ] SCORM packages validate in at least 2 LMS platforms

---

## Existing Proposals: Status

| Existing Proposal | Status After R3 |
|-------------------|----------------|
| P1-speaker-visualization.md | Absorbed into R1 Phase 2 |
| P2-mastery-ribbon-transparency.md | Absorbed into R2 Phase 2 |
| P2-voice-architecture-consolidation.md | Absorbed into R2 Phase 1 |

All three existing proposals are now part of the phased plan. Their content is preserved and expanded in the phase documents.

---

## Release Roadmap Summary

| Release | Phase | Focus | Effort | Key Deliverables |
|---------|-------|-------|--------|-----------------|
| **R1** | Phase 1 | UX Foundation | 2 weeks | Content classification, unified library, simplified nav |
| **R1** | Phase 2 | Player & Consumption | 3 weeks | Persistent player, speaker viz, Car Mode/Q&A, dark mode |
| **R1** | Phase 3 | Creation & Home | 2 weeks | Create page, personalized home, classification improvement |
| **R2** | Phase 1 | Content Quality | 3 weeks | Voice consolidation, series memory, ratings, genre polish |
| **R2** | Phase 2 | Interview Polish | 2 weeks | Mastery ribbon, company intel, mock improvements, gap analysis |
| **R3** | Full | Platform & Growth | 4 weeks | Distribution, sharing, analytics, i18n, team features |

**Total**: ~16 weeks (4 months) for the complete transformation.

### What Ships When

- **Week 2**: Users can find all their content in one place (library) with simple navigation
- **Week 5**: Audio persists across pages, dark mode, sleep timer, offline
- **Week 7**: Beautiful create page, personalized home, smart classification
- **Week 10**: Better audio quality, series memory, content safety
- **Week 12**: Interview prep is deeply polished with company-specific intelligence
- **Week 16**: RSS feeds, sharing, analytics, i18n, team features

### Mobile-First Note

Every deliverable above must be designed mobile-first. The rule: design the phone experience first, then enhance for desktop. No feature ships without mobile testing on iOS Safari and Android Chrome. The Commuter, Bedtime Parent, and Fitness Consumer — 3 of the top 5 personas — use phones exclusively.
