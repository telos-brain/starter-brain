---
name: "Execution API: File Transcription"
code: BRA410
version: 1
description: How to extract text from uploaded files via the Execution API
  POST /transcription endpoint — supported types, request shape, response
  envelope, error conditions, and how image transcription resolves its vision
  model from Brain settings and environment secrets.
---

# Execution API: File Transcription

See BRA401 for authentication and conventions. See BRA202 for environment
variable / API-key naming (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`).

`POST /transcription` turns a single uploaded file into extracted text. The
file is **never persisted** — it is discarded immediately after extraction.
Harness applications and external integrations use this endpoint when they need
plain text from images or documents before posting to the inbox, appending unit-
of-work context, or feeding a workflow.

The Management API exposes a separate admin-console path
(`POST /brains/{instance}/transcribe`, Clerk-authenticated) that shares the same
`ITranscriptionService`. This skill documents the **Execution API** contract
only.

---

## `POST /transcription`

Authenticated with the brain API key (BRA401). Content type is
`multipart/form-data` with a single file field.

```http
POST /transcription
Authorization: Bearer <brain-api-key>
Content-Type: multipart/form-data

------boundary
Content-Disposition: form-data; name="file"; filename="notes.pdf"
Content-Type: application/pdf

<binary>
------boundary--
```

| Constraint | Value |
|---|---|
| Form field name | `file` |
| Max size | 20 MB (`RequestSizeLimit` + multipart body limit) |
| Persistence | None — extract and discard |

### Success response — `200 OK`

```json
{
  "result": "Extracted plain text…"
}
```

| Field | Notes |
|---|---|
| `result` | Extracted text. Encoding is UTF-8 plain text (or tab-separated rows for XLSX). |

---

## Accepted file types

| Extension | Extraction path |
|---|---|
| `.png`, `.jpg`, `.jpeg`, `.webp` | Vision model (Claude or OpenAI) — see [Image model resolution](#image-model-resolution) |
| `.pdf` | Server-side (PdfPig) — no AI call |
| `.docx` | Server-side (DocumentFormat.OpenXml) — no AI call |
| `.xlsx` | Server-side (ClosedXML) — rows as tab-separated text |
| `.csv`, `.md`, `.markdown` | Plain `StreamReader` — no AI call |

Any other extension returns `400` with an unsupported-type error.

---

## Image model resolution

For image uploads the service resolves a vision provider in this order:

1. **Brain `transcriptionModel`** (set in Brain Settings or via
   `transcription-model` in `brain-compose.yml` — see BRA201 / BRA131) when a
   matching provider API key is available
2. Else **`ANTHROPIC_API_KEY`** → default Claude vision model
3. Else **`OPENAI_API_KEY`** → default OpenAI vision model
4. Else **`400`** — no transcription model available

Document types never require an AI key.

---

## Error responses

Failures use the BRA401 error envelope `{ "error": "…" }`.

| Condition | HTTP | `error` |
|---|---|---|
| Missing / empty `file` | `400` | `No file provided.` |
| File larger than 20 MB | `400` | `File exceeds the 20 MB size limit.` |
| Unsupported extension | `400` | `Unsupported file type: .xyz` (or `(no extension)`) |
| Image with no usable AI key / model | `400` | `No transcription model available. Configure ANTHROPIC_API_KEY or OPENAI_API_KEY, or set a transcription model in Brain Settings.` |
| Extraction / vision call failed | `400` | e.g. `Failed to extract text from .pdf file.` / `Image transcription failed.` |
| Missing or invalid API key | `401` | — (BRA401 auth) |

---

## Known constraint — Tool Router / declared tools

The Tool Router's webhook dispatch **cannot inject HTTP headers**. A Brain
cannot call `POST /transcription` via a declared tool definition — the request
would arrive without `Authorization` and receive `401`. Use this endpoint from
harness code, CLI scripts, or other callers that can send the brain API key.
In-brain workflows that need text from files should receive already-transcribed
content (e.g. via inbox body) rather than calling this HTTP path as a tool.

---

## Endpoint summary

| Method | Path | Purpose | Success |
|---|---|---|---|
| `POST` | `/transcription` | Extract text from one uploaded file | `200` `{ "result": "…" }` |

> Tenancy is implicit (BRA401): the API key scopes the request to one brain.
> Image transcription reads that brain's `transcriptionModel` and environment
> variables; no `brain_id` is accepted in the route or body.

---

## See also

- **BRA401** — authentication, error envelope, common status codes
- **BRA202** — `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` naming and upload rules
- **BRA404** — inbox intake (a common destination for transcribed text)
- **BRA201** — `transcription-model` in `brain-compose.yml`
