# Worker Pipeline Diagram

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Pipeline Overview

```mermaid
flowchart LR
    subgraph Input
        API[API Service]
    end

    subgraph "Worker Pipeline"
        W1[Initiator<br/>job-initiate]
        W2[Research Planner<br/>job-research-planner]
        W3[Tools Executor<br/>job-execute-tools]
        W4[Script Generator<br/>job-script]
        W5[Audio Worker<br/>job-audio]
    end

    subgraph Output
        GCS[(Cloud Storage)]
        Complete((Completed))
    end

    API -->|"Pub/Sub"| W1
    W1 -->|"Pub/Sub"| W2
    W2 -.->|"Wait for Approval"| W2
    W2 -->|"Pub/Sub"| W3
    W3 -->|"Pub/Sub"| W4
    W4 -->|"Pub/Sub"| W5
    W5 --> GCS
    W5 --> Complete
```

## Detailed Worker Flow

```mermaid
flowchart TB
    subgraph Initiator["1. Initiator Worker"]
        I1[Receive Message]
        I2[Load Job from Firestore]
        I3{Topic Ambiguous?}
        I4[Generate Clarifier Questions]
        I5[Wait for User Answers]
        I6[Publish to Research Planner]

        I1 --> I2 --> I3
        I3 -->|Yes| I4 --> I5 --> I6
        I3 -->|No| I6
    end

    subgraph Planner["2. Research Planner Worker"]
        P1[Receive Message]
        P2[Analyze Topic + Answers]
        P3[Generate Research Tasks]
        P4[Estimate Credits & Duration]
        P5[Save Plan to Firestore]
        P6[Set awaiting_user_action]
        P7[Wait for Approval]
        P8[Publish to Tools Executor]

        P1 --> P2 --> P3 --> P4 --> P5 --> P6 --> P7 --> P8
    end

    subgraph Tools["3. Tools Executor Worker"]
        T1[Receive Message]
        T2[Load Research Plan]
        T3[Execute Web Searches]
        T4[Extract URL Content]
        T5[Aggregate Results]
        T6[Save to Firestore]
        T7[Publish to Script Generator]

        T1 --> T2 --> T3 --> T4 --> T5 --> T6 --> T7
    end

    subgraph Script["4. Script Generator Worker"]
        S1[Receive Message]
        S2[Load Research Results]
        S3[Prepare LLM Prompt]
        S4[Generate Podcast Script]
        S5[Validate Script]
        S6[Save to Firestore]
        S7[Publish to Audio Worker]

        S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7
    end

    subgraph Audio["5. Audio Worker"]
        A1[Receive Message]
        A2[Load Script]
        A3[TTS Synthesis per Segment]
        A4[Mix Audio Segments]
        A5[Generate Waveform]
        A6[Upload to GCS]
        A7[Mark Job COMPLETED]

        A1 --> A2 --> A3 --> A4 --> A5 --> A6 --> A7
    end

    Initiator --> Planner --> Tools --> Script --> Audio
```

## Worker Specifications

| Worker | Topic | Timeout | Memory | Budget |
|--------|-------|---------|--------|--------|
| Initiator | job-initiate | 120s | 512Mi | 0% |
| Research Planner | job-research-planner | 180s | 512Mi | 15% |
| Tools Executor | job-execute-tools | 300s | 1Gi | 30% |
| Script Generator | job-script | 480s | 512Mi | 40% |
| Audio Worker | job-audio | 300s | 1Gi | 25% |

## Error Handling

```mermaid
flowchart TD
    Worker[Any Worker]
    Error{Error Type?}
    Transient[Transient Error]
    Permanent[Permanent Error]
    Retry{Retry Count < 5?}
    NACK[NACK Message]
    DeadLetter[Dead Letter Queue]
    FailJob[Mark Job Failed]

    Worker --> Error
    Error -->|Network, Timeout| Transient
    Error -->|Validation, Logic| Permanent
    Transient --> Retry
    Retry -->|Yes| NACK
    Retry -->|No| DeadLetter
    Permanent --> FailJob
    DeadLetter --> FailJob
```
