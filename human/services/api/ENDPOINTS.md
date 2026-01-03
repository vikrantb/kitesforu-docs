# API Endpoints

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Base URL

```
https://kitesforu-api-m6zqve5yda-uc.a.run.app
```

## Authentication

All endpoints except `/v1/health` require a Bearer token in the Authorization header:

```
Authorization: Bearer <clerk_jwt_token>
```

---

## Health

### GET /v1/health

Health check endpoint.

**Auth**: None

**Response**
```json
{
  "status": "ok"
}
```

---

## Podcasts

### POST /v1/podcasts

Create a new podcast job.

**Auth**: Required

**Request Body**
```json
{
  "topic": "Introduction to quantum computing",
  "duration_min": 10,
  "style": "Explainer",
  "audience": "beginners",
  "allow_premium": false
}
```

**Response** (201)
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "questions": null
}
```

**Errors**
- 400: Validation error
- 401: Unauthorized
- 402: Insufficient credits
- 429: Quota exceeded

---

### GET /v1/podcasts/{job_id}/status

Get job status and progress.

**Auth**: Required

**Response**
```json
{
  "status": "running",
  "progress": {
    "stage": "script",
    "pct": 45
  },
  "awaiting_user_action": null
}
```

---

### GET /v1/podcasts/{job_id}/result

Get completed podcast result.

**Auth**: Required

**Response** (200)
```json
{
  "audio_url": "https://storage.googleapis.com/...",
  "script": {
    "segments": [...],
    "metadata": {...}
  }
}
```

**Errors**
- 404: Job not found
- 409: Job not completed

---

### PATCH /v1/podcasts/{job_id}/answers

Submit clarifier question answers.

**Auth**: Required

**Request Body**
```json
{
  "answers": {
    "q1": "technical professionals",
    "q2": "intermediate"
  }
}
```

**Errors**
- 409: Job not in CLARIFYING state

---

### GET /v1/podcasts/{job_id}/research-plan

Get research plan for approval.

**Auth**: Required

**Response**
```json
{
  "tasks": [
    {
      "task_id": "t1",
      "task_type": "web_search",
      "query": "quantum computing basics",
      "estimated_credits": 5
    }
  ],
  "user_facing_summary": "I'll research...",
  "total_estimated_credits": 15,
  "status": "pending_approval",
  "can_edit": true
}
```

---

### POST /v1/podcasts/{job_id}/research-plan/approve

Approve research plan.

**Auth**: Required

**Response**
```json
{
  "success": true,
  "job_status": "running",
  "message": "Research plan approved"
}
```

---

### POST /v1/podcasts/{job_id}/research-plan/edit

Edit research plan (max 1 edit).

**Auth**: Required

**Request Body**
```json
{
  "tasks": [
    {
      "task_id": "t1",
      "task_type": "web_search",
      "query": "updated query",
      "estimated_credits": 5,
      "enabled": true
    }
  ]
}
```

---

### POST /v1/podcasts/{job_id}/cancel

Cancel a running job.

**Auth**: Required

**Errors**
- 409: Cannot cancel (already completed/failed)

---

## User

### GET /v1/me/limits

Get user tier and limits.

**Auth**: Required

**Response**
```json
{
  "tier": "enthu",
  "tier_name": "Enthusiast",
  "credits_balance": 850,
  "credits_per_month": 1000,
  "max_duration_min": 30
}
```

---

### GET /v1/me/credits

Get credit balance.

**Auth**: Required

**Response**
```json
{
  "credits_balance": 850
}
```

---

### GET /v1/me/credits/history

Get credit transaction history.

**Auth**: Required

**Response**
```json
{
  "transactions": [
    {
      "transaction_id": "...",
      "type": "deduction",
      "amount": -50,
      "description": "Podcast: Quantum Computing",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

---

## Pricing

### POST /v1/pricing/calculate

Calculate job cost.

**Auth**: Optional

**Request Body**
```json
{
  "duration_min": 15,
  "quality": "standard",
  "priority": "standard"
}
```

**Response**
```json
{
  "credits_required": 75,
  "breakdown": {
    "base": 75,
    "quality_multiplier": 1.0,
    "priority_multiplier": 1.0
  },
  "estimated_cost_usd": 7.50
}
```

---

## Checkout

### POST /v1/checkout

Create Stripe checkout session.

**Auth**: Required

**Request Body**
```json
{
  "tier": "enthu",
  "success_url": "https://...",
  "cancel_url": "https://..."
}
```

**Response**
```json
{
  "checkout_url": "https://checkout.stripe.com/...",
  "session_id": "cs_..."
}
```

---

### POST /v1/customer-portal

Get Stripe customer portal URL.

**Auth**: Required

**Response**
```json
{
  "portal_url": "https://billing.stripe.com/..."
}
```

---

## Webhooks

### POST /v1/webhooks/stripe

Handle Stripe webhook events.

**Auth**: Stripe signature verification

**Headers**
- `stripe-signature`: Required
