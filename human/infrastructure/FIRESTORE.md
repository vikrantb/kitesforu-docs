# Firestore Database

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses Firestore in Native mode as its primary database. It stores job data, user information, credit transactions, and model router state.

## Database Configuration

| Attribute | Value |
|-----------|-------|
| Database ID | (default) |
| Mode | Native |
| Location | us-central1 |

## Collections

### podcast_jobs

**Purpose**: Job state and metadata

**Document ID**: job_id (UUID)

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| job_id | string | UUID |
| user_id | string | Clerk user ID |
| topic | string | User's topic (1-500 chars) |
| duration_min | float | Target duration |
| status | string | JobStatus enum |
| progress.stage | string | JobStage enum |
| progress.pct | int | 0-100 |
| research_plan | object | Generated research plan |
| outputs.audio_url | string | GCS URL |
| created_at | timestamp | Creation time |
| updated_at | timestamp | Last update |

**Indexes**:
- user_id + created_at DESC
- status + created_at DESC

---

### users

**Purpose**: User data and subscription info

**Document ID**: user_id (Clerk user ID)

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| user_id | string | Clerk user ID |
| email | string | User email |
| tier | string | SubscriptionTier enum |
| credits_balance | int | Current credits |
| stripe_customer_id | string | Stripe customer ID |
| created_at | timestamp | Registration time |

**Indexes**:
- email
- stripe_customer_id

---

### credit_transactions

**Purpose**: Credit transaction history

**Document ID**: transaction_id (UUID)

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| transaction_id | string | UUID |
| user_id | string | Clerk user ID |
| type | string | TransactionType enum |
| amount | int | +/- credits |
| balance_after | int | Balance after transaction |
| job_id | string | Related job (optional) |
| created_at | timestamp | Transaction time |

**Indexes**:
- user_id + created_at DESC
- job_id

---

### model_provider_health

**Purpose**: LLM provider health tracking

**Document ID**: provider_id

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| provider | string | Provider name |
| status | string | HealthStatus enum |
| last_check | timestamp | Last health check |
| latency_ms | int | Average latency |
| error_rate | float | Error rate |

---

### model_quotas

**Purpose**: Provider usage limits tracking

**Document ID**: {provider}_{model_id}

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| provider | string | Provider name |
| model_id | string | Model identifier |
| tokens_used | int | Tokens consumed |
| tokens_limit | int | Token limit |
| reset_at | timestamp | Quota reset time |

---

### model_router_alerts

**Purpose**: Routing system alerts

**Document ID**: alert_id (UUID)

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| type | string | AlertType enum |
| severity | string | AlertSeverity enum |
| provider | string | Related provider |
| message | string | Alert message |
| acknowledged | bool | Acknowledged flag |
| created_at | timestamp | Alert time |

**TTL**: 90 days (acknowledged alerts)

---

### routing_decisions

**Purpose**: Audit log of model routing decisions

**Document ID**: decision_id (UUID)

**Key Fields**:
| Field | Type | Description |
|-------|------|-------------|
| job_id | string | Related job |
| task_type | string | TaskType enum |
| selected_provider | string | Chosen provider |
| selected_model | string | Chosen model |
| score | float | Selection score |
| timestamp | timestamp | Decision time |

**TTL**: 30 days

## Querying

### Python (google-cloud-firestore)

```python
from google.cloud import firestore

db = firestore.Client(project="kitesforu-dev")

# Get a document
job_ref = db.collection("podcast_jobs").document(job_id)
job = job_ref.get().to_dict()

# Query with filters
jobs = db.collection("podcast_jobs") \
    .where("user_id", "==", user_id) \
    .where("status", "==", "completed") \
    .order_by("created_at", direction=firestore.Query.DESCENDING) \
    .limit(10) \
    .stream()

# Update a document
job_ref.update({
    "status": "running",
    "updated_at": firestore.SERVER_TIMESTAMP
})
```

### gcloud CLI

```bash
# Get document
gcloud firestore documents describe \
  --database="(default)" \
  --collection=podcast_jobs \
  --document={job_id}

# Export collection
gcloud firestore export gs://backup-bucket \
  --collection-ids=podcast_jobs
```

## Indexes

### Composite Indexes

Firestore requires composite indexes for queries with multiple conditions:

```yaml
# firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "podcast_jobs",
      "fields": [
        {"fieldPath": "user_id", "order": "ASCENDING"},
        {"fieldPath": "created_at", "order": "DESCENDING"}
      ]
    }
  ]
}
```

Deploy indexes:

```bash
firebase deploy --only firestore:indexes
```

## TTL Policies

Automatic cleanup for old data:

| Collection | Field | Duration | Condition |
|------------|-------|----------|-----------|
| routing_decisions | timestamp | 30 days | All documents |
| model_router_alerts | created_at | 90 days | acknowledged == true |

## Backup & Restore

### Export

```bash
gcloud firestore export gs://kitesforu-backups/$(date +%Y%m%d)
```

### Import

```bash
gcloud firestore import gs://kitesforu-backups/20240115
```

## Best Practices

1. **Use batched writes** for multiple updates
2. **Avoid large documents** (max 1MB)
3. **Use subcollections** for related data
4. **Index thoughtfully** - Only create needed indexes
5. **Handle offline** gracefully in clients
