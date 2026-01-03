# Schemas Package Overview

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Summary

The `kitesforu-schemas` package is a shared Python library containing all Pydantic models and enums used across the platform. It ensures type safety and consistent data structures between the API and workers.

## Quick Reference

| Attribute | Value |
|-----------|-------|
| Package Name | kitesforu-schemas |
| Language | Python 3.11+ |
| Distribution | Artifact Registry |
| Consumers | API, Workers |

## Purpose

1. **Single Source of Truth** - All data models in one place
2. **Type Safety** - Pydantic validation across services
3. **Consistency** - Same models used everywhere
4. **Documentation** - Models serve as API documentation

## Project Structure

```
kitesforu-schemas/
├── src/kitesforu_schemas/
│   ├── __init__.py          # Package exports
│   ├── enums.py              # All enumerations
│   └── models.py             # Pydantic models
├── tests/
├── pyproject.toml
└── README.md
```

## Enumerations

### Job-Related
```python
class JobStatus(str, Enum):
    QUEUED = "queued"
    CLARIFYING = "clarifying"
    RUNNING = "running"
    FAILED = "failed"
    COMPLETED = "completed"

class JobStage(str, Enum):
    QUEUED = "queued"
    RESEARCH = "research"
    SCRIPT = "script"
    VOICE = "voice"
    MIX = "mix"
    PUBLISH = "publish"
    COMPLETED = "completed"
    FAILED = "failed"
```

### Tier-Related
```python
class SubscriptionTier(str, Enum):
    FREE = "free"
    ENTHU = "enthu"
    PRO = "pro"
    PRO_PLUS = "pro_plus"
    ULTIMATE = "ultimate"

class QueuePriority(str, Enum):
    STANDARD = "standard"
    PRIORITY = "priority"
    EXPRESS = "express"
```

### Model Router
```python
class ModelProvider(str, Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    GOOGLE = "google"
    ELEVENLABS = "elevenlabs"

class HealthStatus(str, Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNAVAILABLE = "unavailable"
```

## Key Models

### Job Models
```python
class PodcastJob(BaseModel):
    job_id: str
    user_id: str
    inputs: PodcastJobInput
    status: JobStatus
    progress: JobProgress
    outputs: Optional[PodcastJobOutputs]
    created_at: datetime
    updated_at: datetime

class PodcastJobInput(BaseModel):
    topic: str = Field(..., min_length=1, max_length=500)
    duration_min: float = Field(..., ge=0.5, le=180.0)
    style: PodcastStyle
    audience: Optional[str]
```

### API Models
```python
class CreateJobRequest(BaseModel):
    topic: str
    duration_min: float
    style: PodcastStyle
    audience: Optional[str]
    allow_premium: bool = False

class CreateJobResponse(BaseModel):
    job_id: str
    questions: Optional[List[ClarifierQuestion]]
```

### Payment Models
```python
class User(BaseModel):
    user_id: str
    email: EmailStr
    credits_balance: int
    tier: SubscriptionTier
    stripe_customer_id: Optional[str]

class CreditTransaction(BaseModel):
    transaction_id: str
    user_id: str
    type: TransactionType
    amount: int
    balance_after: int
    created_at: datetime
```

## Validation Rules

### Topic
- Minimum: 1 character
- Maximum: 500 characters

### Duration
- Minimum: 0.5 minutes (30 seconds)
- Maximum: 180 minutes (3 hours)

### Audience
- Maximum: 200 characters
- Optional field

## Installation

### From Artifact Registry
```bash
pip install kitesforu-schemas \
  --extra-index-url https://us-central1-python.pkg.dev/kitesforu-dev/kitesforu-python/simple/
```

### In requirements.txt
```
--extra-index-url https://us-central1-python.pkg.dev/kitesforu-dev/kitesforu-python/simple/
kitesforu-schemas>=1.0.0
```

## Usage

```python
from kitesforu_schemas import (
    PodcastJob,
    JobStatus,
    JobStage,
    CreateJobRequest,
)

# Create a job request
request = CreateJobRequest(
    topic="Introduction to Machine Learning",
    duration_min=10,
    style=PodcastStyle.EXPLAINER,
    audience="beginners"
)

# Parse job from Firestore
job = PodcastJob(**firestore_doc)
print(f"Status: {job.status}")
```

## Publishing

### Build
```bash
cd kitesforu-schemas
python -m build
```

### Upload
```bash
twine upload \
  --repository-url https://us-central1-python.pkg.dev/kitesforu-dev/kitesforu-python/ \
  dist/*
```

## Versioning

- Follows Semantic Versioning (SemVer)
- Breaking changes = major version bump
- New features = minor version bump
- Bug fixes = patch version bump
