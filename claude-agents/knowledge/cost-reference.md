# KitesForU Cost & Financial Reference
<!-- last_verified: 2026-02-07 -->

## Credit System

### Tier Pricing
| Tier | Monthly Price | Monthly Credits | Max Duration | LLM Model | TTS Model |
|------|-------------|----------------|-------------|-----------|-----------|
| Trial | $0 | 10 | 1 min | gpt-4.1-mini | tts-1 |
| Enthusiast | $49 | 1,000 | 15 min | gpt-4.1-mini | tts-1 |
| Creator | $99 | 2,500 | 30 min | gpt-5 | tts-1 |
| Pro Creator | $199 | 6,000 | 60 min | claude-sonnet-4-5 | eleven_flash_v2_5 |
| Ultimate Creator | $399 | 15,000 | 120 min | claude-sonnet-4-5 | eleven_multilingual_v2 |
| Pay-As-You-Go | $0 base | 0 | 10 min | gpt-4.1-mini | tts-1 |

### PAYG Credit Packages
- Starter: 100 credits / $6.00 ($0.060/credit)
- Small: 250 credits / $14.00 ($0.056/credit)
- Medium: 500 credits / $27.00 ($0.054/credit)
- Large: 1,000 credits / $50.00 ($0.050/credit)

### Credit Calculation Formula
```
Credits = Duration(minutes) x Quality_Multiplier x Priority_Multiplier

Quality: Standard=1.0x, HD=1.5x
Priority: Standard=1.0x, Priority=1.5x, Express=2.0x

Example: 10 min HD Express = 10 x 1.5 x 2.0 = 30 credits
```

### Credit Operations (All Atomic)
- **Location**: `kitesforu-api/src/api/credit_manager.py`
- deduct_credits() -- validates balance, creates transaction, updates user
- refund_credits() -- abuse detection (>5 refunds/30 days), over-refund prevention
- add_credits() -- for purchases, subscription grants, admin adjustments
- **Storage**: `users` collection (balance), `credit_transactions` (audit log)

## Stripe Integration
- **Location**: `kitesforu-api/src/api/services/stripe.py`
- **Webhooks**: customer.subscription.created/updated/deleted, invoice.payment_succeeded/failed
- **Security**: IP whitelisting, signature verification, replay protection, event deduplication
- **Idempotency**: processed_invoices collection prevents duplicate credit grants
- **Endpoints**: POST /v1/checkout, POST /v1/customer-portal, POST /v1/webhooks/stripe, GET /v1/me/credits

## Per-Stage Budget Tracking
- **Location**: `kitesforu-workers/src/workers/common/budget_tracker.py`

### Stage Budget Allocation (Podcast Jobs)
| Stage | Budget % | Typical Cost |
|-------|---------|-------------|
| Initiate | 0% | $0 (no LLM) |
| Plan | 5% | ~$0.005 |
| Execute Tools (Research) | 25% | ~$0.012 |
| Assimilator | 5% | ~$0.005 |
| Script | 40% | ~$0.023 |
| Audio (TTS) | 25% | ~$0.015 |

- Values stored in `podcast_jobs.budget_by_stage` as **USD** (not credits)
- Budget enforced BEFORE each operation (BudgetExceededError)
- Cost recorded AFTER each operation (atomic increment)

## Model Pricing (from model_catalog.csv)

### LLM Costs
| Model | Input $/MTok | Output $/MTok | Agentic Multiplier |
|-------|-------------|--------------|-------------------|
| gpt-4.1-mini | $0.70 | $2.80 | - |
| gpt-5-mini | $0.25 | $3.60 | - |
| gpt-5 | $1.25 | $10.00 | 1.5x |
| claude-sonnet-4-5 | $3.00 | $15.00 | 1.2x |
| gemini-2.0-flash-exp | $0.10 | - | - |

### TTS Costs
| Model | $/1M chars | ~$/min audio (1000 chars/min) |
|-------|-----------|-------------------------------|
| tts-1 | $15 | $0.015 |
| tts-1-hd | $30 | $0.030 |
| eleven_flash_v2_5 | $150 | $0.150 |
| eleven_multilingual_v2 | $300 | $0.300 |
| gcp-neural2 | $16 | $0.016 |

### Research Task Costs
| Task Type | Credits | Duration (seconds) |
|-----------|---------|-------------------|
| web_search | 3.0 | 15 |
| url_extraction | 2.0 (x depth) | 20 (x depth) |
| document_scan | 4.0 | 30 |
| deep_research | 8.0 | 60 |

## Actual Cost Calculation
- **Location**: `kitesforu-api/src/api/services/tiers/calculations.py`
- Function: `calculate_actual_cost(duration_min, quality, llm_model)`
- Formula: LLM cost + TTS cost + infra cost ($0.02/min)
- Used for margin analysis, not exposed to users

## Cost Analysis Tool (Offline)
- **Location**: `kitesforu-workers/src/workers/common/cost_reporter.py`
- **Script**: `kitesforu-workers/scripts/analyze_job_costs.py`
- Usage:
  ```bash
  python scripts/analyze_job_costs.py --recent 10          # Last 10 jobs
  python scripts/analyze_job_costs.py --days 7             # Last 7 days
  python scripts/analyze_job_costs.py --recent 50 --summary-only  # Summary only
  python scripts/analyze_job_costs.py --recent 10 --json   # JSON output
  ```
- Outputs: total cost USD, cost per 5-min (normalized), stage breakdown, batch averages

## Key Financial Files
| File | Purpose |
|------|---------|
| `kitesforu-api/src/api/credit_manager.py` | Credit operations |
| `kitesforu-api/src/api/services/stripe.py` | Payment processing |
| `kitesforu-api/src/api/services/tiers/configs.py` | Tier definitions |
| `kitesforu-api/src/api/services/tiers/calculations.py` | Actual cost calculation |
| `kitesforu-workers/src/workers/common/budget_tracker.py` | Per-stage budget enforcement |
| `kitesforu-workers/src/workers/common/cost_reporter.py` | Offline cost analysis |
| `kitesforu-workers/src/workers/common/pricing.py` | Model cost lookups |
| `kitesforu-workers/config/model_catalog.csv` | Model pricing source |

## Financial Health Indicators
- Cost per 5-min unit (normalized comparison metric)
- Credit-to-USD conversion (track if credits are profitable)
- Stage cost distribution (is research eating too much budget?)
- Tier margin analysis (actual cost vs credit price per tier)
- Refund rate and patterns (abuse detection)
