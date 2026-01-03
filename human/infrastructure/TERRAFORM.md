# Terraform Infrastructure

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

All GCP infrastructure is defined as code using Terraform in the `kitesforu-infrastructure` repository.

## Repository Location

```
/Users/vikrantbhosale/gitprojects/kitesforu/kitesforu-infrastructure/
```

## Directory Structure

```
kitesforu-infrastructure/
├── terraform/
│   ├── main.tf              # Provider config, backend
│   ├── variables.tf         # Input variables
│   ├── outputs.tf           # Output values
│   ├── cloud_run.tf         # Cloud Run services
│   ├── pubsub.tf            # Pub/Sub topics & subscriptions
│   ├── pubsub_workers.tf    # Worker-specific Pub/Sub config
│   ├── firestore.tf         # Firestore database
│   ├── storage.tf           # GCS buckets
│   ├── iam.tf               # Service accounts & roles
│   ├── secrets.tf           # Secret Manager resources
│   └── artifact_registry.tf # Docker & Python repos
├── environments/
│   └── dev/
│       └── terraform.tfvars # Dev environment values
└── README.md
```

## Key Files

### main.tf
```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }

  backend "gcs" {
    bucket = "kitesforu-terraform-state"
    prefix = "terraform/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

### variables.tf
```hcl
variable "project_id" {
  description = "GCP project ID"
  type        = string
}

variable "region" {
  description = "GCP region"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}
```

### cloud_run.tf (example)
```hcl
resource "google_cloud_run_service" "api" {
  name     = "kitesforu-api"
  location = var.region

  template {
    spec {
      containers {
        image = "${var.region}-docker.pkg.dev/${var.project_id}/kitesforu-docker/kitesforu-api:latest"

        resources {
          limits = {
            memory = "512Mi"
            cpu    = "1"
          }
        }

        env {
          name  = "GOOGLE_PROJECT_ID"
          value = var.project_id
        }

        # Secret environment variables
        env {
          name = "OPENAI_API_KEY"
          value_from {
            secret_key_ref {
              name = "openai-api-key"
              key  = "latest"
            }
          }
        }
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}
```

## Terraform Commands

### Initialize
```bash
cd kitesforu-infrastructure/terraform
terraform init
```

### Plan Changes
```bash
terraform plan -var-file=../environments/dev/terraform.tfvars
```

### Apply Changes
```bash
terraform apply -var-file=../environments/dev/terraform.tfvars
```

### View State
```bash
terraform state list
terraform state show google_cloud_run_service.api
```

## State Management

- **Backend**: GCS bucket `kitesforu-terraform-state`
- **State Locking**: Enabled via GCS
- **State Encryption**: Server-side encryption

## Resource Naming Convention

```
{project}-{service}-{optional-suffix}
```

Examples:
- `kitesforu-api`
- `kitesforu-worker-script`
- `kitesforu-podcasts` (bucket)

## Import Existing Resources

If a resource was created manually:

```bash
# Import Cloud Run service
terraform import google_cloud_run_service.api \
  locations/us-central1/services/kitesforu-api

# Import Pub/Sub topic
terraform import google_pubsub_topic.job_initiate \
  projects/kitesforu-dev/topics/job-initiate
```

## Outputs

```hcl
output "api_url" {
  value = google_cloud_run_service.api.status[0].url
}

output "frontend_url" {
  value = google_cloud_run_service.frontend.status[0].url
}
```

## Best Practices

1. **Never edit resources manually** - All changes through Terraform
2. **Review plans carefully** - Check what will be destroyed
3. **Use workspaces** - Separate state per environment
4. **Lock versions** - Pin provider versions
5. **Document changes** - Commit messages should explain why
