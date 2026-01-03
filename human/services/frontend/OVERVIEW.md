# Frontend Service Overview

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Summary

The frontend is a Next.js 14 application providing the user interface for KitesForU. It handles authentication, podcast creation, and result playback.

## Quick Reference

| Attribute | Value |
|-----------|-------|
| Service Name | kitesforu-frontend |
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| URL | https://kitesforu-frontend-m6zqve5yda-uc.a.run.app |
| Port | 3000 |

## Key Features

1. **Authentication** - Clerk integration for sign-in/sign-up
2. **Podcast Creation** - Multi-step creation wizard
3. **Status Tracking** - Real-time job progress
4. **Research Plan Review** - Approve/edit research plans
5. **Audio Playback** - Built-in player for results
6. **Payments** - Stripe checkout integration

## Project Structure

```
kitesforu-frontend/
├── src/
│   ├── app/
│   │   ├── page.tsx              # Landing page
│   │   ├── layout.tsx            # Root layout + Clerk
│   │   ├── dashboard/
│   │   │   └── page.tsx          # User dashboard
│   │   ├── create/
│   │   │   └── page.tsx          # Podcast creation
│   │   ├── jobs/
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Job status & result
│   │   ├── pricing/
│   │   │   └── page.tsx          # Pricing page
│   │   ├── sign-in/
│   │   │   └── [[...sign-in]]/
│   │   │       └── page.tsx
│   │   └── sign-up/
│   │       └── [[...sign-up]]/
│   │           └── page.tsx
│   ├── components/
│   │   ├── ui/                   # shadcn/ui components
│   │   ├── forms/
│   │   ├── layout/
│   │   └── podcast/
│   ├── lib/
│   │   ├── api.ts               # API client
│   │   └── utils.ts
│   └── hooks/
│       ├── useJob.ts            # Job status hook
│       └── useCredits.ts
├── middleware.ts                 # Route protection
├── tailwind.config.ts
└── next.config.js
```

## Key Pages

### Landing Page (`/`)
- Hero section
- Feature highlights
- Pricing preview
- Call-to-action

### Dashboard (`/dashboard`)
- Recent podcasts
- Credit balance
- Quick create button
- Job history

### Create (`/create`)
- Topic input
- Duration selector
- Style picker
- Audience field
- Credit estimate

### Job Status (`/jobs/[id]`)
- Progress indicator
- Stage display
- Clarifier questions
- Research plan review
- Audio player (when complete)

### Pricing (`/pricing`)
- Tier comparison
- Feature matrix
- Checkout buttons

## UI Framework

- **Component Library**: shadcn/ui (Radix primitives)
- **Styling**: Tailwind CSS
- **Icons**: Lucide React
- **Animations**: Framer Motion

## API Integration

### API Client
```typescript
// lib/api.ts
const api = {
  async createJob(data: CreateJobRequest) {
    return fetch(`${API_BASE}/v1/podcasts`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${await getToken()}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(data)
    });
  },

  async getJobStatus(jobId: string) {
    return fetch(`${API_BASE}/v1/podcasts/${jobId}/status`, {
      headers: {
        'Authorization': `Bearer ${await getToken()}`
      }
    });
  }
};
```

### Status Polling
```typescript
// hooks/useJob.ts
function useJob(jobId: string) {
  const { data, error } = useSWR(
    `/v1/podcasts/${jobId}/status`,
    fetcher,
    {
      refreshInterval: 3000,
      // Stop polling when complete
      isPaused: () => data?.status === 'completed'
    }
  );
  return { data, error };
}
```

## Authentication

### Middleware
```typescript
// middleware.ts
import { authMiddleware } from "@clerk/nextjs";

export default authMiddleware({
  publicRoutes: ["/", "/pricing", "/sign-in(.*)", "/sign-up(.*)"],
});
```

### Protected Components
```tsx
import { useAuth } from "@clerk/nextjs";

function Dashboard() {
  const { userId, isLoaded } = useAuth();

  if (!isLoaded) return <Loading />;
  if (!userId) return redirect("/sign-in");

  return <DashboardContent />;
}
```

## Environment Variables

### Build Time
```env
NEXT_PUBLIC_API_BASE=https://kitesforu-api-m6zqve5yda-uc.a.run.app
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
```

### Runtime
```env
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_PREVIEW_MODE=false
```

## Deployment

### Cloud Run Configuration
- Memory: 512Mi
- CPU: 1
- Min instances: 0
- Max instances: 10

### Deploy Command
```bash
cd kitesforu-frontend
./infra/deploy-frontend.sh
```

### Build Args
```dockerfile
ARG NEXT_PUBLIC_API_BASE
ARG NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
```
