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
