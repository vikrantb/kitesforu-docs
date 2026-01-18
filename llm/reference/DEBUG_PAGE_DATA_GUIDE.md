# Debug Page Data Guide

Reference for understanding the debug API data structure and how it maps to the FlowView UI components.

## Debug API Data Structure Overview

The `/v1/podcasts/{job_id}/debug` endpoint returns a `DebugInfo` object with multiple data sources:

### Primary Data Sources

| Field | Type | Description | UI Usage |
|-------|------|-------------|----------|
| `stages_detailed` | `Record<string, StageDetailed>` | Per-stage status/duration (SPARSE on details) | Stage structure, duration, status |
| `llm_call_logs` | `LLMCallLog[]` | Detailed LLM calls with prompts/tokens | LLM info display (preferred) |
| `tool_call_logs` | `ToolCallLogEntry[]` | Research tool executions | Tool execution details |
| `tts_segment_logs` | `TTSSegmentLogEntry[]` | TTS segment details | Audio segment info (preferred) |
| `research_plan_tasks` | `ResearchPlanTask[]` | LLM-planned research tasks | Research planner output |
| `research_results` | `ResearchResult[]` | Executed research results | Execute tools output |
| `script_preview` | `ScriptPreview` | Script dialogue preview | Script stage output |
| `stage_input_logs` | `StageInputLogEntry[]` | Worker trigger data | **NOT YET DISPLAYED** |

### Data Source Priority

**IMPORTANT**: The detailed log arrays contain RICHER data than the summary fields in `stages_detailed`:

```
PREFER: llm_call_logs > stages_detailed[stage].llm_call
PREFER: tts_segment_logs > stages_detailed[stage].tts_call
```

## Stage Pipeline Flow

```
job-initiate → job-research-planner → job-execute-tools → job-script → job-audio
                      |                        |               |            |
                      v                        v               v            v
              research_plan_tasks      tool_call_logs   script_preview  tts_segment_logs
              llm_call_logs            research_results  llm_call_logs
```

### Stage Data Mapping

| Stage | Input Data | Output Data | Detailed Logs |
|-------|-----------|-------------|---------------|
| job-initiate | topic, duration, language | job_id, validated config | - |
| job-research-planner | topic, duration, language | research_plan_tasks | llm_call_logs (stage=research_planner) |
| job-execute-tools | research_plan_tasks | research_results | tool_call_logs |
| job-script | outline + research_results | script | llm_call_logs (stage=script) |
| job-audio | script dialogue lines | audio file | tts_segment_logs |

## Stage Name Mapping

The API uses different naming conventions:

| stages_detailed key | Log stage field patterns |
|---------------------|--------------------------|
| `job-initiate` | `initiate`, `job-initiate` |
| `job-research-planner` | `research_planner`, `research-planner` |
| `job-plan` (legacy) | `plan`, `job-plan` |
| `job-execute-tools` | `execute_tools`, `execute-tools` |
| `job-script` | `script`, `job-script` |
| `job-audio` | `audio`, `job-audio`, `tts` |

## Current Implementation (FlowView)

### LLM Data Aggregation

```typescript
// Filter llm_call_logs by stage patterns
const llmCalls = getLLMCallsForStage(stageName, debugInfo.llm_call_logs)

// Aggregate tokens and cost from all calls for this stage
const totalTokens = llmCalls.reduce((sum, c) => sum + (c.total_tokens || 0), 0)
const totalCost = llmCalls.reduce((sum, c) => sum + (c.cost || 0), 0)
```

### TTS Data Aggregation

```typescript
// Aggregate from tts_segment_logs for audio stage
const segments = debugInfo.tts_segment_logs
const totalCost = segments.reduce((sum, s) => sum + (s.cost || 0), 0)
const totalDurationMs = segments.reduce((sum, s) => sum + (s.duration_ms || 0), 0)
```

## Data Interfaces

### LLMCallLog (detailed)
```typescript
interface LLMCallLog {
  stage: string           // Stage that made the call
  model: string           // e.g., "gpt-4", "claude-3-opus"
  provider: string        // e.g., "openai", "anthropic"
  input_tokens?: number   // Prompt tokens
  output_tokens?: number  // Completion tokens
  total_tokens?: number   // Total tokens
  cost?: number           // Cost in dollars
  system_prompt: string   // Full system prompt (for debugging)
  user_prompt: string     // Full user prompt (for debugging)
  success: boolean        // Whether call succeeded
  error?: string          // Error message if failed
}
```

### TTSSegmentLogEntry (detailed)
```typescript
interface TTSSegmentLogEntry {
  index: number           // Segment order
  speaker: string         // "Host" or "Guest"
  text_preview: string    // First ~100 chars of text
  voice_name: string      // Voice used
  provider: string        // TTS provider
  language?: string       // Language code
  duration_ms?: number    // Audio duration
  cost?: number           // Cost in dollars
}
```

### StageInputLogEntry (NOT YET USED)
```typescript
interface StageInputLogEntry {
  stage: string                    // e.g., "research_planner"
  worker_class: string             // e.g., "ResearchPlannerWorker"
  message_data: Record<string>     // Pub/Sub message that triggered worker
  job_state_snapshot: Record<string> // Job state at time of execution
  description: string              // Human-readable summary
}
```

## Future Enhancements

### Priority 1: Stage Input Display
- Show what Pub/Sub message triggered each worker
- Display job_state_snapshot for debugging

### Priority 2: LLM Prompt Display
- Add expandable section showing full system_prompt and user_prompt
- Helps understand LLM decision-making

### Priority 3: Tool Results Preview
- Show actual search result snippets from tool_call_logs.results_preview
- Currently only showing results_count

### Priority 4: Error Context
- Show per-LLM-call errors, not just stage-level
- Display retry attempts and escalation levels

## Debugging Tips

1. **LLM shows "unknown" or 0 tokens**: Check if `llm_call_logs` is being filtered correctly by stage patterns
2. **Duration shows "-"**: Check `stages_detailed[stage].duration_seconds`
3. **TTS shows no data**: Check if `tts_segment_logs` array is populated
4. **No research tasks**: Check if `research_plan_tasks` array exists

## Related Files

- Frontend: `components/debug/FlowView.tsx`, `components/debug/FlowCard.tsx`
- API: `api/routes/debug.py` (debug endpoint)
- Backend logging: `workers/common/debug_logging.py`
