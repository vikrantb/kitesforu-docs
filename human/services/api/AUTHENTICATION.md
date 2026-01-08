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

---

## Test API Key Authentication

For E2E and integration testing, KitesForU supports a simplified authentication method using a static API key. This bypasses Clerk JWT validation entirely.

### When to Use

- **E2E Tests**: Automated Playwright tests that need to call the API
- **Integration Tests**: API test suites (kitetest/apitests)
- **Local Development**: Faster iteration without real Clerk tokens

### Configuration

#### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `TEST_API_KEY` | The secret test key value | `test-key-abc123` |
| `ALLOW_TEST_API_KEY` | Enable test key in cloud | `true` |
| `TEST_USER_ID` | User ID for test requests | `test_user_e2e` |
| `TEST_USER_EMAIL` | Email for test requests | `test@kitesforu.com` |

#### Where It Works

| Environment | Condition |
|-------------|-----------|
| Local DEV | Always enabled (DEV mode auto-detected) |
| Cloud DEV | Requires `ALLOW_TEST_API_KEY=true` |
| Production | **NEVER** - rejected automatically |

### How It Works

```
Test Request → API receives TEST_API_KEY as Bearer token
           → Checks if TEST_API_KEY matches environment
           → If match AND allowed: returns TEST_USER_ID
           → If match but not allowed: 401 Unauthorized
```

### Usage Example

```bash
# Using test API key instead of Clerk JWT
curl -H "Authorization: Bearer $TEST_API_KEY" \
  https://kitesforu-api-dev-m6zqve5yda-uc.a.run.app/v1/me/credits
```

### Test Authentication Flow (Code)

```python
# In src/api/auth/clerk.py
async def verify_clerk_token(token: str) -> str:
    # Check for TEST_API_KEY authentication (for E2E testing)
    # Allowed in DEV mode OR when ALLOW_TEST_API_KEY=true
    test_api_key = os.getenv("TEST_API_KEY")
    allow_test_key = is_dev_mode() or os.getenv("ALLOW_TEST_API_KEY", "").lower() == "true"

    if test_api_key and token == test_api_key:
        if allow_test_key:
            logger.info("TEST_API_KEY authentication successful (E2E testing)")
            return os.getenv("TEST_USER_ID", "test_user_e2e")
        else:
            raise JWTError("Test API key not allowed in this environment")

    # Fall through to standard Clerk JWT verification...
```

### Quota Bypass

When using TEST_API_KEY authentication with `ALLOW_TEST_API_KEY=true`, credit quotas are automatically bypassed:

- No credit balance checks
- No free tier limits
- Jobs created without deducting credits

This enables unlimited E2E testing without managing test account credits.

### Security Considerations

1. **Never commit TEST_API_KEY** - Store in Secret Manager or env files
2. **Never enable in production** - ALLOW_TEST_API_KEY must be false
3. **Use unique keys per environment** - Different keys for dev/staging
4. **Rotate regularly** - Change keys if exposed
5. **Monitor usage** - Log all TEST_API_KEY authentications

### CI/CD Configuration

The dev environment deploys with test authentication enabled:

```yaml
# .github/workflows/ci.yml
--set-secrets="...,TEST_API_KEY=TEST_API_KEY:latest" \
--update-env-vars="ALLOW_TEST_API_KEY=true"
```

### Related

- [Testing Guide](../../development/TESTING.md) - How to run E2E and API tests
- [API Overview](./OVERVIEW.md) - All environment variables
