---
name: "Execution API: Authentication & Conventions"
code: BRA401
version: 3
description: How to authenticate with the Telos Brain Execution API, how tenancy
  is resolved, the brain identity endpoint, and the common conventions (status
  codes, error format, content types) that apply across all endpoints.
---

# Execution API: Authentication & Conventions

The Execution API is the runtime surface of a Telos Brain. It is entirely separate from the Management API (the deploy plane used by the CLI) — different middleware, different credentials, different routes.

| | Management API | Execution API |
|---|---|---|
| Purpose | Provision & configure brains | Run workflows against entities/units of work |
| Auth | Clerk org JWT / `tbk_` org API key | Per-brain API key |
| Tenant scope | Explicit `BrainId` per query | Implicit — resolved from the API key |
| Routes | `/brains/...` | `/brain`, `/entities`, `/units-of-work`, `/workflows`, `/runs`, `/inbox`, `/skills`, `/transcription` |

---

## Authentication

All Execution API endpoints are secured by the **brain API key**, issued once by the Management API when the brain is created. Present it as a bearer token:

```http
Authorization: Bearer <brain-api-key>
```

The key resolves to a single active brain, which becomes the implicit tenant scope for the entire request. An EF global query filter is activated — every entity, unit of work, workflow, run, and telemetry read/write is automatically scoped to that brain. **No `brain_id` is ever accepted in a request body or route.**

### Auth behaviour

| Situation | Response |
|---|---|
| Missing or malformed `Authorization` header | `401 Unauthorized` |
| Key does not resolve to an active brain | `401 Unauthorized` |
| Resource belongs to a different brain | `404 Not Found` (never `403`) |

Cross-tenant access always reads as not found — the existence of another brain's resources is never revealed.

---

## Brain identity

### `GET /brain`

Returns the public header metadata for the brain resolved from the API key. Harness callers use this to confirm which instance they are talking to. Brains have no schema/deploy version stamp — versioning is per-resource on the Management API.

Response `200 OK`:

```json
{
  "id": "3f0c...",
  "instanceName": "salesmate",
  "name": "Salesmate",
  "description": "Sales coaching brain",
  "status": "Active",
  "embeddingModel": "voyage-3-lite",
  "createdAt": "2026-07-09T07:00:00Z",
  "updatedAt": "2026-07-09T07:00:00Z"
}
```

| Field | Notes |
|---|---|
| `id` | Brain GUID |
| `instanceName` | Organisation-scoped DNS slug |
| `name` | Display name |
| `description` | Optional |
| `status` | `Active` when authenticated |
| `embeddingModel` | From deploy; may be null on older brains |
| `createdAt` / `updatedAt` | Timestamps |

Does **not** return the API key or organisation id.

---

## Conventions

- **Base URL** — all paths are relative to the brain service host.
- **Content type** — request and response bodies are JSON (`application/json`), except the sync run endpoint which streams `text/event-stream`, and `POST /transcription` which accepts `multipart/form-data` (see BRA410).
- **Tenancy** — implicit; derived from the API key.
- **Codes vs IDs** — entity types and unit-of-work types are referenced by their stable deploy **code**; instances are referenced by their **id** (GUID).
- **Errors** — failures return `{ "error": "message" }` with an appropriate status code.

### Common status codes

| Status | Meaning |
|---|---|
| `200 OK` | Successful read or idempotent action |
| `201 Created` | Resource created |
| `202 Accepted` | Async run queued |
| `400 Bad Request` | Missing/invalid field, or unknown type code |
| `401 Unauthorized` | Missing or invalid API key |
| `404 Not Found` | Resource not found within the brain's scope |