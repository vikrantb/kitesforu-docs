# Secret Management

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

All secrets are stored in Google Cloud Secret Manager and accessed by services at runtime. No secrets are stored in code or environment files.

## Secret Inventory

### API Keys

| Secret Name | Service | Provider |
|-------------|---------|----------|
| openai-api-key | API, Workers | OpenAI |
| anthropic-api-key | API, Workers | Anthropic |
| google-ai-api-key | Workers | Google AI |
| elevenlabs-api-key | API, Workers | ElevenLabs |
| deepgram-api-key | Workers | Deepgram |

### Authentication

| Secret Name | Service | Provider |
|-------------|---------|----------|
| clerk-secret-key | API, Frontend | Clerk |
| pubsub-verification-token | Workers | Internal |

### Payment

| Secret Name | Service | Provider |
|-------------|---------|----------|
| stripe-secret-key | API | Stripe |
| stripe-webhook-secret | API | Stripe |

### Email

| Secret Name | Service | Provider |
|-------------|---------|----------|
| zoho-smtp-user | API | Zoho |
| zoho-smtp-pass | API | Zoho |

## Access Patterns

### Cloud Run Services

Secrets are mounted as environment variables:

```hcl
# Terraform configuration
env {
  name = "OPENAI_API_KEY"
  value_from {
    secret_key_ref {
      name = "openai-api-key"
      key  = "latest"
    }
  }
}
```

### Local Development

Use `.env` files (never commit):

```bash
# .env.local
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

Or fetch from Secret Manager:

```bash
gcloud secrets versions access latest --secret=openai-api-key
```

## Managing Secrets

### Create a New Secret

```bash
# Create secret
echo -n "secret-value" | gcloud secrets create my-secret --data-file=-

# Or from a file
gcloud secrets create my-secret --data-file=./secret-value.txt
```

### Update a Secret

```bash
# Add a new version
echo -n "new-value" | gcloud secrets versions add my-secret --data-file=-
```

### View Secret Value

```bash
gcloud secrets versions access latest --secret=openai-api-key
```

### List All Secrets

```bash
gcloud secrets list
```

### List Secret Versions

```bash
gcloud secrets versions list openai-api-key
```

## IAM Permissions

### Required Role

Services need `roles/secretmanager.secretAccessor` to read secrets.

### Grant Access

```bash
gcloud secrets add-iam-policy-binding openai-api-key \
  --member="serviceAccount:kitesforu-worker@kitesforu-dev.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Security Best Practices

### Do

1. **Rotate secrets regularly** - Especially API keys
2. **Use latest version** - Don't pin to specific versions
3. **Limit access** - Only grant to services that need it
4. **Audit access** - Review who can read secrets
5. **Use service accounts** - Don't use personal credentials

### Don't

1. **Never commit secrets** - Use `.gitignore`
2. **Never log secrets** - Sanitize logs
3. **Never share via chat** - Use secure channels
4. **Never use in URLs** - Use headers instead
5. **Never hardcode** - Always use Secret Manager

## Troubleshooting

### "Permission Denied" Error

Check service account has access:

```bash
gcloud secrets get-iam-policy openai-api-key
```

Grant access if needed:

```bash
gcloud secrets add-iam-policy-binding openai-api-key \
  --member="serviceAccount:SERVICE_ACCOUNT_EMAIL" \
  --role="roles/secretmanager.secretAccessor"
```

### Secret Not Found

Verify secret exists:

```bash
gcloud secrets describe openai-api-key
```

### Wrong Value

Check you're accessing the right version:

```bash
gcloud secrets versions list openai-api-key
gcloud secrets versions access VERSION_ID --secret=openai-api-key
```

## Rotation Procedure

1. **Create new API key** in provider dashboard
2. **Add as new secret version**:
   ```bash
   echo -n "new-key" | gcloud secrets versions add openai-api-key --data-file=-
   ```
3. **Deploy services** to pick up new version
4. **Verify functionality** with new key
5. **Disable old version** (optional):
   ```bash
   gcloud secrets versions disable VERSION_ID --secret=openai-api-key
   ```
6. **Revoke old key** in provider dashboard
