# Debug Page

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

The debug page (`/debug/[jobId]`) provides detailed visibility into podcast generation jobs for troubleshooting and quality analysis.

**URL Pattern**: `https://beta.kitesforu.com/debug/<job-id>`

## Features

### Job Header
- Status badge (processing, completed, failed)
- Language, duration, estimated cost
- Topic summary

### Execution Timeline
Interactive timeline showing all operations:
- **Stage inputs**: What triggered each worker
- **LLM calls**: Model, tokens, cost, prompts/responses
- **Tool calls**: Search queries, results, duration
- **TTS calls**: Voice, text, duration

### Research Flow
- Research tasks with rationale
- Results per task with URLs and snippets
- Success/failure status per task

### Sections
- **Errors**: Any errors encountered during processing
- **Stage Accordion**: Expand each stage for detailed view
- **Script Preview**: Generated podcast script
- **Outline**: Structured outline from planning
- **Outputs**: Audio player, download links
- **Raw JSON**: Full job data dump for debugging

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `E` | Expand all stages |
| `C` | Collapse all stages |
| `R` | Refresh data |

## Export Function

Click "Export JSON" button to download the complete debug data as a JSON file for offline analysis.

## Data Sources

The debug page fetches data from:

```
GET /api/jobs/{jobId}/debug
```

Response includes:
- `job_id`, `status`, `topic`
- `stages_detailed`: Per-stage breakdown
- `execution_timeline`: Chronological events
- `research_flow`: Research tasks and results
- `errors`: Any error messages
- `outputs`: Audio URLs, script, outline

## Firestore Collections

Debug data is sourced from:

| Collection | Data |
|------------|------|
| `podcast_jobs` | Main job document |
| `podcast_jobs/{id}/debug_logs` | Execution timeline events |
| `podcast_jobs/{id}/llm_calls` | LLM call details |
| `podcast_jobs/{id}/tool_calls` | Tool execution logs |

## Use Cases

### Investigating Failed Jobs
1. Navigate to `/debug/<job-id>`
2. Check "Errors" section for immediate issues
3. Expand failed stage to see detailed logs
4. Check tool calls for external API failures

### Quality Analysis
1. Review LLM prompts to understand inputs
2. Compare research results vs final script
3. Check TTS segment details for audio issues

### Cost Optimization
1. View per-stage cost breakdown
2. Analyze LLM token usage
3. Identify expensive operations

## Access Control

Debug page requires authentication. Only authenticated users can access debug data for jobs they own.

## Related

- [API Debug Endpoint](../../services/api/ENDPOINTS.md)
- [Worker Pipeline](../../architecture/OVERVIEW.md)
- [Troubleshooting](../../operations/TROUBLESHOOTING.md)
