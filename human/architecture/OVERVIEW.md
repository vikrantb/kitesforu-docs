# Architecture Overview

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## What is KitesForU?

KitesForU is an AI-powered podcast generation platform. Users provide a topic, and the system automatically researches, scripts, and synthesizes a professional podcast.

## High-Level Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Frontend  │────>│     API     │────>│   Workers   │
│  (Next.js)  │<────│  (FastAPI)  │<────│  (Python)   │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       │                   │                   │
       v                   v                   v
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Clerk    │     │  Firestore  │     │     GCS     │
│    (Auth)   │     │ (Database)  │     │  (Storage)  │
└─────────────┘     └─────────────┘     └─────────────┘
```

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Frontend | Next.js 14 | User interface, SSR |
| API | FastAPI | REST endpoints, business logic |
| Workers | Python | Background processing |
| Database | Firestore | Document storage |
| Messaging | Pub/Sub | Async communication |
| Storage | Cloud Storage | Audio files |
| Auth | Clerk | User authentication |
| Payments | Stripe | Subscriptions, credits |
| AI | OpenAI, Anthropic | LLM, TTS |

## Core Components

### 1. Frontend Service
- **Technology**: Next.js 14 with App Router
- **Hosting**: Cloud Run
- **Responsibilities**:
  - User authentication flow
  - Podcast creation wizard
  - Job status monitoring
  - Audio playback

### 2. API Service
- **Technology**: FastAPI (Python)
- **Hosting**: Cloud Run
- **Responsibilities**:
  - Job management
  - Credit/payment handling
  - User data access
  - Pub/Sub publishing

### 3. Worker Services (5 total)
- **Technology**: Python with FastAPI handlers
- **Hosting**: Cloud Run (separate services)
- **Responsibilities**:
  - Research and planning
  - Script generation
  - TTS synthesis
  - Audio processing

## Data Flow

1. **User submits podcast request** → Frontend
2. **Create job in database** → API → Firestore
3. **Start async processing** → API → Pub/Sub
4. **Worker pipeline executes** → Workers (5 stages)
5. **Audio uploaded** → Workers → Cloud Storage
6. **User notified** → Frontend polls for completion

## Key Design Decisions

### Async Processing
- Workers connected via Pub/Sub
- Each stage is independently scalable
- Failures isolated to specific stages
- Dead-letter queue for debugging

### Model Router
- Intelligent provider selection
- Health monitoring
- Automatic failover
- Quota tracking

### Credit System
- Prepaid credits model
- Per-job cost calculation
- Refunds on failures
- Tier-based limits

## Environment

- **Project**: kitesforu-dev
- **Region**: us-central1
- **Infrastructure**: Terraform managed

## Related Documentation

- [Data Flow](./DATA_FLOW.md) - Detailed request flow
- [Worker Pipeline](./WORKER_PIPELINE.md) - Pipeline stages
- [Diagrams](../diagrams/) - Visual representations
