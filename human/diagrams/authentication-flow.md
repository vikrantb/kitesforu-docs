# Authentication Flow Diagram

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses Clerk for authentication with JWT tokens verified by the API.

## Sign-In Flow

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant CL as Clerk
    participant API as API

    U->>FE: Navigate to /sign-in
    FE->>CL: Load Clerk SignIn Component
    U->>CL: Enter Credentials
    CL->>CL: Authenticate User
    CL-->>FE: Session Token
    FE->>FE: Store Token (httpOnly cookie)
    FE-->>U: Redirect to /dashboard

    Note over FE,API: Subsequent API Requests

    FE->>API: GET /v1/me/credits<br/>Authorization: Bearer {token}
    API->>CL: Fetch JWKS
    CL-->>API: Public Keys
    API->>API: Verify JWT Signature
    API->>API: Extract user_id from 'sub' claim
    API-->>FE: User Data
```

## JWT Verification

```mermaid
flowchart TD
    Request[API Request with Bearer Token]
    Extract[Extract JWT from Header]
    Fetch[Fetch JWKS from Clerk]
    Verify[Verify Signature]
    Check[Check Expiration]
    Claims[Extract Claims]
    Success[Allow Request]
    Fail[401 Unauthorized]

    Request --> Extract
    Extract --> Fetch
    Fetch --> Verify
    Verify -->|Valid| Check
    Verify -->|Invalid| Fail
    Check -->|Not Expired| Claims
    Check -->|Expired| Fail
    Claims --> Success
```

## Protected Routes

```mermaid
flowchart LR
    subgraph "Public Routes"
        P1["/"]
        P2["/pricing"]
        P3["/sign-in"]
        P4["/sign-up"]
    end

    subgraph "Protected Routes"
        R1["/dashboard"]
        R2["/create"]
        R3["/jobs/*"]
    end

    subgraph "Middleware"
        MW[Clerk Middleware]
    end

    P1 --> MW
    P2 --> MW
    P3 --> MW
    P4 --> MW
    R1 --> MW
    R2 --> MW
    R3 --> MW

    MW -->|No Auth| P1
    MW -->|No Auth| P2
    MW -->|No Auth| P3
    MW -->|No Auth| P4
    MW -->|Auth Required| R1
    MW -->|Auth Required| R2
    MW -->|Auth Required| R3
```

## Token Lifecycle

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant CL as Clerk
    participant API as API

    Note over FE,CL: Token Refresh (Automatic)

    FE->>CL: Token Near Expiry
    CL->>CL: Validate Session
    CL-->>FE: New Token

    Note over FE,API: API Request with Fresh Token

    FE->>API: Request with New Token
    API->>API: Verify Token
    API-->>FE: Response
```

## Components

### Frontend (Next.js)

```
- ClerkProvider: Wraps app in _app.tsx
- SignIn/SignUp: Clerk-hosted or embedded components
- UserButton: User menu with sign-out
- middleware.ts: Route protection
```

### API (FastAPI)

```
- Auth middleware validates JWT on every request
- Extracts user_id from 'sub' claim
- Caches JWKS for performance
- Returns 401 if token invalid/expired
```

## Configuration

### Frontend Environment

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
```

### API Environment

```env
CLERK_JWKS_URL=https://[clerk-domain]/.well-known/jwks.json
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 401 Unauthorized | Missing/invalid token | Re-authenticate |
| Token Expired | JWT past expiry | Clerk auto-refreshes |
| Invalid Signature | JWKS mismatch | Check Clerk config |
