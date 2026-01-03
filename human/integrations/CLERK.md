# Clerk Authentication Integration

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses [Clerk](https://clerk.com) for authentication, providing user sign-up, sign-in, and JWT verification.

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Frontend  │────▶│    Clerk    │────▶│     API     │
│  (Next.js)  │     │   Service   │     │  (FastAPI)  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                                       │
       │         JWT Token                     │
       └───────────────────────────────────────┘
                  Authorization Header
```

## Clerk Dashboard

Access: https://dashboard.clerk.com

### Application Settings

| Setting | Value |
|---------|-------|
| Application Name | KitesForU |
| Sign-in methods | Email, Google OAuth |
| Session duration | 7 days |

### API Keys

| Key | Usage |
|-----|-------|
| Publishable Key | Frontend (public) |
| Secret Key | Backend verification |

## Frontend Integration

### Setup

```typescript
// app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs'

export default function RootLayout({ children }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  )
}
```

### Environment Variables

```bash
# .env.local
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
```

### Sign In/Up Components

```typescript
// app/sign-in/[[...sign-in]]/page.tsx
import { SignIn } from '@clerk/nextjs'

export default function SignInPage() {
  return <SignIn />
}
```

### Protected Routes

```typescript
// middleware.ts
import { authMiddleware } from '@clerk/nextjs'

export default authMiddleware({
  publicRoutes: ['/', '/sign-in', '/sign-up']
})

export const config = {
  matcher: ['/((?!.*\\..*|_next).*)', '/', '/(api|trpc)(.*)']
}
```

### Getting User Info

```typescript
// In a component
import { useUser, useAuth } from '@clerk/nextjs'

export function UserProfile() {
  const { user, isLoaded } = useUser()
  const { getToken } = useAuth()

  // Get JWT for API calls
  const token = await getToken()

  return <div>Welcome, {user?.firstName}</div>
}
```

## Backend Integration

### JWT Verification

```python
# src/api/auth/clerk.py
from clerk_backend_api import Clerk
from clerk_backend_api.jwks_helpers import authenticate_request

clerk = Clerk(bearer_auth=os.environ.get("CLERK_SECRET_KEY"))

async def verify_token(request: Request) -> dict:
    """Verify Clerk JWT token"""
    request_state = authenticate_request(
        request,
        clerk_client=clerk
    )

    if request_state.is_signed_in:
        return {
            "user_id": request_state.payload.get("sub"),
            "email": request_state.payload.get("email")
        }

    raise HTTPException(status_code=401, detail="Unauthorized")
```

### FastAPI Dependency

```python
# src/api/dependencies.py
from fastapi import Depends, HTTPException, Request
from .auth.clerk import verify_token

async def get_current_user(request: Request):
    """Dependency to get current authenticated user"""
    auth_header = request.headers.get("Authorization")

    if not auth_header or not auth_header.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing token")

    token = auth_header.split(" ")[1]
    return await verify_token(token)
```

### Using in Endpoints

```python
# src/api/routes/jobs.py
from fastapi import APIRouter, Depends
from ..dependencies import get_current_user

router = APIRouter()

@router.post("/jobs")
async def create_job(
    request: CreateJobRequest,
    user: dict = Depends(get_current_user)
):
    user_id = user["user_id"]
    # Create job for user
```

## User Sync

### Webhook for User Events

Clerk can send webhooks for user events:

```python
# src/api/routes/webhooks.py
@router.post("/webhooks/clerk")
async def clerk_webhook(request: Request):
    payload = await request.json()
    event_type = payload.get("type")

    if event_type == "user.created":
        # Create user in Firestore
        user_data = payload["data"]
        await create_user_record(
            user_id=user_data["id"],
            email=user_data["email_addresses"][0]["email_address"]
        )

    elif event_type == "user.deleted":
        # Handle user deletion
        await delete_user_record(payload["data"]["id"])
```

### Webhook Configuration

In Clerk Dashboard:
1. Go to Webhooks
2. Add endpoint: `https://kitesforu-api-m6zqve5yda-uc.a.run.app/api/v1/webhooks/clerk`
3. Select events: `user.created`, `user.updated`, `user.deleted`

## Session Management

### Session Duration

Configure in Clerk Dashboard:
- Session timeout: 7 days
- Inactivity timeout: 24 hours

### Token Refresh

Clerk handles token refresh automatically in the frontend SDK.

## Testing

### Test User

Create test users in Clerk Dashboard for development:
- test@example.com

### Mock Authentication

```python
# tests/conftest.py
import pytest
from unittest.mock import patch

@pytest.fixture
def mock_auth():
    with patch("src.api.dependencies.verify_token") as mock:
        mock.return_value = {
            "user_id": "test_user_123",
            "email": "test@example.com"
        }
        yield mock
```

## Troubleshooting

### "Invalid token" Error

1. Check CLERK_SECRET_KEY is set correctly
2. Verify token hasn't expired
3. Check clock synchronization

### "User not found"

1. Verify webhook is configured
2. Check Firestore for user record
3. Manual sync if needed

### CORS Issues

Ensure Clerk domain is allowed in CORS settings:
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://kitesforu.com", "http://localhost:3000"],
    allow_credentials=True
)
```

## Security Best Practices

1. **Never expose secret key** in frontend
2. **Always verify tokens** on backend
3. **Use HTTPS** in production
4. **Rotate keys** periodically
5. **Monitor** for suspicious activity

## Related

- [AUTHENTICATION.md](../services/api/AUTHENTICATION.md)
- [Clerk Documentation](https://clerk.com/docs)
