# KitesForU Firestore Schema
<!-- last_verified: 2026-02-07 -->

## Collections

### users
- **Read by**: API, Frontend (via API)
- **Write by**: API
- **Document ID**: user_id (Clerk user ID)
- **Fields**:
  - email: string
  - credits_balance: int
  - total_credits_used: int (uses Firestore.Increment)
  - total_credits_purchased: int (uses Firestore.Increment)
  - tier: SubscriptionTier enum (FREE, ENTHU, PRO, PRO_PLUS, ULTIMATE, PAYG)
  - subscription_status: SubscriptionStatus
  - subscription_period_end: timestamp (optional)
  - stripe_customer_id: string (optional)
  - updated_at: timestamp

### courses
- **Read by**: API, Course-workers, Frontend (via API)
- **Write by**: Course-workers, API
- **Document ID**: course_id
- **Fields**:
  - user_id: string
  - topic: string
  - course_type: string ("standard" | "interview_prep")
  - num_modules: int
  - module_duration_min: float (IMPORTANT: this is a float, e.g., 0.167 for 10 seconds)
  - style: string ("Explainer" | "Storytelling" | "Interview" | "Motivational")
  - language: string (BCP 47, default "en-US")
  - audience: string (optional)
  - context: string (optional)
  - interview_prep_context: string (enriched context from API extraction -- contains combined resume + JD + help context)
  - status: string ("pending" | "generating_curriculum" | "curriculum_ready" | "generating_episodes" | "completed" | "failed")
  - curriculum: object (the generated curriculum JSON -- summary, objectives, episodes[])
  - curriculum_debug: object (LLM call details -- model, temperature, tokens, prompts, response)
  - episodes: array of objects (episode_number, title, description, job_id, status, audio_url)
  - credits_charged: int
  - created_at: timestamp
  - updated_at: timestamp

### podcast_jobs
- **Read by**: Podcast-workers, API, Frontend (via API)
- **Write by**: Podcast-workers
- **Document ID**: job_id
- **Fields**:
  - user_id: string
  - topic: string
  - status: string (pending, processing, completed, failed)
  - current_stage: string
  - language: string
  - custom_instructions: string (episode type template content, if from course orchestrator)
  - credits_deducted: int
  - cost_estimate_cents: int
  - budget_by_stage: map (stage_name -> USD cost)
  - total_budget: float (USD)
  - stages: map of stage objects (status, started_at, completed_at, duration_seconds, error, result)
  - llm_calls: array (model, tokens, cost, prompt, response)
  - tts_calls: array (provider, voice, segments, duration, cost)
  - audio_url: string (GCS signed URL)
  - created_at: timestamp
  - updated_at: timestamp
  - completed_at: timestamp

### credit_transactions
- **Read by**: API
- **Write by**: API (credit_manager.py)
- **Document ID**: transaction_id (UUID)
- **Fields**:
  - user_id: string
  - type: TransactionType (DEDUCTION, REFUND, PURCHASE, SUBSCRIPTION_GRANT, ADMIN_ADJUSTMENT)
  - amount: int (positive for additions, negative for deductions)
  - balance_after: int
  - job_id: string (optional -- which job this is for)
  - stripe_payment_intent_id: string (optional)
  - subscription_id: string (optional)
  - original_transaction_id: string (optional -- for refunds)
  - refund_reason: string (optional)
  - description: string
  - created_at: timestamp
- **Indexes**: user_id + created_at DESC, job_id

### subscriptions
- **Read by**: API
- **Write by**: API (Stripe webhook handler)
- **Document ID**: subscription_id
- **Fields**:
  - stripe_subscription_id: string
  - stripe_customer_id: string
  - tier: SubscriptionTier
  - status: SubscriptionStatus
  - current_period_start: timestamp
  - current_period_end: timestamp
  - cancel_at_period_end: boolean
  - monthly_credits: int
  - credits_granted_this_period: boolean

### processed_invoices
- **Read by**: API
- **Write by**: API (Stripe webhook handler)
- **Document ID**: invoice_id (Stripe)
- **Purpose**: Idempotency -- prevents duplicate monthly credit grants
- **Fields**:
  - subscription_id: string
  - processed_at: timestamp
  - amount: int
  - currency: string

### model_provider_health
- **Read by**: Workers (model router)
- **Write by**: Workers (health monitor)
- **Document ID**: provider name
- **Fields**:
  - provider: string (openai, anthropic, google, elevenlabs)
  - status: string (healthy, degraded, unavailable)
  - success_rate: float (0.0-1.0)
  - avg_latency_ms: float
  - error_count_24h: int
  - circuit_breaker_open: boolean
  - circuit_opened_at: timestamp
  - half_open_state: boolean
  - consecutive_successes: int
  - consecutive_failures: int
  - last_error: string

### model_quotas
- **Read by**: Workers (quota manager)
- **Write by**: Workers (quota manager, atomic transactions)
- **Document ID**: {provider}_{model_id}_{quota_type}
- **Fields**:
  - provider: string
  - model_id: string
  - quota_type: string (requests, tokens, cost)
  - limit: int
  - used: int
  - remaining: int
  - reset_at: timestamp
  - reset_period: string (hourly, daily)
  - status: string (available, warning, critical, exhausted)

### routing_decisions
- **Read by**: Workers, API (admin dashboard)
- **Write by**: Workers (model router)
- **Document ID**: auto-generated
- **Fields**:
  - job_id: string
  - task_type: string (LLM, TTS, TRANSCRIPTION)
  - selected_model: string
  - score: float
  - selection_reason: string
  - estimated_cost: float
  - fallback_chain: array of model IDs
  - all_candidates: array of {model, score, reason}
  - timestamp: timestamp

### model_router_alerts
- **Read by**: API (admin dashboard)
- **Write by**: Workers (alert manager)
- **Document ID**: auto-generated
- **Fields**:
  - type: string (QUOTA_WARNING, PROVIDER_UNAVAILABLE, CIRCUIT_BREAKER_OPENED, etc.)
  - severity: string (info, warning, critical, emergency)
  - provider: string
  - model_id: string (optional)
  - message: string
  - context: map (quota_remaining, circuit_state, etc.)
  - acknowledged: boolean
  - created_at: timestamp
  - resolved_at: timestamp

### pipeline_config
- **Read by**: Course-workers, Podcast-workers
- **Write by**: Admin
- **Purpose**: Runtime feature flags
- **Key fields**:
  - planner_enabled: boolean (routes initiate -> planner if true, else -> syllabus)
  - Other feature toggles

### circuit_breakers
- **Read by**: Workers
- **Write by**: Workers
- **Purpose**: Circuit breaker state per provider
- **Fields**: provider, state, opened_at, half_open_attempts, etc.

### error_reports / worker_errors
- **Read by**: API, all services
- **Write by**: All services
- **Purpose**: Error tracking and reporting

## Key Relationships
- users.credits_balance <- credit_transactions (running balance)
- courses.episodes[].job_id -> podcast_jobs.job_id (1:1 per episode)
- podcast_jobs.budget_by_stage -> uses model_catalog.csv pricing
- routing_decisions.job_id -> podcast_jobs.job_id
- subscriptions.stripe_customer_id -> users.stripe_customer_id

## Important Notes
1. All credit operations use Firestore TRANSACTIONS (atomic)
2. budget_by_stage values are in USD, not credits
3. module_duration_min is a FLOAT (0.167 for 10 seconds)
4. interview_prep_context is a large text field (combined resume + JD + help)
5. curriculum_debug stores the full LLM prompt/response for debugging
