---
name: Transcribe Image System Tool
code: BRA412
version: 1
description: The transcribe_image system tool — load an image from a URL and
  extract text via the brain's vision transcription service (Claude or OpenAI).
  Accepts an optional prompt to guide what to extract; omit it to transcribe
  all visible text. Complements the file-upload Execution API in BRA410.
tools:
  - transcribe_image
---

# Transcribe Image System Tool

BRA410 covers the **Execution API** file-upload path (`POST /transcription`).
This skill covers the complementary **AI-facing system tool**: a running brain
fetching an image from a URL and returning the transcript — no webhook URL or
API key on the tool call.

This is an ordinary `system` tool (BRA201 §5.2). Declaration lives under
`tools/system-tools/transcribe-image.yml`. This skill lists it in frontmatter
`tools:` so a workflow that keeps it under `available-tools` can promote it via
`get_skill`.

**Scope:** always the current Brain (`BrainId` harness-injected). Never pass a
brain id. Image model resolution uses that brain's `transcriptionModel` and
environment secrets (same order as BRA410).

---

## `transcribe_image`

| | |
| --- | --- |
| **Purpose** | Load an image from a URL and return the transcribed text |
| **When** | The image is already hosted at a public HTTPS URL and the workflow needs its text (or a guided extraction) |
| **YAML** | `tools/system-tools/transcribe-image.yml` |

### Parameters

| Parameter | Required | Notes |
| --- | --- | --- |
| `url` | Yes | Public **HTTPS** URL of the image. Supported types: **png, jpg, jpeg, webp**. Private, loopback, and metadata addresses are blocked. |
| `prompt` | No | Instruction for the vision model describing what to extract. When omitted, the default is used: *Extract all text content from this image. Return only the transcribed text with no commentary.* |

### Behaviour

1. Validates `url` (absolute HTTPS; SSRF guard — no private / loopback / metadata IPs).
2. Fetches the image (no redirect following; 20 MB cap).
3. Sends the image to the brain's vision provider with `prompt`, or the default when `prompt` is omitted.
4. Returns the transcript as plain text.

Document types (PDF, DOCX, …) are **not** accepted here — this tool is image-only. Use BRA410's upload endpoint for documents.

### Returns

The extracted text on success. On failure, a plain-English error, for example:

- Missing `url` / invalid JSON arguments
- URL blocked for safety
- Failed to load the image from the URL
- Unsupported image type
- File exceeds the 20 MB size limit
- No transcription model available (missing `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`, or no `transcription-model`)
- Image transcription failed / returned no text

### When to use

- A workflow is given an image URL (receipt, screenshot, scanned form) and needs the text
- You want a **guided** extraction (e.g. “return only the invoice total and date as JSON”) — pass `prompt`
- You need an in-brain path because declared HTTP tools cannot call `POST /transcription` (BRA410)

Do **not** use this tool for pages of HTML — use the native `web_fetch` tool. Do **not** pass file bytes; the tool takes a URL only.

---

## Wiring it into a workflow

1. Include the `tools/system-tools/` group in `brain-compose.yml`.
2. Either inject `transcribe_image` under workflow `tools:`, or list it under
   `available-tools:` and load this skill (`BRA412`) via `get_skill` to promote it.

---

## Safety and scope

- **Brain-scoped.** Vision keys and `transcriptionModel` come from the current brain.
- **HTTPS only** in production (HTTP is allowed only for development localhost / `host.docker.internal`).
- **SSRF-guarded.** The host is resolved and private / loopback / link-local / metadata ranges are refused. The failure message does not leak resolved IPs.
- **No persistence.** The downloaded bytes are discarded after transcription.

---

## See also

- **BRA410** — Execution API file transcription (`POST /transcription`)
- **BRA201** — `transcription-model` in `brain-compose.yml`; system tool YAML
- **BRA202** — `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` naming
- **BRA411** — skill discovery (`get_skill` can promote this tool)
