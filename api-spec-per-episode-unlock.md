# KitesForU Per-Episode Unlock & Generation API Design

**Date**: 2026-02-07
**Scope**: API changes for per-episode unlock, credit deduction, and generation
**Status**: Draft

---

## 1. New Endpoints

### 1.1 Unlock Episode

```
POST /v1/courses/{course_id}/episodes/{episode_number}/unlock
```

**Request Body:** `{}` (no body required — credit cost computed server-side)

**Response (200 OK):**
```json
{
  "episode_number": 1,
  "unlock_status": "unlocked",
  "credits_deducted": 0,
  "credits_remaining": 42,
  "is_free": true,
  "message": "First episode unlocked for free."
}
```

**Response (200 OK, paid episode):**
```json
{
  "episode_number": 3,
  "unlock_status": "unlocked",
  "credits_deducted": 5,
  "credits_remaining": 37,
  "is_free": false
}
```

**Error Responses:**
- `402 Payment Required` — insufficient credits
- `409 Conflict` — episode already unlocked
- `404 Not Found` — course or episode does not exist
- `422 Unprocessable Entity` — course not in `curriculum_ready` or later state

**Server-Side Logic:**
1. Verify course exists and status is in (`curriculum_ready`, `partial`, `completed`, `generating`)
2. Verify episode exists in curriculum
3. If `episode.unlock_status != "locked"`, return 409
4. Compute `credits_required` from episode duration and multipliers
5. If `episode_number == 1`, set `credits_required = 0` (free first episode)
6. Execute Firestore transaction: deduct credits atomically, set `unlock_status = "unlocked"`, record `credits_charged_episode`
7. Return response

---

### 1.2 Generate Single Episode

```
POST /v1/courses/{course_id}/episodes/{episode_number}/generate
```

**Request Body:** `{}`

**Response (202 Accepted):**
```json
{
  "episode_number": 3,
  "unlock_status": "generating",
  "job_id": "job_abc123",
  "message": "Episode generation started."
}
```

**Error Responses:**
- `409 Conflict` — episode not in `unlocked` or `failed` state
- `404 Not Found` — course or episode does not exist
- `422 Unprocessable Entity` — course status does not allow generation

---

### 1.3 Update Episode Metadata (Post-Unlock Edit)

```
PATCH /v1/courses/{course_id}/episodes/{episode_number}
```

**Request Body:**
```json
{
  "title": "Revised Episode Title",
  "description": "Updated description focusing on advanced topics."
}
```

Both fields optional. At least one required.

**Editable States:** `unlocked`, `failed` only. Locked, generating, completed = 409.

---

### 1.4 Bulk Unlock All Episodes

```
POST /v1/courses/{course_id}/episodes/unlock-all
```

**Atomicity:** Single Firestore transaction. All-or-nothing.

**Error:** `402` if insufficient credits for all remaining locked episodes.

---

### 1.5 Generate All (Backward Compatible, Updated)

```
POST /v1/courses/{course_id}/generate
```

**Updated Behavior:**
1. Unlock all locked episodes (deduct credits atomically, episode 1 free)
2. Queue generation for all episodes in `unlocked` or `failed` state
3. Return the same response shape as today

---

### 1.6 Get Episode Credit Estimate

```
GET /v1/courses/{course_id}/episodes/{episode_number}/credit-estimate
```

**Response:**
```json
{
  "episode_number": 3,
  "credits_required": 5,
  "is_free": false,
  "breakdown": {
    "base_duration_min": 10.0,
    "quality_multiplier": 1.0,
    "priority_multiplier": 0.5,
    "computed_credits": 5
  },
  "user_credits_available": 42,
  "can_afford": true
}
```

---

## 2. Firestore Schema Changes

### 2.1 Episode-Level Fields (New)

```
curriculum.episodes[n]:
  # Existing (unchanged)
  episode_number, title, description, objectives, key_topics, duration_min,
  generation_status, audio_url, error_message, job_id

  # New fields
  unlock_status: str              # locked|unlocked|generating|completed|failed
  credits_charged_episode: int    # credits deducted for this episode (0 for ep 1)
  is_free: bool                   # true for episode 1
  unlocked_at: timestamp | null
  unlocked_by: str | null
  title_original: str | null      # audit trail
  description_original: str | null
  edited_at: timestamp | null
```

### 2.2 Course-Level Fields (New)

```
/courses/{course_id}:
  credits_charged_breakdown: map   # {episode_1: 0, episode_2: 5, ...}
  unlock_version: int              # optimistic locking counter
  generation_mode: str             # "bulk" | "per_episode"
```

### 2.3 Migration (Existing Data)

Read-repair pattern on first access:
- `generation_status == "completed"` → `unlock_status = "completed"`
- `generation_status == "failed"` → `unlock_status = "failed"`
- `generation_status == "pending"` → `unlock_status = "locked"`
- Episode 1: `is_free = true`

---

## 3. Episode State Machine

```
locked ──unlock──> unlocked ──generate──> generating ──success──> completed
                      ▲                       │
                      │ edit (same state)      │ failure
                      │                       ▼
                      └────── retry ──── failed
```

### Valid Transitions

| From | To | Trigger | Notes |
|------|----|---------|-------|
| `locked` | `unlocked` | POST .../unlock | Credits deducted |
| `unlocked` | `generating` | POST .../generate | Pub/Sub sent |
| `generating` | `completed` | Worker callback | audio_url set |
| `generating` | `failed` | Worker callback | error_message set |
| `failed` | `unlocked` | POST .../retry | Re-edit before retry |
| `failed` | `generating` | POST .../generate | Direct retry |
| `unlocked` | `unlocked` | PATCH .../ | Title/description edit |

### Dual Field Synchronization

| unlock_status | generation_status | Meaning |
|---------------|-------------------|---------|
| locked | pending | Not purchased |
| unlocked | pending | Purchased, not generated |
| generating | running | Audio in progress |
| completed | completed | Ready |
| failed | failed | Generation failed |

`generation_status` kept for backward compatibility with workers. `unlock_status` is source of truth for unlock flow.

---

## 4. Credit Flow

### Per-Episode Calculation

```python
def compute_episode_credits(episode, course) -> int:
    if episode["episode_number"] == 1:
        return 0  # First episode always free

    base = episode["duration_min"]
    quality_mult = QUALITY_MULTIPLIERS[course["quality"]]
    priority_mult = PRIORITY_MULTIPLIERS[course["priority"]]

    return max(math.ceil(base * quality_mult * priority_mult), 1)
```

### Atomic Unlock Transaction

Uses Firestore transaction spanning both user doc (credits) and course doc (episode state):
1. Read current state
2. Validate episode is locked
3. Compute cost (0 for episode 1)
4. Verify balance
5. Atomic: deduct credits + set unlock_status + increment unlock_version

### Refund Policy

Credits not refunded on generation failure — unlock is a content access purchase. Free retry included.

---

## 5. Course Status Transitions

```
curriculum_ready → generating (any episode starts) → completed (all done)
                                                   → partial (some done)
```

Status recomputation runs after every worker callback:
- Any `generating` → course = `generating`
- All `completed` → course = `completed`
- Any `completed` + others not → course = `partial`
- Course stays `curriculum_ready` during unlock-only phase

---

## 6. Error Catalog

| Code | HTTP | Trigger | Recovery |
|------|------|---------|----------|
| INSUFFICIENT_CREDITS | 402 | Balance too low | Purchase credits |
| EPISODE_ALREADY_UNLOCKED | 409 | Re-unlock | No action |
| EPISODE_LOCKED | 409 | Generate locked ep | Unlock first |
| EPISODE_GENERATING | 409 | Edit during gen | Wait |
| EPISODE_COMPLETED | 409 | Edit completed ep | Use retry |
| INVALID_COURSE_STATE | 422 | Unlock before ready | Wait for curriculum |
| CONCURRENT_MODIFICATION | 409 | Version mismatch | Retry request |

---

## 7. Endpoint Summary

| Method | Path | Purpose | Idempotent |
|--------|------|---------|------------|
| POST | /v1/courses/{id}/episodes/{ep}/unlock | Unlock single | Yes (409) |
| POST | /v1/courses/{id}/episodes/unlock-all | Unlock all locked | Yes |
| POST | /v1/courses/{id}/episodes/{ep}/generate | Generate single | Yes (409) |
| PATCH | /v1/courses/{id}/episodes/{ep} | Edit title/desc | Yes |
| GET | /v1/courses/{id}/episodes/{ep}/credit-estimate | Cost preview | Yes |
| POST | /v1/courses/{id}/generate | Generate all (compat) | No |

---

## 8. Implementation Sequence

1. Schema migration — add new fields with defaults
2. Credit estimate endpoint — read-only, safe to ship first
3. Single episode unlock — core Firestore transaction
4. Episode edit endpoint — PATCH with state validation
5. Single episode generate — Pub/Sub with updated message
6. Worker callback update — sync unlock_status
7. Course status recomputation
8. Bulk unlock-all
9. Refactor Generate All — compose from unlock-all + generate-each
10. Retry endpoint update — reset to unlocked on retry

Each step deployable independently behind feature flag.
