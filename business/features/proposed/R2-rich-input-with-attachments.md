# R2 — Rich-Text Input with Inline File Attachments

**Status**: PROPOSED (no phases shipped yet; Phase 1 is the next pickup)
**Owner**: Smart Create surface (frontend + API + extraction service)
**Priority**: P2
**Effort**: phased across ~4 weeks (Phase 1 → Phase 4; each independently shippable)
**Origin**: 2026-04-19 product-owner review — "we dont have the user a way to copy paste or upload files… I would love if this is a rich text type of thing where user can properly provide attachments inline."

## Pickup pointer (2026-04-24)

Phase 1 (text-only Tiptap composer, ~1 week, frontend-only, zero risk to existing flow) is the right next pickup. Tiptap is the default tech pick (open question 13.1) — confirm before kickoff. RichComposer feature flag (`feature_rich_composer_live`) is currently OFF in production; Phase 1 ships behind the same flag and flips ON only after a 50-creation sanity review. No api / workers / schemas / extraction-service work needed for Phase 1. Phase 2 (PDF / DOCX extraction with presigned GCS) is where the multi-repo coordination begins — needs a follow-up proposal pass before code.

## 1. Problem

Today's Smart Create surface gives the user two disjoint lanes:

- A `Say it` textarea (`app/create-smart/page.tsx:159-166`, rendered via `components/smart-create/IntentSection.tsx:262-334`), capped at 2000 chars with no file awareness.
- A separate `I have notes` tab (`IntentSection.tsx:337-406`) that supports drag-drop and a hidden file picker, but files are parsed **client-side** into a flat concatenated blob (`IntentSection.tsx:162-170`, via `lib/file-parser.ts`) and sent as `pasted_content` to `/v1/smart-create`.

The product owner's ask:

> "we dont have [given] the user a way to copy paste or upload files … I would love if this is a rich text type of thing where user can properly provide attachments inline with like even copy paste … the reference is there and then next do bla bla …"

The composer must behave like a modern rich-text box (Notion, Linear, ChatGPT) where files live **as chips inside the prose**, and everything serializes into one deterministic LLM prompt.

Current limits (`lib/file-config.ts:1-17`): 5 MB/file, 3 files, 50K chars total, extensions `.txt / .pdf / .docx / .rtf / .md / .csv`. These are reasonable but the UX around them is not.

## 2. Goals / Non-goals

**Goals**

- Single composer that accepts prose + inline file chips + inline images.
- OS-agnostic clipboard paste (macOS Cmd+V, Windows Ctrl+V, Linux middle-click) for PNG/JPEG, PDF bytes, DOCX, plain text, and screenshot data URLs from Chrome/Safari/Firefox.
- Drag-and-drop anywhere inside the composer, not just the paste tab.
- Deterministic assembly into the final LLM prompt via a slot grammar.
- Reject obviously abusive uploads **before** they cost us LLM tokens.

**Non-goals**

- Collaborative editing (no cursors, no CRDT).
- Full Markdown authoring for the *user's* prose — we only need light formatting (bold, bullets, inline chip). The user types natural language; the LLM handles the rest.
- Audio/video attachments (out of scope for R2; R3 candidate).

## 3. Current surface — file:line references

| Concern | Location |
| --- | --- |
| Composer entry point | `app/create-smart/page.tsx:159-166` |
| Textarea + attach button | `components/smart-create/IntentSection.tsx:262-334` |
| Drop zone | `components/smart-create/IntentSection.tsx:337-370` |
| Client-side file parsing | `lib/file-parser.ts` (called at `IntentSection.tsx:103`) |
| Limits | `lib/file-config.ts:1-17` |
| Payload flattening | `buildFileContent` at `IntentSection.tsx:162-170` |
| Intake API consumer | `kitesforu-api/src/services/smart_create/` (receives `pasted_content`) |
| Generic LLM extraction route | `kitesforu-api/src/api/routes/extraction/extract.py:35-57` |

No server-side upload/attachment endpoint exists today (grep in `kitesforu-api/src/api/routes/` shows only transcribe/SCORM/stream matches). The API currently only accepts *already-extracted text*.

## 4. Proposed UX

Replace `IntentSection` tabs `prompt` + `paste` with a single composer backed by a lightweight rich-text editor (Tiptap or Lexical; Tiptap is the default pick — smaller bundle, better React 18 story, already mature ProseMirror under the hood).

**Inline chip behavior**

User types: `First read this `, pastes a PDF from clipboard, continues `and then write a 5-minute summary`. The editor renders:

```
First read this [📎 resume.pdf · 142 KB · ✓ extracted] and then
write a 5-minute summary
```

The chip is a non-editable inline node with id, filename, size, mime, extraction status, and a remove button. Removing the chip removes the attachment from the payload.

**Interactions**

- `Cmd/Ctrl+V` with file bytes on clipboard → insert chip at caret.
- `Cmd/Ctrl+V` with text → insert as plain text (no chip).
- Drag-drop onto composer → insert chip at drop point.
- Click paperclip button → file picker → chip appended at end.
- Chip click → sheet with filename, size, extracted-text preview, replace, remove.

**Limits (proposed — up from current)**

| Limit | Value | Rationale |
| --- | --- | --- |
| Max file size | **10 MB** | Covers most resumes, JDs, longer PDFs |
| Max files per composer | **5** | Up from 3; keeps prompt budget sane |
| Max total size | **25 MB** | Guards GCS egress + extraction cost |
| Max extracted chars per file | **40K** | Per-file truncation |
| Max total extracted chars | **120K** | Keeps final prompt under ~150K tokens |
| Allowed extensions | `.txt .md .csv .rtf .pdf .docx .png .jpg .jpeg .webp` | Docs + images for OCR |

Updates `lib/file-config.ts` — current 50K total cap is tight once OCR is in play.

## 5. Architecture

```
┌──────────────────┐   multipart    ┌──────────────────────┐
│  RichComposer    │ ─────────────> │ POST /v1/attachments │
│  (Tiptap)        │                │  (presigned GCS PUT) │
│  - chips carry   │ <───────────── │  returns {id, uri}   │
│    attachment_id │    JSON        └──────────┬───────────┘
└────────┬─────────┘                           │ Pub/Sub
         │ submit                              ▼
         │ { body_richtext, attachments[] } ┌──────────────────────┐
         ▼                                   │ extraction worker    │
┌──────────────────┐                         │ (OCR / pdf / docx)   │
│ POST /v1/smart-  │ ◄─────────────────────  │ writes extracted_text│
│ create/intake    │  resolves slots          │ to Firestore         │
└──────────────────┘                          └──────────────────────┘
```

**Frontend**

- `components/smart-create/RichComposer/` (new) — Tiptap editor + custom `AttachmentNode` inline extension.
- `lib/attachments/client.ts` (new) — presigned upload, progress, abort.
- Keep `lib/file-parser.ts` for a **text-only fast path** in Phase 1 (so .txt/.md don't need a server round-trip).

**API**

- `POST /v1/attachments` (new) — returns presigned GCS URL + attachment id. Enforces per-user rate limit (SlowAPI, body param NOT named `request` — global Python rule).
- `POST /v1/attachments/{id}/finalize` — client calls after GCS PUT; triggers Pub/Sub job.
- `GET /v1/attachments/{id}` — polling/status for extraction.
- Extend `smart_create.py` intake to accept `{ body_richtext, attachments }` and resolve slots server-side before calling the existing intake prompt.

**Extraction service** (new package: `services/attachments/`)

- `txt/md/csv/rtf` → pass-through read.
- `pdf` → `pypdf` for text, falls back to OCR (`pytesseract` or Document AI) if text layer is empty.
- `docx` → `python-docx`.
- `png/jpg/webp` → Document AI OCR (preferred) or `pytesseract`.

Pydantic data model on read follows the "permissive" rule from `kitesforu-api/CLAUDE.md` #2: `ConfigDict(extra='ignore')` on the model **and** nested models; no `max_length` on `List` fields holding LLM content.

## 6. Data model

**Firestore:** `users/{user_id}/creation_attachments/{attachment_id}`

```python
class CreationAttachment(BaseModel):
    model_config = ConfigDict(extra='ignore')

    id: str                         # ULID
    user_id: str
    filename: str
    content_type: str               # mime
    size_bytes: int
    storage_uri: str                # gs://kitesforu-uploads/{user}/{id}
    sha256: str
    extraction_status: Literal[     # use cast() per global rule
        'pending', 'extracting', 'ready', 'failed', 'blocked'
    ]
    extracted_text_preview: str     # first 500 chars, for UI
    extracted_text_len: int         # full length lives in GCS
    extraction_uri: Optional[str]   # gs://.../extracted.txt
    blocked_reason: Optional[str]   # 'mime_mismatch' | 'av_positive' | ...
    created_at: datetime
    expires_at: datetime            # 30-day TTL sweep
```

**GCS:** `gs://kitesforu-uploads/{user_id}/{attachment_id}/{filename}` — lifecycle rule deletes after 30 days unless the parent job is retained.

## 7. Prompt assembly (slot grammar)

The composer serializes to:

```
body_richtext: "First read this {{ATTACHMENT:01J…:resume.pdf:pdf}}
                and then create a 5-minute podcast."
attachments:   [ { id: "01J…", … }, … ]
```

The API resolves each slot server-side (never trust client-extracted text — cost estimate + AV must run server-side):

```
First read this

<attachment id="01J…" filename="resume.pdf" type="pdf">
[extracted text body, truncated to per-file cap]
</attachment>

and then create a 5-minute podcast.
```

Key rules (and why — see `~/.claude/rules/pipeline-integration.md` #3 about template variable placeholders):

- If the prompt template forgets the `{body_richtext}` placeholder, the substitution silently yields nothing. Add a test that greps every relevant template for the placeholder.
- If `attachments` is empty, the slot syntax must still pass through; no empty `<attachment>` blocks.
- User prose is never rewritten — we only *surround* attachments with a quoted XML-ish block. This keeps the LLM's understanding of "first / then / next" intact.

## 8. Abuse & cost guards

- Extension allow-list + mime sniff (`python-magic`) on the server — client-declared mime is advisory only.
- Per-user rate limit: 30 uploads/hour.
- ClamAV sidecar on the extraction worker (cheap; runs before text extraction). Positive → `blocked_reason: 'av_positive'`, chip shows red.
- Cost estimate: after extraction, compute projected prompt tokens (`tiktoken` for OpenAI, rough char/4 fallback). If > user's per-job cap, the Create button shows `Estimated cost: $0.12 — confirm?` modal.
- Dedupe via `sha256`: if the same user re-uploads identical bytes, reuse the existing attachment row.

## 9. Phased scope

**Phase 1 — text-only rich composer (1 week)**
- Tiptap editor replaces the two textareas.
- Chip only supports `text/plain` paste (already works today, just prettier).
- No server upload. Ships zero risk to existing flow.

**Phase 2 — PDF / DOCX extraction (1.5 weeks)**
- `POST /v1/attachments` + presigned GCS.
- Extraction worker handles `pdf`, `docx`, `rtf`, `md`, `csv`, `txt`.
- Intake resolves slots.

**Phase 3 — OCR for images (1 week)**
- Route `image/*` through Document AI (or pytesseract dev fallback).
- Chip shows `OCR 87% confidence` badge.

**Phase 4 — rich formatting (0.5 week)**
- Bold, italic, bullet list, ordered list. Serialized as Markdown into `body_richtext`.
- Keyboard shortcuts `Cmd+B`, `Cmd+I`, `Cmd+Shift+7/8`.

## 10. Acceptance criteria

- [ ] User can paste a PNG screenshot from clipboard on macOS, Windows, and Linux; a chip appears inline at the caret within 200 ms.
- [ ] Drag-drop a 4-file batch onto the composer → 4 chips inserted in drop order; the 5th dropped file is rejected with a toast.
- [ ] Each chip displays filename, size, extraction status, and a remove button; remove drops the attachment from the final payload.
- [ ] A 12 MB PDF is rejected client-side with "File too large (max 10 MB per file)" and never hits the server.
- [ ] `.exe`, `.zip`, `.mp4` are rejected with the allow-list message.
- [ ] Prose + chips serialize into a single `body_richtext` string with `{{ATTACHMENT:id:name:type}}` slots at chip positions.
- [ ] Intake resolves every slot into a quoted `<attachment …>` block; unresolved slots cause a 422 (not a silent drop).
- [ ] Extraction is durable: a failed OCR does not block the other attachments; the failed chip shows red and the user can retry.
- [ ] Total extracted characters exceeding 120K surfaces a pre-submit "trim or remove an attachment" modal.
- [ ] Estimated cost is shown before submit when the projected prompt cost exceeds $0.05.
- [ ] Uploaded file bytes disappear from GCS 30 days after creation unless the parent job is retained.
- [ ] Playwright E2E on `beta.kitesforu.com`: paste PDF → chip appears → submit → job runs → Studio shows "resume.pdf extracted (2.1K chars)" in the debug panel.

## 11. Risks

- **Bundle size.** Tiptap + ProseMirror adds ~120 KB gzipped. Lazy-load the composer; keep `/` and other pages untouched.
- **iOS Safari clipboard file access** is gated — paste of image bytes works on 16.4+ but not earlier. Fallback: show "tap to attach" hint.
- **Silent template drift.** If a prompt template adds a new variable and we forget to thread `body_richtext`, the LLM sees an empty slot (see pipeline-integration rule #3). Mitigation: unit test asserts every Smart Create template contains `{body_richtext}`.
- **Extraction cost.** OCR on a 20-page scanned PDF is measurably expensive. Phase 3 must ship with a per-user monthly OCR cap (e.g. 50 pages on free tier).
- **AV false positives** on legitimate PDFs from some compilers. Provide a "request review" path rather than a hard block.

## 12. Out of scope

- Google Drive / Dropbox pickers.
- Audio / video transcription as attachments (already covered by `routes/transcribe.py`; a later PR can bridge them in as chips).
- Collaborative multi-user composing.
- Inline @mentions of past creations.

## 13. Open questions

1. Tiptap vs Lexical — default proposal is Tiptap; confirm before Phase 1 kickoff.
2. Presigned GCS vs proxying through Cloud Run — proxying is simpler but burns Cloud Run CPU on large PDFs. Default: presigned.
3. Keep the legacy `pasted_content` field for one release as a back-compat bridge, or cut over atomically?
