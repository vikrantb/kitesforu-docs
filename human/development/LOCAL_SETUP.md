# Local Development Setup

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

Guide to setting up KitesForU services for local development.

## Prerequisites

| Requirement | Version | Installation |
|-------------|---------|--------------|
| Python | 3.11+ | `brew install python@3.11` |
| Node.js | 18+ | `brew install node@18` |
| Docker | Latest | [Docker Desktop](https://docker.com/desktop) |
| gcloud CLI | Latest | `brew install google-cloud-sdk` |
| Git | Latest | `brew install git` |

## Repository Structure

```
kitesforu/
├── kitesforu-api/          # FastAPI backend
├── kitesforu-frontend/     # Next.js frontend
├── kitesforu-workers/      # Python workers
├── kitesforu-schemas/      # Shared schemas
├── kitesforu-infrastructure/  # Terraform
├── kitesforu-support/      # Debug scripts
└── kitesforu-docs/         # Documentation
```

## Initial Setup

### 1. Clone Repositories

```bash
cd ~/gitprojects
mkdir kitesforu && cd kitesforu

git clone git@github.com:vikrantb/kitesforu-api.git
git clone git@github.com:vikrantb/kitesforu-frontend.git
git clone git@github.com:vikrantb/kitesforu-workers.git
git clone git@github.com:vikrantb/kitesforu-schemas.git
```

### 2. Configure GCP

```bash
# Login to GCP
gcloud auth login
gcloud auth application-default login

# Set project
gcloud config set project kitesforu-dev
```

### 3. Get Secrets for Local Dev

Create `.env` files with secrets:

```bash
# API secrets
cd kitesforu-api
cat > .env << 'EOF'
OPENAI_API_KEY=$(gcloud secrets versions access latest --secret=openai-api-key)
ANTHROPIC_API_KEY=$(gcloud secrets versions access latest --secret=anthropic-api-key)
CLERK_SECRET_KEY=$(gcloud secrets versions access latest --secret=clerk-secret-key)
STRIPE_SECRET_KEY=$(gcloud secrets versions access latest --secret=stripe-secret-key)
GOOGLE_PROJECT_ID=kitesforu-dev
EOF

# Or fetch individually
gcloud secrets versions access latest --secret=openai-api-key
```

## Service Setup

### API (FastAPI)

```bash
cd kitesforu-api

# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Install local schemas
pip install -e ../kitesforu-schemas

# Copy env file
cp .env.example .env
# Edit .env with your secrets

# Run development server
uvicorn src.api.main:app --reload --port 8000
```

API available at: http://localhost:8000
Docs at: http://localhost:8000/docs

### Frontend (Next.js)

```bash
cd kitesforu-frontend

# Install dependencies
npm install

# Copy env file
cp .env.example .env.local
# Edit .env.local with your values

# Required environment variables:
# NEXT_PUBLIC_API_URL=http://localhost:8000
# NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
# CLERK_SECRET_KEY=sk_test_...

# Run development server
npm run dev
```

Frontend available at: http://localhost:3000

### Workers

Workers are triggered by Pub/Sub and typically run in Cloud Run. For local testing:

```bash
cd kitesforu-workers

# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Install local schemas
pip install -e ../kitesforu-schemas

# Run specific worker locally (for testing)
WORKER_TYPE=initiator python -m src.workers.main

# Or test a single stage
python -c "from src.workers.stages.script import generate_script; ..."
```

### Schemas Package

```bash
cd kitesforu-schemas

# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install in development mode
pip install -e .

# Run tests
pytest tests/ -v
```

## Environment Variables

### API

| Variable | Description | Required |
|----------|-------------|----------|
| OPENAI_API_KEY | OpenAI API key | Yes |
| ANTHROPIC_API_KEY | Anthropic API key | Yes |
| CLERK_SECRET_KEY | Clerk secret key | Yes |
| STRIPE_SECRET_KEY | Stripe secret key | Yes |
| STRIPE_WEBHOOK_SECRET | Stripe webhook secret | For payments |
| GOOGLE_PROJECT_ID | GCP project ID | Yes |
| ELEVENLABS_API_KEY | ElevenLabs API key | For TTS |

### Frontend

| Variable | Description | Required |
|----------|-------------|----------|
| NEXT_PUBLIC_API_URL | Backend API URL | Yes |
| NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY | Clerk public key | Yes |
| CLERK_SECRET_KEY | Clerk secret key | Yes |

### Workers

| Variable | Description | Required |
|----------|-------------|----------|
| WORKER_TYPE | Worker type to run | Yes |
| GOOGLE_PROJECT_ID | GCP project ID | Yes |
| OPENAI_API_KEY | OpenAI API key | Yes |
| ANTHROPIC_API_KEY | Anthropic API key | Yes |
| ELEVENLABS_API_KEY | ElevenLabs API key | Yes |

## Running Full Stack Locally

### Option 1: Individual Services

1. Terminal 1: API
   ```bash
   cd kitesforu-api && source .venv/bin/activate
   uvicorn src.api.main:app --reload --port 8000
   ```

2. Terminal 2: Frontend
   ```bash
   cd kitesforu-frontend
   npm run dev
   ```

3. Workers run in Cloud Run (use deployed services)

### Option 2: Docker Compose

```bash
# From project root
docker-compose up

# Or specific services
docker-compose up api frontend
```

## Testing Against Cloud Services

Local services can connect to cloud resources:

- **Firestore**: Uses Application Default Credentials
- **Pub/Sub**: Use deployed topics/subscriptions
- **Cloud Storage**: Use dev buckets

```bash
# Ensure you're authenticated
gcloud auth application-default login
```

## Common Issues

### Python Import Errors

```bash
# Ensure schemas package is installed
pip install -e ../kitesforu-schemas
```

### Clerk Auth Not Working

```bash
# Check env variables are set
echo $NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
echo $CLERK_SECRET_KEY
```

### Can't Connect to Firestore

```bash
# Check authentication
gcloud auth application-default print-access-token

# Verify project
gcloud config get-value project
```

### Port Already in Use

```bash
# Find and kill process on port 8000
lsof -i :8000
kill -9 PID
```

## IDE Setup

### VS Code

Recommended extensions:
- Python (Microsoft)
- Pylance
- ESLint
- Prettier
- Tailwind CSS IntelliSense

### PyCharm

1. Set Python interpreter to `.venv`
2. Mark `src/` as Sources Root
3. Enable pytest as test runner

## Related

- [TESTING.md](./TESTING.md)
- [CI_CD.md](./CI_CD.md)
