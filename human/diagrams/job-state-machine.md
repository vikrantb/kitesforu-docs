# Job State Machine Diagram

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Job Status States

```mermaid
stateDiagram-v2
    [*] --> queued: Job Created

    queued --> clarifying: Needs User Input
    queued --> running: Auto-start

    clarifying --> queued: Answers Received
    clarifying --> running: Plan Approved

    running --> completed: Success
    running --> failed: Error

    completed --> [*]
    failed --> [*]

    note right of queued
        Initial state after job creation
    end note

    note right of clarifying
        Waiting for clarifier answers
        or research plan approval
    end note

    note right of running
        Active processing through
        worker pipeline
    end note
```

## Job Stage Progression

```mermaid
stateDiagram-v2
    [*] --> queued

    queued --> research: Start Processing
    research --> script: Research Complete
    script --> voice: Script Generated
    voice --> mix: TTS Complete
    mix --> publish: Mixing Complete
    publish --> completed: Upload Success

    research --> failed: Error
    script --> failed: Error
    voice --> failed: Error
    mix --> failed: Error
    publish --> failed: Error

    completed --> [*]
    failed --> [*]
```

## Research Plan Approval Flow

```mermaid
stateDiagram-v2
    [*] --> pending_approval: Plan Generated

    pending_approval --> approved: User Approves
    pending_approval --> editing: User Edits

    editing --> pending_approval: Edit Submitted
    editing --> cancelled: User Cancels

    approved --> [*]: Continue to Execution
    cancelled --> [*]: Job Cancelled

    note right of pending_approval
        User can:
        - Approve as-is
        - Edit (max 1 time)
        - Cancel
    end note

    note right of editing
        Only 1 edit allowed
        per research plan
    end note
```

## Combined State View

```mermaid
flowchart TD
    subgraph "Job Creation"
        Create[Create Job]
        Publish[Publish to Pub/Sub]
    end

    subgraph "Initialization Phase"
        Init[Initiator Worker]
        Clarify{Need Clarification?}
        WaitAnswers[Wait for Answers]
    end

    subgraph "Planning Phase"
        Plan[Research Planner]
        WaitApproval[Wait for Approval]
        UserApprove{User Action}
    end

    subgraph "Execution Phase"
        Tools[Tools Executor]
        Script[Script Generator]
        Audio[Audio Worker]
    end

    subgraph "Completion"
        Success[Job COMPLETED]
        Fail[Job FAILED]
    end

    Create --> Publish --> Init
    Init --> Clarify
    Clarify -->|Yes| WaitAnswers --> Init
    Clarify -->|No| Plan
    Plan --> WaitApproval --> UserApprove
    UserApprove -->|Approve| Tools
    UserApprove -->|Edit| Plan
    UserApprove -->|Cancel| Fail
    Tools --> Script --> Audio --> Success

    Init -.->|Error| Fail
    Plan -.->|Error| Fail
    Tools -.->|Error| Fail
    Script -.->|Error| Fail
    Audio -.->|Error| Fail
```

## State Descriptions

### JobStatus Values

| Status | Description | User Action |
|--------|-------------|-------------|
| `queued` | Job waiting to start | None |
| `clarifying` | Needs user input | Submit answers or approve plan |
| `running` | Active processing | Wait for completion |
| `completed` | Finished successfully | Play/download audio |
| `failed` | Error occurred | Check error, credits refunded |

### JobStage Values

| Stage | Description | Worker |
|-------|-------------|--------|
| `queued` | Initial state | - |
| `research` | Research in progress | Initiator, Planner, Tools |
| `script` | Script generation | Script Generator |
| `voice` | TTS synthesis | Audio Worker |
| `mix` | Audio mixing | Audio Worker |
| `publish` | Uploading to GCS | Audio Worker |
| `completed` | Done | - |
| `failed` | Error state | - |
