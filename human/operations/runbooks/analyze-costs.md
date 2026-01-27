# Cost Analysis Runbook

## Overview

This runbook describes how to analyze podcast production costs using the offline cost analysis tools. These are **BI/analytics tools** - they do NOT impact job processing.

## Prerequisites

1. Access to Firestore (via service account or gcloud auth)
2. Python environment with `kitesforu-workers` dependencies
3. Firestore composite index deployed (see Infrastructure Requirements)

## Tools Available

### analyze_job_costs.py

Location: `kitesforu-workers/scripts/analyze_job_costs.py`

**Usage:**

```bash
# Navigate to workers repo
cd kitesforu-workers

# Analyze 10 most recent completed jobs
python scripts/analyze_job_costs.py --recent 10

# Analyze specific jobs
python scripts/analyze_job_costs.py --jobs job_id_1 job_id_2

# Analyze all jobs from last 7 days
python scripts/analyze_job_costs.py --days 7

# Output as JSON for further processing
python scripts/analyze_job_costs.py --recent 10 --json

# Summary only (no individual job reports)
python scripts/analyze_job_costs.py --recent 10 --summary-only
```

## Output Interpretation

### Per-Job Report

```
============================================================
Job: abc123
Topic: Latest tech news in AI...
Duration: 5 min | Language: en-US
------------------------------------------------------------
Stage Costs (USD):
  research: 0.0200
  script: 0.0035
  audio: 0.0180
------------------------------------------------------------
Total Cost (USD): $0.0415
Cost per 5 min (USD): $0.0415
```

### Summary Statistics

```
============================================================
COST SUMMARY
============================================================
Jobs Analyzed: 10
Average Duration: 7.5 min
------------------------------------------------------------
Average Cost per Job: $0.0520
------------------------------------------------------------
COST PER 5 MINUTES (Production Cost Baseline):
  Average: $0.0380
  Best Case: $0.0280
  Worst Case: $0.0580
------------------------------------------------------------

For 30 min podcast (6 x cost per 5 min):
  Average: $0.2280
  Worst Case: $0.3480
```

## Key Metrics

| Metric | Description | Use Case |
|--------|-------------|----------|
| `cost_per_5min_usd` | Normalized cost per 5 minutes | Pricing tier baseline |
| `total_cost_usd` | Actual job cost | Per-job accounting |
| `stage_costs_credits` | Per-stage breakdown | Cost attribution |

## Pricing Calculation

### Base Tier (Current)

Costs are calculated from:
- **LLM**: Token usage x model rate (from model_catalog.csv)
- **TTS**: Character count x TTS rate
- **Research**: Budget spent on research stage

### Scaling Formula

For any duration:
```
estimated_cost = cost_per_5min * (duration_minutes / 5)
```

Example: 30-minute podcast at $0.04/5min = $0.24 production cost

## Infrastructure Requirements

### Firestore Index

The script requires a composite index on `podcast_jobs`:

```
Collection: podcast_jobs
Fields:
  - status (Ascending)
  - completed_at (Descending)
```

**Deployment:**
- Managed via Terraform in `kitesforu-infrastructure`
- Resource: `google_firestore_index.podcast_jobs_status_completed`

If you see "index required" errors, ensure the infrastructure PR is deployed.

## Troubleshooting

### "Index required" Error

The composite index hasn't been deployed. Options:
1. Wait for infrastructure PR to be merged and deployed
2. Create index manually via Firebase Console
3. Use `--jobs` flag with specific job IDs (no index required)

### No Jobs Found

- Check date range or limit
- Verify jobs have `status: completed` and `completed_at` field
- Check Firestore connectivity

### Cost Report Missing for Job

Jobs may lack cost data if:
- Job completed before cost tracking was added
- `budget_by_stage` field is missing or empty
- Job is not in `completed` status

## Data Sources

Cost reports use existing job document data:

| Field | Source | Description |
|-------|--------|-------------|
| `budget_by_stage` | Job document | Per-stage budget tracking |
| `inputs.duration_min` | Job document | Target duration |
| `status` | Job document | Job completion status |
| `completed_at` | Job document | Completion timestamp |

## Related Documentation

- [Model Pricing Catalog](../../architecture/model-catalog.md) (if exists)
- [Firestore Schema](../../../llm/reference/firestore-schema.yaml)
- [Workers Architecture](../../architecture/workers.md) (if exists)
