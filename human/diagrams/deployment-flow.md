# Deployment Flow Diagram

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## CI/CD Pipeline

```mermaid
flowchart LR
    subgraph "Developer"
        Code[Push to main]
    end

    subgraph "GitHub Actions"
        Trigger[Workflow Triggered]
        Lint[Lint & Type Check]
        Test[Run Tests]
        Build[Build Docker Image]
        Push[Push to Artifact Registry]
        Deploy[Deploy to Cloud Run]
    end

    subgraph "GCP"
        AR[(Artifact Registry)]
        CR[Cloud Run]
    end

    Code --> Trigger
    Trigger --> Lint --> Test --> Build --> Push --> Deploy
    Push --> AR
    Deploy --> CR
```

## Deployment Stages

```mermaid
flowchart TD
    subgraph "Stage 1: Lint & Test"
        L1[Install Dependencies]
        L2[Run Linter]
        L3[Type Check]
        L4[Run Unit Tests]
    end

    subgraph "Stage 2: Build"
        B1[Build Docker Image]
        B2[Tag with Git SHA]
        B3[Tag with 'latest']
    end

    subgraph "Stage 3: Push"
        P1[Authenticate to GCR]
        P2[Push to Artifact Registry]
    end

    subgraph "Stage 4: Deploy"
        D1[Deploy to Cloud Run]
        D2[Route 100% Traffic]
        D3[Verify Health Check]
    end

    L1 --> L2 --> L3 --> L4
    L4 --> B1 --> B2 --> B3
    B3 --> P1 --> P2
    P2 --> D1 --> D2 --> D3
```

## Service Deployment Order

```mermaid
flowchart TD
    subgraph "1. Schemas (if changed)"
        S1[Build Python Package]
        S2[Publish to Artifact Registry]
    end

    subgraph "2. API"
        A1[Build API Image]
        A2[Deploy kitesforu-api]
    end

    subgraph "3. Workers"
        W1[Build Workers Image]
        W2[Deploy All 5 Workers]
    end

    subgraph "4. Frontend"
        F1[Build Frontend Image]
        F2[Deploy kitesforu-frontend]
    end

    S1 --> S2 --> A1
    A1 --> A2 --> W1
    W1 --> W2 --> F1
    F1 --> F2
```

## Rollback Flow

```mermaid
flowchart TD
    Issue[Issue Detected]
    Identify[Identify Failing Service]
    ListRev[List Revisions]
    SelectRev[Select Good Revision]
    RouteTraffic[Route 100% Traffic]
    Verify[Verify Health]
    Investigate[Investigate Root Cause]

    Issue --> Identify --> ListRev --> SelectRev --> RouteTraffic --> Verify --> Investigate
```

## Canary Deployment

```mermaid
flowchart LR
    subgraph "Phase 1"
        C1[Deploy New Revision]
        C2[Route 10% Traffic]
    end

    subgraph "Phase 2"
        C3[Monitor 15 min]
        C4{Errors?}
        C5[Route 50% Traffic]
        C6[Rollback]
    end

    subgraph "Phase 3"
        C7[Monitor 15 min]
        C8{Errors?}
        C9[Route 100% Traffic]
    end

    C1 --> C2 --> C3 --> C4
    C4 -->|No| C5 --> C7 --> C8
    C4 -->|Yes| C6
    C8 -->|No| C9
    C8 -->|Yes| C6
```

## Environment Configuration

```mermaid
flowchart TD
    subgraph "Secret Manager"
        SM[GCP Secret Manager]
    end

    subgraph "Environment Variables"
        EV1[Build-time Variables]
        EV2[Runtime Variables]
    end

    subgraph "Cloud Run Service"
        CR[Service Configuration]
    end

    SM -->|"Mounted at Runtime"| CR
    EV1 -->|"Docker Build Args"| CR
    EV2 -->|"--set-env-vars"| CR
```

## Deployment Commands

### Manual Deployment

```bash
# API
cd kitesforu-api
./infra/deploy-api.sh

# Workers
cd kitesforu-workers
./infra/deploy-workers.sh

# Frontend
cd kitesforu-frontend
./infra/deploy-frontend.sh
```

### Verify Deployment

```bash
# List services
gcloud run services list --region=us-central1

# Check active revision
gcloud run services describe kitesforu-api \
  --region=us-central1 \
  --format="value(status.traffic[0].revisionName)"

# Health check
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/v1/health
```

## Artifact Registry

| Repository | Type | Contents |
|------------|------|----------|
| kitesforu-docker | Docker | API, Workers, Frontend images |
| kitesforu-python | Python | kitesforu-schemas package |
