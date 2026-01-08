# API Service Overview

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Summary

The API service is the central hub for KitesForU, handling all client requests, job management, and external integrations.

## Quick Reference

| Attribute | Value |
|-----------|-------|
| Service Name | kitesforu-api |
| Framework | FastAPI |
| Language | Python 3.11+ |
| URL | https://kitesforu-api-m6zqve5yda-uc.a.run.app |
| Port | 8080 |

## Responsibilities

1. **Job Management**
   - Create new podcast jobs
   - Track job status
   - Return results

2. **Authentication**
   - Verify Clerk JWT tokens
   - Extract user identity

3. **Credit System**
   - Calculate job costs
   - Deduct credits
   - Issue refunds

4. **Payment Integration**
   - Stripe checkout sessions
   - Webhook handling
   - Subscription management

5. **Pipeline Coordination**
   - Publish to Pub/Sub
   - Handle clarifier answers
   - Research plan approval

## Project Structure

```
kitesforu-api/
├── src/api/
│   ├── main.py              # FastAPI app entry
│   ├── routes/
│   │   ├── health.py        # Health check
│   │   ├── podcasts.py      # Job endpoints
│   │   ├── users.py         # User endpoints
│   │   ├── pricing.py       # Pricing calculator
│   │   ├── checkout.py      # Stripe integration
│   │   └── webhooks.py      # Stripe webhooks
│   ├── services/
│   │   ├── auth.py          # Clerk JWT
│   │   ├── firestore_service.py
│   │   ├── pubsub_service.py
│   │   └── credits.py
│   └── middleware/
│       └── error_handler.py
├── tests/
├── Dockerfile
└── requirements.txt
```

## Dependencies

### Internal
- **kitesforu-schemas**: Shared Pydantic models

### External Services
- **Clerk**: Authentication
- **Stripe**: Payments
- **Firestore**: Database
- **Pub/Sub**: Messaging

## Environment Variables

### Required
- `OPENAI_API_KEY`
- `CLERK_JWKS_URL`

### Optional
- `ANTHROPIC_API_KEY`
- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `FRONTEND_URL`
- `LOG_LEVEL`

### Testing (E2E/Integration)
| Variable | Description | Default |
|----------|-------------|---------|
| `TEST_API_KEY` | Static API key for test authentication | None |
| `ALLOW_TEST_API_KEY` | Enable TEST_API_KEY in cloud environments | `false` |
| `TEST_USER_ID` | User ID returned for test auth | `test_user_e2e` |
| `TEST_USER_EMAIL` | Email returned for test auth | `test@kitesforu.com` |

See [Authentication](./AUTHENTICATION.md#test-api-key-authentication) for details.

### Cloud Run
- `GCS_BUCKET`
- `PUBSUB_TOPIC`
- `GOOGLE_PROJECT_ID`

## Deployment

### Cloud Run Configuration
- Memory: 512Mi
- CPU: 1
- Min instances: 0
- Max instances: 10

### Deploy Command
```bash
cd kitesforu-api
./infra/deploy-api.sh
```

## Health Check

```bash
curl https://kitesforu-api-m6zqve5yda-uc.a.run.app/v1/health
# {"status": "ok"}
```

## Related Documentation

- [Endpoints](./ENDPOINTS.md) - Complete endpoint reference
- [Authentication](./AUTHENTICATION.md) - Clerk JWT flow
