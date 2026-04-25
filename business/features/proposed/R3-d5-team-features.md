# R3 D5 — Team Features (B2B)

**Status**: PROPOSED (awaiting Codex audit before any code per triangulation rule)
**Priority**: P2 (closes R3-platform-growth D5; the lone remaining R3 deliverable per partial-shipped status)
**Effort**: ~3 weeks (api + frontend + lms; phased)
**Affected repos**: `kitesforu-api` (team CRUD + Firestore model), `kitesforu-frontend` (team dashboard + invite flow + library scoping), `kitesforu-schemas` (Team / TeamMembership / Completion models), `kitesforu-lms` (SCORM-for-courses extension), `kitesforu-infrastructure` (Firestore security rules)
**Depends on**: R3 D2 sharing (shipped) — invite flow reuses `/classes/join/[shareToken]` patterns. R3 D3 analytics (shipped) — completion tracking surfaces existing `content_completed` events.
**Origin**: R3-platform-growth D5 ACs explicitly named team features as the unshipped scope; `done/R1-phase1-ux-foundation.md` and `done/R2-phase2-interview-prep-polish.md` close-outs both deferred B2B/team work to a dedicated proposal.

---

## 1. One-paragraph thesis

Today the platform is single-user only: there's no concept of a team, no shared library, no completion tracking that spans members, and SCORM export only works for classes (not courses). Three personas blocked on this: the Corporate Trainer (#10) who needs a team dashboard with completion tracking, the K-12 Teacher who already has class infrastructure but wants it extended to multi-section / co-teacher flows, and the Internal Climber who wants to share an interview-prep series with their hiring manager for feedback. This proposal adds a `Team` Firestore collection with explicit owner + member roles, extends the existing class-share-token pattern (`/classes/join/[shareToken]`) into a team-invite pattern, scopes the unified library by team, persists completion events to a `team_completions` collection visible to team owners, and extends `kitesforu-lms` to package multi-episode courses as multi-SCO SCORM 1.2 packages. None of this requires breaking changes to existing single-user flows — all team features are additive and feature-flagged.

## 2. Goals / Non-goals

### Goals

1. Team owner can create a team, invite members by email (5+ at a time), and revoke membership.
2. Team library shows all content shared by team owner; members see it as a filter pill in the unified library.
3. Completion tracking: team owner sees a per-content / per-member completion grid (who listened to what, when, how far).
4. Simple analytics for team owner: completion rate per content, per member; trend over a 30-day window.
5. SCORM 1.2 export works for multi-episode courses (not just classes), packaged as multi-SCO with a default ordered launch sequence.
6. SCORM packages validate in at least 2 LMS platforms (Moodle + Docebo are the default targets).
7. Team owner billing = sum of member usage (tracked but not yet enforced — billing rules ship in a follow-up).

### Non-goals

- **Free-text team chat / discussion** (Slack-style threads on content). Out of scope; users have Slack.
- **Real-time co-listening** (synchronous group playback). Out of scope.
- **Multi-org hierarchy** (an org with multiple teams under it). Single-team-per-account in v1; nested orgs in a follow-up.
- **Per-member content authoring permissions.** Only the team owner authors / shares. Members consume.
- **xAPI / cmi5 export.** SCORM 1.2 only in v1 (existing infra basis). xAPI follow-up if customer demand surfaces.
- **Single sign-on (SSO) for team members.** Members sign up via Clerk like every other user; team-membership lookup happens after auth. SAML SSO is a v2 follow-up.
- **Invoicing / pooled credit billing.** Tracking only in v1; pricing changes ship as a separate proposal.

## 3. Current surface — file:line references

| Concern | Location |
| --- | --- |
| Existing class-share token | `app/classes/[classId]/page.tsx` + `app/classes/join/[shareToken]/page.tsx` |
| Class teacher dashboard | `components/classes/TeacherDashboard.tsx` |
| Unified library | `app/library/page.tsx` + `hooks/useUnifiedLibrary.ts` |
| Library filter pills | `app/library/page.tsx` (5 pills today: All / Interview Prep / Audio Series / Classes / Writing) |
| Sharing modal | `components/sharing/ShareModal.tsx` + `components/ShareContentModal.tsx` |
| Completion analytics | `lib/analytics.ts` `content_completed` event (PR #506 emits to Cloud Logging) |
| SCORM (classes only) | `kitesforu-lms/src/server.py` + `kitesforu-frontend/app/classes/import-scorm/page.tsx` |
| Course detail | `app/courses/[courseId]/page.tsx` |
| Clerk auth | `lib/api.ts` (api wrapper sends Clerk token) |
| User profile schema | `kitesforu-schemas/src/kitesforu_schemas/users.py` (UserProfile model) |
| `team*` grep | **ZERO matches** for team Firestore concepts in api/schemas (verified 2026-04-24) |

## 4. Architecture

### 4.1 Schemas — Team + Membership + Completion

```python
# kitesforu-schemas — new file teams.py
class TeamRole(str, Enum):
    OWNER = "owner"
    MEMBER = "member"

class Team(BaseModel):
    model_config = ConfigDict(extra='ignore')
    id: str                          # ULID
    name: str
    owner_user_id: str               # Clerk user id of the creator
    created_at: datetime
    updated_at: datetime
    member_count: int                # denormalized for fast list rendering
    plan: Literal["trial", "team", "enterprise"] = "trial"

class TeamMembership(BaseModel):
    model_config = ConfigDict(extra='ignore')
    id: str                          # ULID
    team_id: str
    user_id: str                     # Clerk user id
    email: str                       # at-invite-time, may diverge from Clerk
    role: TeamRole
    invited_at: datetime
    accepted_at: Optional[datetime]  # None when invite is pending
    revoked_at: Optional[datetime]   # None when active

class TeamContentShare(BaseModel):
    model_config = ConfigDict(extra='ignore')
    id: str                          # ULID
    team_id: str
    content_id: str
    content_type: Literal["course", "class", "writeup", "podcast", "interview_prep"]
    shared_by_user_id: str           # team owner who shared
    shared_at: datetime

class TeamCompletion(BaseModel):
    model_config = ConfigDict(extra='ignore')
    team_id: str
    user_id: str
    content_id: str
    content_type: str
    completion_pct: float            # 0.0–1.0; >= 0.95 = "completed"
    last_event_at: datetime
    listen_seconds_total: int
    started_at: datetime
    completed_at: Optional[datetime]
```

Firestore paths:
- `teams/{team_id}` — Team doc.
- `teams/{team_id}/memberships/{membership_id}` — TeamMembership.
- `teams/{team_id}/shares/{share_id}` — TeamContentShare.
- `teams/{team_id}/completions/{user_id}_{content_id}` — TeamCompletion (composite-key doc id).

### 4.2 API endpoints

```
POST   /v1/teams                            create team (owner = caller)
GET    /v1/teams/me                         list teams the caller is in
GET    /v1/teams/{team_id}                  get team (members + owner only)
PATCH  /v1/teams/{team_id}                  update name / plan (owner only)
DELETE /v1/teams/{team_id}                  delete (owner only, soft-delete)

POST   /v1/teams/{team_id}/invites          batch invite up to 25 emails
                                              ↓ creates TeamMembership rows
                                                with accepted_at=null,
                                                emails Clerk to invite
DELETE /v1/teams/{team_id}/invites/{id}     revoke pending invite (owner only)
GET    /v1/teams/{team_id}/memberships      list members + pending invites
DELETE /v1/teams/{team_id}/memberships/{id} revoke member (owner only)

POST   /v1/teams/join/{invite_token}        accept invite (any signed-in user)

POST   /v1/teams/{team_id}/shares           team owner shares content with team
DELETE /v1/teams/{team_id}/shares/{id}      unshare

GET    /v1/teams/{team_id}/library          list of TeamContentShare entries
GET    /v1/teams/{team_id}/completions      grid view: members × content with %
GET    /v1/teams/{team_id}/analytics        30-day rollup
```

All endpoints behind `feature_team_features` flag.

### 4.3 Frontend surfaces

- `/teams` — landing for team owners; create + list teams.
- `/teams/[teamId]` — dashboard: members, shared content, completion grid, analytics.
- `/teams/[teamId]/invites` — pending invite list, cancel, resend.
- `/teams/join/[inviteToken]` — invite-acceptance page (mirrors `app/classes/join/[shareToken]/page.tsx` shape).
- `app/library/page.tsx` — adds 6th filter pill **"Team"** (visible only when caller is in ≥ 1 team) that filters to `team_id IS NOT NULL`.
- Content detail pages — owner gets a "Share with team" button; team library entries get a small team-name badge.

### 4.4 Completion tracking pipe

Existing `content_completed` analytics event already includes `content_id`, `content_type`, `completion_pct`. We extend the `/api/analytics/events` Next.js route (PR #506) to ALSO write a `TeamCompletion` row in Firestore when:
- The viewer is a team member (`team_membership` lookup),
- AND the content is in `team_shares` for that team.

This is a single additional Firestore write per completed event. Cost negligible.

### 4.5 SCORM-for-courses extension

`kitesforu-lms/src/server.py` already builds SCORM packages for classes. The class is naturally single-SCO (one lesson). A course is multi-SCO (one episode = one SCO).

```
content/
  imsmanifest.xml         ← lists every episode as a <resource>
  sco_001/index.html      ← per-episode launch HTML with audio player
  sco_002/index.html
  ...
  shared/
    audio/                ← MP3 files
    cmi-bridge.js         ← SCORM 1.2 cmi.* communication
```

Default sequencing: ordered (episode 1 → 2 → 3, mastery-gate at 80% completion). Team owner can override per-package in v2.

### 4.6 Firestore security rules

```
match /teams/{teamId} {
  // Owner can write; members can read.
  allow read: if request.auth.uid == resource.data.owner_user_id ||
                 exists(/databases/$(database)/documents/teams/$(teamId)/memberships/$(request.auth.uid));
  allow write: if request.auth.uid == resource.data.owner_user_id;

  match /memberships/{membershipId} {
    allow read: if request.auth.uid == get(/databases/$(database)/documents/teams/$(teamId)).data.owner_user_id ||
                  request.auth.uid == resource.data.user_id;
    allow write: if request.auth.uid == get(/databases/$(database)/documents/teams/$(teamId)).data.owner_user_id;
  }

  match /shares/{shareId} {
    allow read: if request.auth.uid == get(/databases/$(database)/documents/teams/$(teamId)).data.owner_user_id ||
                  exists(/databases/$(database)/documents/teams/$(teamId)/memberships/$(request.auth.uid));
    allow write: if request.auth.uid == get(/databases/$(database)/documents/teams/$(teamId)).data.owner_user_id;
  }

  match /completions/{completionId} {
    allow read: if request.auth.uid == get(/databases/$(database)/documents/teams/$(teamId)).data.owner_user_id ||
                  request.auth.uid == resource.data.user_id;
    allow write: if false;  // server-only writes via analytics pipe
  }
}
```

## 5. Phased scope

### Phase 1 — team CRUD + invite flow (1 week)

- schemas: Team / TeamMembership / TeamContentShare / TeamCompletion.
- api: CRUD endpoints + invite acceptance + share endpoints.
- frontend: `/teams` landing + `/teams/[teamId]` dashboard scaffold + invite flow (mirrors class-share patterns).
- Firestore security rules + composite indexes.
- Single hard-coded plan ("trial") — no plan picker UI yet.

### Phase 2 — completion tracking + team library + analytics (1 week)

- Analytics pipe writes to `team_completions` when applicable (one Firestore write per completion event).
- Library 6th filter pill "Team" visible to team members.
- Team dashboard completion grid (members × content with %).
- 30-day rollup analytics view.
- "Share with team" button on content detail pages (owner only).

### Phase 3 — SCORM-for-courses + 2-LMS validation (1 week)

- `kitesforu-lms` extension: multi-SCO course packaging.
- Default sequencing rules (ordered + 80% mastery gate).
- Validation pass: install in Moodle + Docebo, confirm cmi.* writes work end-to-end.
- Team-owner UI: "Export as SCORM" button on team dashboard for shared courses.

## 6. Acceptance criteria

- [ ] Team owner can create a team, see it in `/teams`, edit name + delete.
- [ ] Owner can batch-invite 5+ emails; each invite generates a `TeamMembership` with `accepted_at=null` and an emailed invite link.
- [ ] Invitee clicks link → signs in via Clerk → lands on `/teams/join/[token]` → confirms → `accepted_at` populates.
- [ ] Owner can revoke a pending invite or active member; `revoked_at` populates; member loses access immediately.
- [ ] Team library lists shared content; members see content via "Team" filter pill in `/library`.
- [ ] `content_completed` event fires for a team member playing shared content → a `TeamCompletion` row appears in Firestore within 5 seconds.
- [ ] Team dashboard renders a member × content completion grid; cells show 0–100% with a tooltip showing last_event_at.
- [ ] 30-day analytics view shows per-member trend line + team total.
- [ ] Course → SCORM 1.2 multi-SCO export works for a 3-episode course; the package validates in Moodle and Docebo (manual QA gate).
- [ ] Firestore security rules deny non-owner writes to team data; deny non-member reads of team library.
- [ ] All endpoints behind `feature_team_features` flag (default OFF in api).
- [ ] Frontend gated behind `feature_team_features_ui` (default OFF). Flag OFF → `/teams/*` 404.
- [ ] Playwright E2E on `beta.kitesforu.com` per CLAUDE.md rule 11: create team → invite second user → second user accepts → owner shares content → second user plays → owner sees completion in dashboard.
- [ ] api + frontend + lms hard-block test gates pass.
- [ ] Schemas bump (additive); legacy clients unaffected.

## 7. Risks & mitigations

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Email invites bounce / land in spam | High | Use Clerk's organization-invite flow as a fallback; surface a "copy invite link" path so owners can share via Slack/email manually. |
| User accepts invite for wrong email account | Medium | The accept page surfaces "you're signing in as X — is this the right account?" before finalizing membership. |
| Team library leaks via unified library when flag is OFF | Low | Library filter pill is conditionally rendered behind `feature_team_features_ui`. Server-side `team_id` filter on the library endpoint also gates. |
| Completion writes burn Firestore quota at scale | Low | One write per `content_completed` event, ~10–100/day per active team. At 1k teams, ~100k writes/day = ~$0.06/day. Bounded. |
| SCORM 1.2 cmi.* communication breaks audio playback in some LMSes | Medium | Validate against Moodle + Docebo before declaring D5 done. If a third LMS surfaces a quirk, mitigate per-LMS in the cmi-bridge.js. Don't block ship on universal LMS support. |
| Team owner shares content they don't own | Low | Share endpoint enforces `content.owner_user_id == team.owner_user_id` before creating share. |
| Race: revoked member retains client-side access until token refresh | Low | Frontend re-checks team membership on each navigation; api always re-validates via Firestore lookup. Worst case: ~5 min of lingering access. |
| GDPR / data residency: team completions hold user listening behavior | Medium | Include team data in the existing data-export endpoint; allow team owner to delete a member's completion history on revoke (default: retain for 30 days). |
| Plan = "trial" hardcoded in v1 means no upgrade path | Acknowledged | Phase 1 ships without billing; subsequent proposal handles plan picker + Stripe integration. Trial cap = 5 members. |

## 8. Out of scope (recap)

- Real-time co-listening / live discussion threads.
- Multi-org / nested team hierarchies.
- Per-member content authoring permissions.
- xAPI / cmi5 export.
- SAML SSO.
- Pooled credit billing / invoicing.

## 9. Codex audit asks

1. **Plan tier hardcoding for Phase 1** — is "trial" + 5-member cap acceptable for the first ship, with billing in a separate proposal? Or do we need at least a "team" paid tier ready before announcing this externally?
2. **Email-vs-Clerk-invite primary path** — default proposal uses Clerk org invites for delivery + our own `TeamMembership` row for state. Alternative: roll our own SES invite. Trade: Clerk gives us auth state for free; SES gives us full template control.
3. **Course SCORM default sequencing** — ordered + 80% mastery gate is the default. Acceptable for v1, or do enterprise LMSes need free-navigation as the default?
4. **Library filter pill placement** — 6th pill ("Team") at the end of the existing 5? Or replace one when caller is in a team? Default proposal: append (lower discovery cost).
5. **Completion grid privacy** — owner sees every member's completion %. Some teams may want member-anonymous "team-wide rate" only. v1 ships with full visibility; configurable in v2.
6. **What happens to a team's content shares when the OWNER revokes their account?** Default proposal: orphan the team to a "deactivated" state; members lose write access but keep read access for 30 days. Confirm.

## 10. Rollback

Phase 1 + 2 + 3 each ships behind independent flags:
- `feature_team_features` (api, default OFF) — flag OFF → all team endpoints 404.
- `feature_team_features_ui` (frontend, default OFF) — flag OFF → `/teams/*` 404, library pill hidden.
- `feature_scorm_courses` (lms, default OFF) — flag OFF → SCORM-for-courses endpoint 404.

Schemas additions are additive; legacy clients unaffected. Firestore security rules don't apply if the docs don't exist (no team docs created when api flag is OFF).

## 11. Sources

- `kitesforu-frontend/app/classes/[classId]/page.tsx` — class share-token pattern this proposal mirrors
- `kitesforu-frontend/app/library/page.tsx` — unified library filter pills extended
- `kitesforu-frontend/components/classes/TeacherDashboard.tsx` — UI pattern for team dashboard
- `kitesforu-lms/src/server.py` — class SCORM extended to course SCORM
- `kitesforu-frontend/lib/analytics.ts` — `content_completed` event the completion pipe consumes
- `proposed/R3-platform-growth-distribution.md` — partial-shipped status that named D5 as remaining
- `done/R1-phase1-ux-foundation.md` — close-out that deferred B2B work
- Clerk Organizations API documentation
- SCORM 1.2 Run-Time Environment specification
