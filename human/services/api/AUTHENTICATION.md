# Authentication

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses Clerk for authentication. The frontend handles user sign-in/sign-up, and the API verifies JWT tokens on every request.

## How It Works

### 1. User Signs In (Frontend)

```
User → Clerk UI Component → Clerk Servers → Session Token
```

The frontend uses Clerk's React components:
- `<SignIn />` for sign-in
- `<SignUp />` for registration
- `<UserButton />` for user menu

### 2. Frontend Makes API Request

```javascript
// Frontend code
const response = await fetch('/v1/podcasts', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${clerkToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ topic, duration_min, style })
});
```

### 3. API Verifies Token

```python
# API code (simplified)
async def verify_token(authorization: str):
    token = authorization.replace("Bearer ", "")

    # Fetch JWKS from Clerk
    jwks = await fetch_jwks(CLERK_JWKS_URL)

    # Verify signature
    payload = jwt.decode(token, jwks)

    # Extract user_id
    user_id = payload["sub"]

    return user_id
```

## JWT Structure

Clerk JWTs contain:

```json
{
  "sub": "user_2abc123",          // User ID
  "iss": "https://clerk.dev",     // Issuer
  "exp": 1705312800,              // Expiration
  "iat": 1705309200,              // Issued at
  "azp": "pk_test_...",           // Authorized party
  "sid": "sess_abc123"            // Session ID
}
```

## Configuration

### Frontend (.env)
```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
```

### API (Secret Manager)
```
CLERK_JWKS_URL=https://[your-clerk-domain]/.well-known/jwks.json
```

## Protected Routes

### Frontend (middleware.ts)
```typescript
import { authMiddleware } from "@clerk/nextjs";

export default authMiddleware({
  publicRoutes: ["/", "/pricing", "/sign-in", "/sign-up"],
});
```

### API (dependency injection)
```python
async def get_current_user(
    authorization: str = Header(...)
) -> str:
    return await verify_token(authorization)

@app.get("/v1/me/credits")
async def get_credits(user_id: str = Depends(get_current_user)):
    return await credits_service.get_balance(user_id)
```

## Error Responses

| Status | Cause | Solution |
|--------|-------|----------|
| 401 | Missing token | User needs to sign in |
| 401 | Expired token | Clerk auto-refreshes |
| 401 | Invalid signature | Check Clerk config |
| 403 | Wrong user | User accessing another's resource |

## Token Refresh

Clerk automatically refreshes tokens before expiry:

1. Token approaches expiration
2. Clerk SDK detects this
3. Refresh request sent to Clerk
4. New token received
5. Subsequent requests use new token

## Security Best Practices

1. **Never log tokens** - They contain sensitive data
2. **Always verify server-side** - Don't trust client claims
3. **Use HTTPS only** - Tokens must be encrypted in transit
4. **Check expiration** - Reject expired tokens
5. **Validate issuer** - Ensure token is from your Clerk instance

## Testing

### Get a Test Token
1. Sign in via frontend
2. Open browser DevTools
3. Check Application → Cookies → `__session`
4. Or use Clerk's `getToken()` method

### Test API Endpoint
```bash
curl -H "Authorization: Bearer <token>" \
  https://kitesforu-api-m6zqve5yda-uc.a.run.app/v1/me/credits
```
