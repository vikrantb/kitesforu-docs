# Testing Guide

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Testing strategy and practices for KitesForU services.

## Test Types

| Type | Scope | Tools | Run Frequency |
|------|-------|-------|---------------|
| Unit | Functions, classes | pytest, jest | Every commit |
| Integration | Service boundaries | pytest, supertest | Every PR |
| E2E | Full user flows | Playwright | Release |

## End-to-End (E2E) Tests

**Location**: `/Users/vikrantbhosale/gitprojects/kitesforu/kitetest`

The `kitetest` project contains Playwright-based E2E tests that verify the complete podcast creation flow through the UI.

### Running E2E Tests

```bash
cd kitetest

# Install dependencies
npm install

# Run staging/beta tests (against beta.kitesforu.com)
npm run test:staging

# Run local tests (requires local dev environment)
npm run test:local
```

### Test Configuration

| Environment | Base URL | Config File |
|-------------|----------|-------------|
| Local | `http://localhost:3000` | `playwright.config.local.ts` |
| Staging | `https://beta.kitesforu.com` | `playwright.config.staging.ts` |

### Test Credentials

Test credentials are configured in `kitetest/.env`:
- **Email**: `test@kitesforu.com`
- **Password**: (see `.env` file)
- **GCP Project**: `kitesforu-dev`

### What E2E Tests Verify

1. **Login via Clerk** - Authentication flow
2. **Create podcast form** - Topic, duration, style selection
3. **Clarification handling** - Answer clarifier questions
4. **Firestore verification** - Job created and tracked
5. **Cloud Logging** - All worker stages execute
6. **Audio URL** - Final audio file is accessible
7. **UI completion** - Progress page shows success

### E2E Test Structure

```
kitetest/
├── tests/
│   ├── local/                    # Local environment tests
│   │   └── podcast-creation.spec.ts
│   ├── staging/                  # Staging/beta tests
│   │   └── podcast-creation.spec.ts
│   └── shared/                   # Shared helpers
│       ├── auth.ts               # Clerk authentication
│       ├── gcp-logger.ts         # Cloud Logging queries
│       ├── firestore-checker.ts  # Firestore verification
│       └── gcs-checker.ts        # Cloud Storage checks
├── config/
│   ├── app-config.json          # Test configuration
│   └── test-loader.ts           # Config loader
├── playwright.config.local.ts    # Local config
├── playwright.config.staging.ts  # Staging config
└── package.json
```

### Prerequisites for Local Tests

Local E2E tests require the full development environment running:
1. Frontend: `cd kitesforu-frontend && npm run dev`
2. API: `cd kitesforu-api && python -m src.api.main`
3. Workers: `cd kitesforu-workers && python -m src.workers.monolith`

Or use the restart script if available:
```bash
./restart.sh
```

---

## API Test Suite

**Location**: `/Users/vikrantbhosale/gitprojects/kitesforu/kitetest/apitests`

The `kitetest` project includes a dedicated API test suite (21 tests) that validates API endpoints using the TEST_API_KEY authentication mechanism.

### What It Tests

| Test File | Coverage |
|-----------|----------|
| `auth.test.ts` | TEST_API_KEY authentication, 401 handling |
| `health.test.ts` | Health check endpoint |
| `credits.test.ts` | Credit balance, tier info |
| `jobs.test.ts` | Job creation, status, listing |

### Running API Tests

```bash
cd kitetest

# Install dependencies
npm install

# Run all API tests
npx playwright test --config=playwright.config.api.ts

# Run specific test file
npx playwright test apitests/auth.test.ts

# Run with verbose output
npx playwright test apitests/ --reporter=list
```

### Configuration

API tests use TEST_API_KEY for authentication (no Clerk JWT needed):

```typescript
// apitests/config.ts
export const config = {
  apiUrl: process.env.API_URL || 'https://kitesforu-api-m6zqve5yda-uc.a.run.app',
  testApiKey: process.env.TEST_API_KEY,  // Required
  testUserId: process.env.TEST_USER_ID || 'test_user_e2e',
};
```

### Environment Setup

1. Get the TEST_API_KEY:
```bash
gcloud secrets versions access latest --secret=TEST_API_KEY --project=kitesforu-dev
```

2. Create `.env` in kitetest directory:
```env
TEST_API_KEY=<key from step 1>
API_URL=https://kitesforu-api-m6zqve5yda-uc.a.run.app
```

### API Test Structure

```
kitetest/
├── apitests/
│   ├── config.ts           # API test configuration
│   ├── api-client.ts       # Typed API client wrapper
│   ├── auth.test.ts        # Authentication tests
│   ├── health.test.ts      # Health check tests
│   ├── credits.test.ts     # Credit system tests
│   └── jobs.test.ts        # Job CRUD tests
├── playwright.config.api.ts # API test config
└── .env                    # TEST_API_KEY (not committed)
```

### Why API Tests vs E2E Tests?

| API Tests | E2E Tests |
|-----------|-----------|
| Fast (~30 seconds) | Slow (5-10 minutes) |
| Uses TEST_API_KEY | Uses Clerk auth |
| Tests API directly | Tests full UI flow |
| CI-friendly | Requires browser |
| Validates contracts | Validates UX |

Run API tests on every PR; run E2E tests for releases.

---

## Repository Testing

### API (kitesforu-api)

```bash
cd kitesforu-api
source .venv/bin/activate

# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=src/api --cov-report=html

# Run specific test file
pytest tests/test_jobs.py -v

# Run specific test
pytest tests/test_jobs.py::test_create_job -v

# Run tests matching pattern
pytest tests/ -k "test_auth" -v
```

**Test Structure**:
```
tests/
├── conftest.py           # Fixtures
├── test_jobs.py          # Job endpoints
├── test_users.py         # User endpoints
├── test_auth.py          # Authentication
└── test_webhooks.py      # Stripe webhooks
```

### Frontend (kitesforu-frontend)

```bash
cd kitesforu-frontend

# Run all tests
npm test

# Run with watch mode
npm test -- --watch

# Run with coverage
npm test -- --coverage

# Run specific test file
npm test -- tests/components/JobForm.test.tsx
```

**Test Structure**:
```
tests/
├── components/           # Component tests
├── pages/               # Page tests
├── hooks/               # Hook tests
└── utils/               # Utility tests
```

### Workers (kitesforu-workers)

```bash
cd kitesforu-workers
source .venv/bin/activate

# Run all tests
pytest tests/ -v

# Run specific stage tests
pytest tests/test_script_stage.py -v

# Run with coverage
pytest tests/ --cov=src/workers --cov-report=html
```

**Test Structure**:
```
tests/
├── conftest.py           # Fixtures
├── test_initiator.py     # Initiator stage
├── test_research.py      # Research stages
├── test_script.py        # Script generation
├── test_audio.py         # Audio generation
└── test_model_router.py  # Model routing
```

### Schemas (kitesforu-schemas)

```bash
cd kitesforu-schemas
source .venv/bin/activate

# Run tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=src/kitesforu_schemas --cov-report=html
```

## Testing Patterns

### API Testing

**Testing endpoints**:
```python
from fastapi.testclient import TestClient
from src.api.main import app

client = TestClient(app)

def test_create_job():
    response = client.post(
        "/api/v1/jobs",
        json={"topic": "Test", "duration_min": 5},
        headers={"Authorization": "Bearer test_token"}
    )
    assert response.status_code == 201
    assert "job_id" in response.json()
```

**Mocking external services**:
```python
from unittest.mock import patch, MagicMock

@patch("src.api.services.firestore.Client")
def test_get_job(mock_firestore):
    mock_doc = MagicMock()
    mock_doc.to_dict.return_value = {"job_id": "123", "status": "completed"}
    mock_firestore.return_value.collection.return_value.document.return_value.get.return_value = mock_doc

    response = client.get("/api/v1/jobs/123")
    assert response.status_code == 200
```

### Worker Testing

**Testing stages**:
```python
from unittest.mock import patch, AsyncMock
from src.workers.stages.script import generate_script

@patch("src.workers.stages.script.call_llm")
async def test_generate_script(mock_llm):
    mock_llm.return_value = "Generated script content"

    result = await generate_script(
        research_results={"topic": "AI"},
        job_config={"duration_min": 5}
    )

    assert "script" in result
    mock_llm.assert_called_once()
```

**Testing model router**:
```python
from src.workers.routing.model_router import ModelRouter

def test_model_selection():
    router = ModelRouter()

    # Mock health status
    router._provider_health = {
        "openai": {"status": "healthy", "latency": 100},
        "anthropic": {"status": "healthy", "latency": 150}
    }

    selected = router.select_model(task_type="script")
    assert selected.provider in ["openai", "anthropic"]
```

### Frontend Testing

**Component testing**:
```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { JobForm } from '@/components/JobForm';

describe('JobForm', () => {
  it('submits form with valid data', async () => {
    const onSubmit = jest.fn();
    render(<JobForm onSubmit={onSubmit} />);

    fireEvent.change(screen.getByLabelText('Topic'), {
      target: { value: 'Test Topic' }
    });
    fireEvent.click(screen.getByRole('button', { name: 'Create' }));

    expect(onSubmit).toHaveBeenCalledWith(
      expect.objectContaining({ topic: 'Test Topic' })
    );
  });
});
```

## Test Fixtures

### Shared Fixtures (conftest.py)

```python
import pytest
from unittest.mock import MagicMock

@pytest.fixture
def mock_firestore():
    """Mock Firestore client"""
    mock = MagicMock()
    return mock

@pytest.fixture
def sample_job():
    """Sample job data"""
    return {
        "job_id": "test-123",
        "user_id": "user-456",
        "topic": "Test Topic",
        "duration_min": 10,
        "status": "queued"
    }

@pytest.fixture
def auth_headers():
    """Valid auth headers for testing"""
    return {"Authorization": "Bearer test_token"}
```

## Mocking External Services

### OpenAI

```python
@patch("openai.ChatCompletion.create")
def test_llm_call(mock_openai):
    mock_openai.return_value = MagicMock(
        choices=[MagicMock(message=MagicMock(content="Response"))]
    )
    # Test code
```

### Firestore

```python
@patch("google.cloud.firestore.Client")
def test_database(mock_client):
    mock_doc = MagicMock()
    mock_doc.exists = True
    mock_doc.to_dict.return_value = {"key": "value"}
    mock_client.return_value.collection.return_value.document.return_value.get.return_value = mock_doc
    # Test code
```

### Pub/Sub

```python
@patch("google.cloud.pubsub_v1.PublisherClient")
def test_publish(mock_publisher):
    mock_future = MagicMock()
    mock_future.result.return_value = "message-id"
    mock_publisher.return_value.publish.return_value = mock_future
    # Test code
```

## CI Integration

Tests run automatically on every push:

```yaml
# .github/workflows/ci.yml
- name: Run tests
  run: |
    pytest tests/ -v --cov=src/ --cov-report=xml

- name: Upload coverage
  uses: codecov/codecov-action@v3
```

## Coverage Requirements

| Service | Minimum Coverage |
|---------|------------------|
| API | 80% |
| Workers | 70% |
| Schemas | 90% |
| Frontend | 60% |

## Related

- [LOCAL_SETUP.md](./LOCAL_SETUP.md)
- [CI_CD.md](./CI_CD.md)
