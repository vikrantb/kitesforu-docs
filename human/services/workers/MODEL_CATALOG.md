# Model Catalog

> **FOR HUMAN CONSUMPTION ONLY** - AI agents should skip this folder

## Overview

The Model Catalog defines all available AI models, their capabilities, costs, and configuration. It supports **dynamic updates via Google Sheets** for easy management without code deployments.

## Architecture

```
┌─────────────────────────────────┐
│      Google Sheets              │
│  (MODEL_CATALOG_SHEET_ID)       │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│      TTL Cache (600s)           │
│      Thread-safe RLock          │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│      Model Router               │
│      (model_catalog list)       │
└─────────────────────────────────┘
              │
              ▼ (if Sheets fails)
┌─────────────────────────────────┐
│      CSV Fallback               │
│      config/model_catalog.csv   │
└─────────────────────────────────┘
```

## Data Source Priority

1. **Google Sheets** (primary) - If `MODEL_CATALOG_SHEET_ID` is configured
2. **CSV Fallback** - If Sheets unavailable or not configured

## Google Sheets Setup

### Prerequisites
- Google Cloud project with Sheets API enabled
- Service account with Viewer access to the Sheet

### Steps

1. **Create Google Sheet**
   - Create new spreadsheet in Google Drive
   - Copy columns from `config/model_catalog.csv`
   - Name first sheet "models"

2. **Share with Service Accounts**

   Share the sheet (Viewer permission) with these service accounts:

   | Service Account | Purpose |
   |-----------------|---------|
   | `kitesforu-worker@kitesforu-dev.iam.gserviceaccount.com` | Cloud Run workers (all 6 workers) |
   | `github-actions@kitesforu-dev.iam.gserviceaccount.com` | CI/CD pipelines |

   > **Note**: Uncheck "Notify people" when sharing (service accounts can't receive emails)

3. **Configure Secret**
   ```bash
   # Get spreadsheet ID from URL
   # https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit

   echo -n "YOUR_SPREADSHEET_ID" | gcloud secrets versions add model-catalog-sheet-id \
     --project=kitesforu-dev \
     --data-file=-
   ```

### Current Production Sheet

- **Spreadsheet**: [model_catalog](https://docs.google.com/spreadsheets/d/1vMUy8DiybcG1dAWU7TJpuZCFC_pYQxbewPdHTToxOi8/edit)
- **Sheet ID**: `1vMUy8DiybcG1dAWU7TJpuZCFC_pYQxbewPdHTToxOi8`
- **Tab**: `models`

## Model Catalog Schema

### Required Columns

| Column | Type | Description |
|--------|------|-------------|
| `model_id` | string | Unique identifier (e.g., `gpt-4o`) |
| `provider` | string | Provider name (`openai`, `anthropic`, `google`, `elevenlabs`) |
| `name` | string | Display name |
| `task_types` | string | Pipe-separated list (`llm`, `tts`, `transcription`) |
| `enabled` | boolean | `TRUE`/`FALSE` |

### Optional Columns

| Column | Type | Description |
|--------|------|-------------|
| `max_input_length` | integer | Max input tokens/characters |
| `supported_languages` | string | Comma-separated language codes |
| `supports_streaming` | boolean | Streaming support |
| `context_window` | integer | Total context window |
| `cost_per_unit` | float | Cost per unit |
| `unit_description` | string | Unit description (e.g., "per 1M tokens") |
| `notes` | string | Additional notes |
| `tier_default` | string | Default user tiers |
| `purpose_priority` | string | Purpose:priority mappings |

### Example Row

```csv
model_id,provider,name,task_types,enabled,cost_per_unit,tier_default
gpt-4o,openai,GPT-4o,llm,TRUE,2.50,creator|pro_creator
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `MODEL_CATALOG_SHEET_ID` | Google Sheets spreadsheet ID | (none) |
| `MODEL_CATALOG_CACHE_TTL` | Cache TTL in seconds | 600 |

### Secret Manager

The spreadsheet ID is stored in Secret Manager:
- **Secret**: `model-catalog-sheet-id`
- **Access**: All 6 workers via Cloud Run env injection

## Caching

### TTL Cache Behavior

- **Default TTL**: 600 seconds (10 minutes)
- **Thread-safe**: Uses RLock for concurrent access
- **Lazy loading**: Catalog loaded on first access
- **Auto-refresh**: Refreshes after TTL expires

### Cache Info API

```python
from workers.routing.catalog import get_catalog_cache

cache = get_catalog_cache()
info = cache.get_cache_info()

# Returns:
# {
#   "source": "sheets",      # or "csv"
#   "model_count": 15,
#   "is_expired": false,
#   "last_refresh": "2024-01-15T10:30:00Z"
# }
```

## Fallback Behavior

### When Sheets Fails

If Google Sheets is unavailable:
1. Error is logged with warning
2. System falls back to `config/model_catalog.csv`
3. Cache reports `source: "csv"`
4. Normal operation continues

### Force CSV Usage

To force CSV usage:
- Set `MODEL_CATALOG_SHEET_ID` to empty string
- Or delete the secret version

## Operations

### Check Current Source

```python
from workers.routing.catalog import get_catalog_cache

cache = get_catalog_cache()
print(f"Source: {cache.get_cache_info()['source']}")
```

### Force Refresh

```python
from workers.routing.catalog import reset_catalog_cache

reset_catalog_cache()  # Clears cache, forces reload on next access
```

### View Loaded Models

```python
from workers.routing.catalog import get_catalog_cache

cache = get_catalog_cache()
catalog = cache.get_catalog()

for model_id, model in catalog.items():
    print(f"{model_id}: {model.provider} - {model.enabled}")
```

## Updating Models

### Via Google Sheets (Recommended)

1. Open the Google Sheet
2. Edit model rows (add, modify, delete)
3. Wait for cache TTL (10 min max)
4. Changes automatically propagate

### Via CSV (Fallback)

1. Edit `config/model_catalog.csv`
2. Commit and push changes
3. Deploy workers
4. Changes take effect immediately

## Troubleshooting

### "Sheets not configured"

```
MODEL_CATALOG_SHEET_ID not set or empty
```

**Fix**: Set the secret with a valid spreadsheet ID

### "Cannot access spreadsheet"

```
gspread.exceptions.SpreadsheetNotFound
```

**Fix**:
1. Verify spreadsheet ID is correct
2. Ensure service account has Viewer access
3. Check Sheet tab is named "models"

### "Invalid CSV format"

```
Missing required column: model_id
```

**Fix**: Ensure all required columns exist with correct names

## Integration Tests

Run integration tests:

```bash
pytest tests/integration/test_sheets_integration.py -v
```

Tests verify:
- Sheets client configuration
- Cache loading from Sheets
- CSV fallback behavior
- Router integration
- Environment configuration

## Related Documentation

- [Model Router](./MODEL_ROUTER.md) - How models are selected
- [Pipeline](./PIPELINE.md) - Worker pipeline overview
