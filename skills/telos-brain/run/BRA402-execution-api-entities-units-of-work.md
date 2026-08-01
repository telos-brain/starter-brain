---
name: "Execution API: Entities & Units of Work"
code: BRA402
version: 6
description: How to create and manage entities and units of work via the
  Execution API — including appending context and data logs, and completing a
  unit of work to trigger the learning eval.
---

# Execution API: Entities & Units of Work

See BRA401 for authentication and conventions.

---

## Entities

An entity is a brain-scoped identifier for a real business record held by the harness application (e.g. a customer, a project). Each entity has an immutable entity type, referenced by its deploy **code**.

An entity can also hold **variables** — key/value pairs keyed by the variable
keys its type declares in the brain schema (see BRA201 §4.1). Variables are how
per-entity data (e.g. an external `organisationId`) is stored so a tool parameter
can inject it automatically at dispatch (BRA201 §5.3).

### `POST /entities` — create an entity

```json
{
  "entityTypeCode": "customer",
  "name": "Acme Ltd",
  "description": "Key account",
  "variables": [
    { "key": "organisationId", "value": "crm-12345" }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `entityTypeCode` | yes | Must match a type declared in the brain schema. `400` if unknown. |
| `name` | yes | |
| `description` | no | |
| `variables` | no | Array of `{ key, value }` pairs. Keys should match a variable key declared on the entity's type. |

Response `201 Created` — returns the full entity object including `id`, `entityTypeId`, `createdAt`, `updatedAt`, and `variables` (the stored key/value pairs).

### `GET /entities` — list entities

Optional query parameter `entity_type_code` narrows to a single type. An unknown code yields an empty list.

```http
GET /entities?entity_type_code=customer
```

Response `200 OK` — flat array of entity objects. No pagination.

### `PUT /entities/{id}` — update an entity

Updates `name` and `description` only; entity type is immutable.

```json
{
  "name": "Acme Limited",
  "description": "Renamed"
}
```

Response `200 OK` with the updated entity, or `404 Not Found`.

### `GET /entities/{id}/variables` — read an entity's variables

Response `200 OK` — array of `{ key, value }` pairs, or `404 Not Found`.

### `PUT /entities/{id}/variables` — set an entity's variables

Creates or updates the supplied keys; keys not listed are left untouched (this is
an upsert, not a wholesale replace). Use it to set variables after creation, or to
change them later.

```json
{
  "variables": [
    { "key": "organisationId", "value": "crm-99999" },
    { "key": "accountCode", "value": "AC-42" }
  ]
}
```

Response `200 OK` — the entity's full set of variables after the update, or `404 Not Found`.

---

## Units of Work

A unit of work is a discrete job performed against an entity. It is the context window for the agent and the trigger point for the learning loop.

**Lifecycle:** `Open → InProgress → Complete / Cancelled / Failed`

Two append-only logs hang off each unit of work:

- **Context** — narrative events (decisions, agent reasoning, user actions). Available to scoped workflow runs via template tags (`{{#unitOfWork.context}}`).
- **Data** — structured telemetry payloads (tool traces, data snapshots). Available via `{{#unitOfWork.data}}`. A fast, high-frequency write path.

Both logs are insert-only — there is no update or delete path.

A unit of work can also hold **variables** — key/value pairs keyed by the variable
keys its type declares in the brain schema (see BRA201 §4.1). These work exactly
like entity variables, but scope the value to a single piece of work (e.g. the
external `jobId` created for this job) so a tool parameter bound with
`unitofwork:` can inject it automatically at dispatch (BRA201 §5.3).

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

Initial status is `Open`. Response `201 Created` — returns the full unit-of-work object including `variables` (the stored key/value pairs).

### `GET /units-of-work/{id}/variables` — read a unit of work's variables

Response `200 OK` — array of `{ key, value }` pairs, or `404 Not Found`.

### `PUT /units-of-work/{id}/variables` — set a unit of work's variables

Creates or updates the supplied keys; keys not listed are left untouched (an
upsert, not a wholesale replace). Use it to set variables after creation, or to
change them later — for example once the harness has created the external record.

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
  "source": "agent"
}
```

| Field | Required | Notes |
|---|---|---|
| `date` | no | Event timestamp; defaults to now. |
| `title` | yes | |
| `message` | no | |
| `source` | yes | Free-form string (e.g. `agent`, `user`, `system`). |

Response `201 Created` with `{ "id": "<context-record-id>" }`, or `404 Not Found`.

### `POST /units-of-work/{id}/data` — append data

Optimised for high-frequency writes. Returns `201 Created` with **no body**.

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

No request body. Response `200 OK` with the unit-of-work object, or `404 Not Found`.