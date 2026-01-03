# Stripe Payment Integration

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

KitesForU uses [Stripe](https://stripe.com) for payment processing, handling subscription management and credit purchases.

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Frontend  │────▶│   Stripe    │────▶│     API     │
│  Checkout   │     │   Service   │     │  Webhooks   │
└─────────────┘     └─────────────┘     └─────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │  Firestore  │
                                        │  (credits)  │
                                        └─────────────┘
```

## Stripe Dashboard

Access: https://dashboard.stripe.com

### Products

| Product | Price | Credits |
|---------|-------|---------|
| Starter Pack | $9.99 | 100 credits |
| Pro Pack | $29.99 | 500 credits |
| Enterprise Pack | $99.99 | 2000 credits |

### API Keys

| Key | Usage |
|-----|-------|
| Publishable Key | Frontend checkout |
| Secret Key | Backend operations |
| Webhook Secret | Webhook verification |

## Implementation

### Frontend Checkout

```typescript
// components/Checkout.tsx
import { loadStripe } from '@stripe/stripe-js'

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!)

export async function redirectToCheckout(priceId: string) {
  const response = await fetch('/api/create-checkout-session', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ priceId })
  })

  const { sessionId } = await response.json()
  const stripe = await stripePromise
  await stripe?.redirectToCheckout({ sessionId })
}
```

### Backend - Create Session

```python
# src/api/routes/payments.py
import stripe
from fastapi import APIRouter, Depends

stripe.api_key = os.environ.get("STRIPE_SECRET_KEY")

router = APIRouter()

@router.post("/create-checkout-session")
async def create_checkout_session(
    price_id: str,
    user: dict = Depends(get_current_user)
):
    # Get or create Stripe customer
    customer_id = await get_or_create_customer(user["user_id"], user["email"])

    session = stripe.checkout.Session.create(
        customer=customer_id,
        payment_method_types=["card"],
        line_items=[{"price": price_id, "quantity": 1}],
        mode="payment",
        success_url=f"{BASE_URL}/payment/success?session_id={{CHECKOUT_SESSION_ID}}",
        cancel_url=f"{BASE_URL}/payment/cancelled",
        metadata={"user_id": user["user_id"]}
    )

    return {"sessionId": session.id}
```

### Webhook Handler

```python
# src/api/routes/webhooks.py
@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig_header = request.headers.get("stripe-signature")

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, os.environ.get("STRIPE_WEBHOOK_SECRET")
        )
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid payload")
    except stripe.error.SignatureVerificationError:
        raise HTTPException(status_code=400, detail="Invalid signature")

    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]
        await handle_successful_payment(session)

    elif event["type"] == "payment_intent.payment_failed":
        payment_intent = event["data"]["object"]
        await handle_failed_payment(payment_intent)

    return {"status": "success"}
```

### Credit Fulfillment

```python
# src/api/services/payments.py
async def handle_successful_payment(session: dict):
    user_id = session["metadata"]["user_id"]
    line_items = stripe.checkout.Session.list_line_items(session["id"])

    for item in line_items["data"]:
        price_id = item["price"]["id"]
        credits = PRICE_TO_CREDITS.get(price_id, 0)

        # Add credits to user
        await add_credits(user_id, credits)

        # Record transaction
        await create_credit_transaction(
            user_id=user_id,
            type="purchase",
            amount=credits,
            stripe_session_id=session["id"]
        )

PRICE_TO_CREDITS = {
    "price_starter": 100,
    "price_pro": 500,
    "price_enterprise": 2000
}
```

## Webhook Configuration

### Setup in Stripe Dashboard

1. Go to Developers > Webhooks
2. Add endpoint: `https://kitesforu-api-m6zqve5yda-uc.a.run.app/api/v1/webhooks/stripe`
3. Select events:
   - `checkout.session.completed`
   - `payment_intent.payment_failed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`

### Local Testing

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward webhooks to local server
stripe listen --forward-to localhost:8000/api/v1/webhooks/stripe
```

## Credit System

### Credit Deduction

```python
# src/api/services/credits.py
async def deduct_credits(user_id: str, amount: int, job_id: str):
    """Deduct credits for job processing"""
    user_ref = db.collection("users").document(user_id)

    @firestore.transactional
    def update_in_transaction(transaction):
        user = user_ref.get(transaction=transaction)
        current_balance = user.to_dict().get("credits_balance", 0)

        if current_balance < amount:
            raise InsufficientCreditsError()

        new_balance = current_balance - amount
        transaction.update(user_ref, {"credits_balance": new_balance})

        # Record transaction
        trans_ref = db.collection("credit_transactions").document()
        transaction.set(trans_ref, {
            "user_id": user_id,
            "type": "deduction",
            "amount": -amount,
            "balance_after": new_balance,
            "job_id": job_id,
            "created_at": firestore.SERVER_TIMESTAMP
        })

        return new_balance

    return update_in_transaction(db.transaction())
```

### Credit Pricing

| Duration | Credits Required |
|----------|------------------|
| < 5 min | 5 credits |
| 5-15 min | 15 credits |
| 15-30 min | 30 credits |
| 30-60 min | 60 credits |
| > 60 min | 2 credits/min |

## Environment Variables

| Variable | Description |
|----------|-------------|
| STRIPE_SECRET_KEY | Stripe secret key |
| STRIPE_WEBHOOK_SECRET | Webhook signing secret |
| NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY | Public key for frontend |

## Testing

### Test Cards

| Card Number | Result |
|-------------|--------|
| 4242424242424242 | Success |
| 4000000000000002 | Declined |
| 4000000000009995 | Insufficient funds |

### Test Mode

Use test API keys from Stripe Dashboard for development.

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `card_declined` | Card rejected | User should try another card |
| `insufficient_funds` | Not enough balance | User should add funds |
| `webhook_signature_invalid` | Wrong secret | Check STRIPE_WEBHOOK_SECRET |

### Retry Logic

```python
# Handle webhook idempotency
async def handle_successful_payment(session: dict):
    # Check if already processed
    existing = await get_transaction_by_session(session["id"])
    if existing:
        return  # Already processed

    # Process payment...
```

## Security

1. **Verify webhooks** using signing secret
2. **Use HTTPS** for all endpoints
3. **Never log** sensitive card data
4. **Use idempotency keys** for retries
5. **PCI compliance** - Stripe handles card data

## Refunds

```python
async def process_refund(charge_id: str, amount: int = None):
    refund = stripe.Refund.create(
        charge=charge_id,
        amount=amount  # None for full refund
    )

    # Deduct credits from user
    # Record refund transaction
```

## Related

- [Stripe Documentation](https://stripe.com/docs)
- [ENDPOINTS.md](../services/api/ENDPOINTS.md)
