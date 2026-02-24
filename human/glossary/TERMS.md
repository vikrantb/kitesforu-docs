# Glossary of Terms

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Content Entities

See [ENTITIES.md](./ENTITIES.md) for canonical names and definitions of everything a user can create (Quick Audio, Audio Series, Classroom, Interview Prep, Writing).

## Domain Terms

### Job
A single podcast generation request. Contains topic, duration, style, and tracks progress through the pipeline.

### Stage
A step in the podcast generation pipeline. Stages include: research, script, voice, mix, publish.

### Research Plan
A structured plan for gathering information about a topic. Contains questions to answer, sources to search, and tools to use.

### Research Plan Approval
Feature where users review and approve the research plan before execution proceeds.

### Script
The generated text content for a podcast episode. Includes dialogue, transitions, and speaker assignments.

### Credit
Virtual currency used to pay for podcast generation. Different durations cost different amounts of credits.

## Technical Terms

### Cloud Run
Google Cloud's serverless container platform. Runs the API, frontend, and workers.

### Pub/Sub
Google Cloud's message queue service. Used for async communication between workers.

### Firestore
Google Cloud's NoSQL document database. Stores jobs, users, and transactions.

### Dead Letter Queue (DLQ)
A queue where failed messages are sent after exceeding retry attempts.

### Revision
A specific version of a Cloud Run service. Used for deployments and rollbacks.

### OIDC
OpenID Connect. Authentication protocol used for Pub/Sub push authentication.

## Status Terms

### JobStatus
The overall status of a job:
- **queued**: Job created, waiting to start
- **clarifying**: Waiting for user input (research plan approval)
- **running**: Currently being processed
- **completed**: Successfully finished
- **failed**: Error occurred

### JobStage
The current processing stage:
- **queued**: Not started
- **research**: Gathering information
- **script**: Generating script
- **voice**: Creating audio
- **mix**: Combining audio segments
- **publish**: Uploading final audio
- **completed**: All done
- **failed**: Error state

### HealthStatus
Provider health status:
- **healthy**: Provider is available and performant
- **degraded**: Provider is slow or partially available
- **unavailable**: Provider is not responding

## Architecture Terms

### Worker
A Cloud Run service that processes a specific stage of the pipeline. Triggered by Pub/Sub messages.

### Model Router
Component that selects the best LLM provider based on health, latency, and cost.

### Pipeline
The sequential flow of workers that process a job from start to finish.

## Integration Terms

### Clerk
Authentication service that handles user sign-up, sign-in, and JWT verification.

### Stripe
Payment processing service that handles credit purchases and subscriptions.

### ElevenLabs
Text-to-speech service that generates high-quality voice audio.

### OpenAI
AI provider for language models (GPT-4) and text-to-speech.

### Anthropic
AI provider for language models (Claude).

## API Terms

### Endpoint
A specific URL path that handles a particular type of request.

### JWT
JSON Web Token. Used for authentication between frontend and API.

### Bearer Token
Authentication token passed in the Authorization header.

### Webhook
HTTP callback that receives event notifications from external services.

## Development Terms

### Artifact Registry
Google Cloud service for storing Docker images and Python packages.

### Terraform
Infrastructure as Code tool for managing GCP resources.

### CI/CD
Continuous Integration/Continuous Deployment. Automated testing and deployment pipeline.

### Hot Reload
Development feature that automatically restarts the server when code changes.

## Operations Terms

### Rollback
Reverting to a previous version of a service after a failed deployment.

### Incident
An event that disrupts normal service operation.

### Runbook
Step-by-step procedures for common operational tasks.

### SLA
Service Level Agreement. Defines expected availability and performance.

### Observability
The ability to understand system behavior through logs, metrics, and traces.

## Data Terms

### Collection
A group of documents in Firestore (similar to a table).

### Document
A single record in Firestore (similar to a row).

### Transaction
An atomic operation that either completes entirely or not at all.

### TTL
Time To Live. Automatic deletion of old data after a specified period.

## Abbreviations

| Abbreviation | Full Term |
|--------------|-----------|
| API | Application Programming Interface |
| CLI | Command Line Interface |
| CRUD | Create, Read, Update, Delete |
| DLQ | Dead Letter Queue |
| GCP | Google Cloud Platform |
| IAM | Identity and Access Management |
| JWT | JSON Web Token |
| LLM | Large Language Model |
| OIDC | OpenID Connect |
| PR | Pull Request |
| SDK | Software Development Kit |
| SSR | Server-Side Rendering |
| TTS | Text-to-Speech |
| UUID | Universally Unique Identifier |
