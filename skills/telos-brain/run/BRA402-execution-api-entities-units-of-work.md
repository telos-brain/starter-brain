---
name: "Execution API: Entities & Units of Work"
code: BRA402
version: 7
description: How to create, list, update, and delete entities, and how to
  create, list, read, and complete units of work via the Execution API —
  including paged lists, optional variable inclusion, sensitive-value masking,
  appending context and data logs, and completing a unit of work to trigger
  the learning eval.
---

# Execution API: Entities & Units of Work

See BRA401 for authentication and conventions.

---

## Entities

An entity is a brain-scoped identifier for a real business record held by the harness application (e.g. a customer, a project). Each entity has an immutable entity type, referenced by its deploy **code**.

An entity can also hold **variables** — key/value pairs keyed by the variable
keys its type declares in the brain schema (see BRA213). Variables are how
per-entity data (e.g. an external `organisationId`) is stored so a tool parameter
can inject it automatically at dispatch (BRA214).

**Status:** `active` on create. `DELETE` sets `deleted` and hides the entity
from every read. The list also accepts `archived`. There is no Execution API
call that sets `archived`.

**Sensitive variables.** Each variable has `isSensitive`. When it is true, read
responses include the key and `isSensitive: true`, and `value` is `null`. The
stored value is still available to tool dispatch. Omitting `isSensitive` on
create, or on a new key, stores the variable as sensitive. On update, omitting
`value` or `isSensitive` leaves that field unchanged.

### `POST /entities` — create an entity

```json
{
  "entityTypeCode": "customer",
  "name": "Acme Ltd",
  "description": "Key account",
  "variables": [
    { "key": "organisationId", "value": "crm-12345", "isSensitive": false }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `entityTypeCode` | yes | Must match a type declared in the brain schema. `400` if unknown. |
| `name` | yes | |
| `description` | no | |
| `variables` | no | Array of `{ key, value, isSensitive }`. Keys should match a variable key declared on the entity's type. `isSensitive` defaults to `true`. |

Response `201 Created` — the entity, including `id`, `entityTypeId`, `status`
(`active`), `createdAt`, `updatedAt`, `modifiedDate` (same instant as
`updatedAt`), and `variables`. Sensitive values are `null`.

### `GET /entities` — list entities

Paged. Page size is fixed at 100. `page` is 1-based; a value below 1 is treated as 1. Results are ordered by `updatedAt` descending, then `id`.

| Query | Default | Notes |
|---|---|---|
| `entity_type_code` | all types | Unknown code yields an empty page. |
| `status` | `active` | `active` or `archived` (case-insensitive). Anything else, including `deleted`, is `400`. |
| `page` | `1` | |
| `variables` | omitted | Comma-separated keys. When omitted, items have no `variables` field. When set, each item includes only those keys (an empty array when none match). Sensitive values are `null`. |

```http
GET /entities?entity_type_code=customer&status=active&page=1&variables=organisationId
```

Response `200 OK`:

```json
{
  "items": [
    {
      "id": "3f0c...",
      "entityTypeId": "…",
      "name": "Acme Ltd",
      "description": "Key account",
      "status": "active",
      "createdAt": "2026-07-09T07:06:00Z",
      "updatedAt": "2026-07-09T07:06:00Z",
      "modifiedDate": "2026-07-09T07:06:00Z",
      "variables": [
        { "key": "organisationId", "value": null, "isSensitive": true }
      ]
    }
  ],
  "totalCount": 1,
  "page": 1,
  "pageSize": 100
}
```

When the page is not empty, the response includes `X-Modified-Sync`: the latest
`updatedAt` on **that page**, UTC, with seven fractional digits
(`yyyy-MM-ddTHH:mm:ss.fffffffZ`).

### `PUT /entities/{id}` — update an entity

Updates `name` and `description` only. Entity type and variables are unchanged.

```json
{
  "name": "Acme Limited",
  "description": "Renamed"
}
```

Response `200 OK` with the updated entity (`variables` omitted), or `404 Not Found`.

### `DELETE /entities/{id}` — delete an entity

Soft-delete. Sets status to `deleted` and hides the row. Response `204 No Content`. A missing or already-deleted id is `404 Not Found`.

### `GET /entities/{id}/variables` — read an entity's variables

Response `200 OK` — array of `{ key, value, isSensitive }`. `value` is `null` when `isSensitive` is true. `404 Not Found` when the entity is missing.

### `PUT /entities/{id}/variables` — set an entity's variables

Creates or updates the supplied keys; keys not listed are left untouched (an
upsert, not a wholesale replace). A null `value` leaves the stored value
unchanged. A null `isSensitive` leaves the stored flag unchanged. A new key
with `isSensitive` omitted is stored as sensitive.

```json
{
  "variables": [
    { "key": "organisationId", "value": "crm-99999", "isSensitive": false },
    { "key": "accountCode", "value": "AC-42" }
  ]
}
```

Response `200 OK` — the entity's full set of variables after the update, with sensitive values masked, or `404 Not Found`.

---

## Units of Work

A unit of work is a discrete job performed against an entity. It is the context window for the agent and the trigger point for the learning loop.

**Lifecycle:** `Open → InProgress → Complete / Cancelled / Failed`

Two append-only logs hang off each unit of work:

- **Context** — narrative events (decisions, agent reasoning, user actions). Available to scoped workflow runs via template tags (`{{#unitOfWork.context}}`).
- **Data** — structured telemetry payloads (tool traces, data snapshots). Available via `{{#unitOfWork.data}}`. A fast, high-frequency write path.

The HTTP API is insert-only for these logs. In-brain system tools can update a data entry by reference (BRA407). Merged telemetry is `GET /units-of-work/{id}/telemetry` (BRA403).

A unit of work can also hold **variables** — key/value pairs keyed by the variable
keys its type declares in the brain schema (see BRA213). These work like entity
variables, but scope the value to a single piece of work (e.g. the external
`jobId` created for this job) so a tool parameter bound with `unitofwork:` can
inject it at dispatch (BRA214). Unit-of-work variables are not masked: the
variable endpoints and the detail response return the stored value, with
`isSensitive: false` on detail and list inclusion.

### `POST /units-of-work` — create a unit of work

```json
{
  "entityId": "3f0c...",
  "unitOfWorkTypeCode": "proposal",
  "title": "Q3 proposal for Acme",
  "variables": [
    { "key": "jobId", "value": "job-98765" }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `entityId` | yes | Must belong to the brain. `404` otherwise. |
| `unitOfWorkTypeCode` | yes | Must match a type in the brain schema. `400` if unknown. |
| `title` | yes | |
| `variables` | no | Array of `{ key, value }` pairs. Keys should match a variable key declared on the unit of work's type. |

Initial status is `Open`. Response `201 Created` — the unit of work, including `variables`.

### `GET /units-of-work` — list units of work for an entity

Paged. `page` is 1-based; a value below 1 is treated as 1. `pageSize` defaults to 100; a value below 1 uses 100. Results are ordered by `createdAt` descending, then `id`.

| Query | Default | Notes |
|---|---|---|
| `entity_id` | — | Required. An unknown id yields an empty page. |
| `status` | `Open` | Exact match: `Open`, `InProgress`, `Complete`, `Cancelled`, or `Failed`. Anything else is `400`. |
| `page` | `1` | |
| `pageSize` | `100` | |
| `variables` | omitted | Comma-separated keys. When omitted, items have no `variables` field. When set, each item includes only those keys. |

```http
GET /units-of-work?entity_id=3f0c...&status=Open&page=1&pageSize=100&variables=jobId
```

Response `200 OK`:

```json
{
  "items": [
    {
      "id": "…",
      "reference": "ab12cd34",
      "title": "Q3 proposal for Acme",
      "status": "Open",
      "entityId": "3f0c...",
      "createdAt": "2026-07-09T07:06:00Z",
      "updatedAt": "2026-07-09T07:06:00Z",
      "completedDate": null,
      "variables": [
        { "key": "jobId", "value": "job-98765", "isSensitive": false }
      ]
    }
  ],
  "totalCount": 1,
  "page": 1,
  "pageSize": 100
}
```

### `GET /units-of-work/{id}` — read one unit of work

`format` is `json` (default) or `markdown`. Anything else is `400`. A unit of work in another brain is `404`.

JSON (`200 OK`, `application/json`) includes the summary fields plus:

| Field | Notes |
|---|---|
| `variables` | `{ key, value, isSensitive }`. Values are present; `isSensitive` is `false`. |
| `context` | `{ date, title, message, source }`, in stored order. |
| `data` | `{ date, source, type, body, tags, effort }`, in stored order. |

Markdown (`200 OK`, `text/markdown`) is a document: a title line with status and dates, then `## Context`, `## Data`, and `## Variables`. Each context or data entry is a `###` heading. Each variable is `` `key`: value ``.

```http
GET /units-of-work/{id}?format=markdown
```

### `GET /units-of-work/{id}/variables` — read a unit of work's variables

Response `200 OK` — array of `{ key, value }`, or `404 Not Found`.

### `PUT /units-of-work/{id}/variables` — set a unit of work's variables

Creates or updates the supplied keys; keys not listed are left untouched (an
upsert, not a wholesale replace).

```json
{
  "variables": [
    { "key": "jobId", "value": "job-98765" },
    { "key": "batchCode", "value": "B-2026-07" }
  ]
}
```

Response `200 OK` — the unit of work's full set of variables after the update, or `404 Not Found`.

### `POST /units-of-work/{id}/context` — append context

```json
{
  "date": "2026-07-09T07:06:00Z",
  "title": "Draft created",
  "message": "Initial draft generated from template",
  "source": "agent",
  "tags": "draft"
}
```

| Field | Required | Notes |
|---|---|---|
| `date` | no | Event timestamp; defaults to now. |
| `title` | yes | |
| `message` | no | |
| `source` | yes | Free-form string (e.g. `agent`, `user`, `system`). |
| `tags` | no | Optional comma-separated tags. |

Response `201 Created` with `{ "id": "<context-record-id>", "reference": "<8-char-reference>" }`, or `404 Not Found`.

### `POST /units-of-work/{id}/data` — append data

Optimised for high-frequency writes. Returns `201 Created` with **no body**. The server still assigns a reference, available later via detail, telemetry, and system tools.

```json
{
  "date": "2026-07-09T07:06:30Z",
  "source": "tool:crm_lookup",
  "type": "tool_response",
  "body": "{ ... }",
  "tags": "trace,retry",
  "effort": 300
}
```

| Field | Required | Notes |
|---|---|---|
| `date` | no | Defaults to now. |
| `source` | yes | |
| `type` | yes | |
| `body` | yes | Unbounded string; typically JSON. |
| `tags` | no | Optional comma-separated tags. |
| `effort` | no | Optional effort in seconds (integer). Stored as-is; omit for `null`. |

Response `201 Created` (no body), or `404 Not Found`.

### `POST /units-of-work/{id}/complete` — mark complete

Transitions the unit of work to `Complete`, sets `completedDate`, and **atomically** enqueues the learning-eval workflow as a background job. If the enqueue fails, the status change is rolled back — the two never partially commit.

**Idempotent** — completing an already-complete unit of work returns `200 OK` with no side effects (no duplicate eval).

No request body. Response `200 OK` with the unit-of-work object (`variables` omitted), or `404 Not Found`.
