---
name: "Execution API: Inbox Entries & Tasks"
code: BRA404
version: 6
description: How to create, list, read and update inbox entries and their tasks
  via the Execution API — the learning-signal intake surface. Covers the entry and
  task lifecycles, inbox trigger matching (entry create vs task auto-run), learning
  mode qualifiers, listing by status, adding entries, and updating an entry's or
  task's status and content (including approving a task to run its linked
  workflow). In-brain evals should prefer the create_inbox_entry system tool
  (BRA405) over HTTP.
---

# Execution API: Inbox Entries & Tasks

See BRA401 for authentication and conventions. See BRA403 for workflow execution
and telemetry. For the **AI-facing inbox system tools** (reference-keyed, never
UUID), see BRA405.

The **inbox** is a brain's learning-signal queue: the intake point for anything
the brain might learn from or act on — eval results, meeting transcripts, agent
traces, or external events. A harness application posts signals here and can then
list, read and manage them remotely, in addition to the brain UI.

Learning-eval workflows (BRA207) should prefer the in-process
`create_inbox_entry` **system tool** (BRA405). External harnesses can still
`POST` here once per extracted learning. Entries start as `PENDING` for human
review before any skill, workflow, or system change is applied.

The inbox has two record types:

- **Inbox entry** — a single learning signal (title, body, source, routing type).
- **Inbox task** — a routing/execution unit created from an entry during triage.
  A task may reference a workflow to run, and carries an `action` that becomes
  the workflow's **instructions-only** input when it runs.

### How triggered workflows see the entry

When a `TRIGGERED` workflow runs from an inbox task, entry content is **not**
injected into the input message. Authors must pull it in with template tags
(see BRA204):

| Scope | Use |
| --- | --- |
| `{{inboxEntry.*}}` | Entry scalars: `reference`, `date`, `source`, `title`, `body`, `status`, `routingType` |
| `{{task.*}}` | Triggering task scalars: `reference`, `action`, `response`, `status`, `workflowCode`, `expertOpinion` |
| `{{#inboxTasks}}...{{/inboxTasks}}` | Sibling tasks: `reference`, `action`, `response`, `status`, `workflowCode`, `expertOpinion` |

The task `action` / run input is triage instructions only (routing intent). Auto-
created trigger tasks start with a null `action`. Prefer `{{task.*}}` for the
current task; use `{{#inboxTasks}}` when sibling context is needed. Canonical
example workflow: `WF-INBOX-ENTRY-CONTEXT`.

---

## Inbox triggers (two stages)

Triggers live on **workflows** (`trigger:` in frontmatter — see BRA201 §8), not
on the task row. Inbox work uses them in two distinct stages:

| Stage | When | What is matched | Outcome |
| --- | --- | --- | --- |
| **Entry create** | `POST /inbox` / `create_inbox_entry` with status `PENDING` | Each `TRIGGERED` workflow's `inbox:…` pattern against the **entry's `routingType`** and the brain's **`learning-mode`** | Matching workflows each get a new `InboxTask` (`PENDING`, linked to that workflow) |
| **Task auto-run** | Hangfire processor picks up a `PENDING` task that has a linked workflow | That **task's linked workflow** only — does it have an inbox trigger whose learning-mode qualifier is satisfied? | Yes → `PENDING → RUNNING` and the workflow runs. No → `PENDING → AWAITING_APPROVAL` |

### Stage 1 — which tasks get created

Pattern shape: `inbox:<RoutingType>` or `inbox:*`, optionally with a learning-mode
qualifier (BRA122):

```text
inbox:SKILL_UPDATE
inbox:*
inbox:SKILL_UPDATE:low
inbox:WORKFLOW_UPDATE:medium
```

- `inbox:*` matches any routing type (including null).
- `inbox:<RoutingType>` requires an exact match on the entry's `routingType`.
- Multiple patterns on one workflow are OR'd (BRA121): a scalar, comma-separated
  list, or YAML list in frontmatter.
- Qualifier hierarchy: `off < low < medium < high`. A qualified trigger fires
  when the brain's `learning-mode` **meets or exceeds** the qualifier.
  Unqualified inbox triggers always pass the learning-mode check (including when
  the brain mode is omitted / off).
- Creating an entry as `PROCESSED` skips this stage entirely (no tasks).

### Stage 2 — whether a PENDING task auto-runs

Once a task exists, **the workflow already linked on that task is authoritative**
(via `WorkflowId` / `workflow_code`). The processor does **not** re-match the
entry's routing type.

- Auto-run when the linked workflow has **at least one** `inbox:…` trigger whose
  learning-mode qualifier is satisfied (routing segment ignored at this stage).
- Otherwise the task moves to `AWAITING_APPROVAL` for human sign-off (admin UI,
  `PATCH` approve, or `update_inbox_task` — see below / BRA405).
- A task with **no** linked workflow cannot auto-run; it parks at
  `AWAITING_APPROVAL`.
- Workflow `trigger-mode` (`manual` / `automatic`) does **not** control inbox
  task approval. That field is for eval-style triggers such as
  `workflowrun:complete` (BRA207).

**Authoring tip:** a triage flow that calls `add_inbox_task` with
`workflow_code: WF-SKILL-UPDATE` only auto-runs if `WF-SKILL-UPDATE` itself declares
an inbox trigger (and learning mode allows). Omit the inbox trigger (or use a
qualifier above the brain's mode) to keep a human in the loop.

---

## Lifecycles

Both records progress **forward-only**; a backward move, or overwriting a
terminal state, is rejected with `409 Conflict`. A "no change" (target equals
current) is allowed so a sibling field can be edited without advancing status.

**Inbox entry** — `PENDING → REVIEWING → APPLIED | DISMISSED`
(`APPLIED` and `DISMISSED` are terminal; `DISMISSED` is reachable from `PENDING`
or `REVIEWING`, `APPLIED` only from `REVIEWING`).

**Inbox task** — all new tasks start `PENDING`. Then either:

- `PENDING → RUNNING → COMPLETED | FAILED` when the linked workflow has a
  qualifying inbox trigger (auto-run), or
- `PENDING → AWAITING_APPROVAL → RUNNING → COMPLETED | FAILED` when it does not
  (human approval required),

or `→ CANCELLED` from any non-terminal state.

---

## Entries

### `POST /inbox` — add an entry

Creating an entry **also runs stage-1 inbox trigger matching in the same atomic
write**: every `TRIGGERED` workflow whose `inbox:…` pattern matches the entry's
`routingType` (and learning-mode qualifier) gets a `PENDING` `InboxTask` linked
to that workflow. The response therefore includes any tasks generated for the
entry. Whether those tasks auto-run is decided later (stage 2) from each task's
linked workflow — see [Inbox triggers](#inbox-triggers-two-stages).

```json
{
  "source": "eval-run-123",
  "title": "Refund flow mishandled",
  "body": "Full signal content in **markdown**.",
  "routingType": "SKILL_UPDATE"
}
```

| Field | Required | Notes |
|---|---|---|
| `title` | yes | Short label. |
| `body` | yes | Full signal content (markdown). |
| `source` | no | Free-text producing-system identifier (eval run id, trace id…). |
| `routingType` | no | Free-text classifier. Null matches only the `inbox:*` wildcard trigger. |

Response `201 Created` — the full entry, including its `status` (initially
`PENDING`) and any generated `tasks`.

### `GET /inbox` — list entries (by status)

```http
GET /inbox?status=PENDING&created_since=2026-07-01T00:00:00Z
```

| Query param | Notes |
|---|---|
| `status` | Optional. Filters to one entry status. |
| `created_since` | Optional ISO 8601 lower bound (`CreatedAt >= created_since`), for polling since a last poll. |

Response `200 OK` — a summary array (most recent source event first). The `body`
is **excluded** from the list to keep polling payloads lean.

### `GET /inbox/{id}` — entry detail

Response `200 OK` — the full entry including `body` and all its `tasks`, or
`404 Not Found` when the entry does not belong to the brain.

### `PATCH /inbox/{id}` — update an entry (status and/or content)

Partial update. Supply **at least one** field.

```json
{
  "status": "REVIEWING",
  "body": "Corrected signal content.",
  "routingType": "TOOL_UPDATE"
}
```

| Field | Notes |
|---|---|
| `status` | Optional. Must be a forward transition (see lifecycle). |
| `body` | Optional. Replacement markdown body. |
| `routingType` | Optional. Override the routing classification. |

Identity/provenance fields (`title`, `source`, `date`) are **immutable**.

Response `200 OK` with the updated entry (including its tasks). `404 Not Found`
for a cross-brain entry; `409 Conflict` for an unknown status value or an invalid
status transition; `400 Bad Request` if no fields are supplied.

---

## Tasks

### `GET /inbox/tasks` — list tasks (by status)

Lists the brain's inbox tasks across **all** entries — distinct from the tasks
embedded in a single entry's detail.

```http
GET /inbox/tasks?status=AWAITING_APPROVAL&entry_id=3f0c...
```

| Query param | Notes |
|---|---|
| `status` | Optional. Filters to one task status. |
| `entry_id` | Optional. Scopes to a single parent entry's tasks. |

Response `200 OK` — a task array (most recent first), each flattened with its
linked workflow's title and trigger mode, and its `workflowRunId` and `response`
once it has run.

### `PATCH /inbox/{entryId}/tasks/{taskId}` — update a task (status and/or content)

Partial update. Supply **at least one** field. The `entryId` scopes the task to
its parent entry.

```json
{
  "status": "RUNNING",
  "action": "Revised instructions for the workflow."
}
```

| Field | Notes |
|---|---|
| `status` | Optional. Must be a forward transition (see lifecycle). |
| `action` | Optional. Replacement **instructions-only** text passed to the workflow as its input message when the task runs. Does not carry the entry body — use `{{inboxEntry.body}}` in workflow Instructions. |

**Approving a task:** setting `status` to `RUNNING` from `AWAITING_APPROVAL`
approves it — the linked workflow runs synchronously with `action` as its input
message (instructions-only) and the entry available via `{{inboxEntry.*}}`
template tags. The task settles to `COMPLETED` (or `FAILED` on error), never
stranded in `RUNNING`. Its `workflowRunId` and final `response` (assistant reply)
are recorded. A task with **no** linked
workflow cannot be approved (`409`). **Cancelling** is `status: CANCELLED` from
any non-terminal state.

Response `200 OK` with the updated task. `404 Not Found` when the entry or task
does not belong to the brain (or the task is not under `entryId`); `409 Conflict`
for an unknown status value or an invalid transition; `400 Bad Request` if no
fields are supplied.

---

## Endpoint summary

| Method | Path | Purpose | Success |
|---|---|---|---|
| `POST` | `/inbox` | Add an entry (runs trigger matching) | `201` |
| `GET` | `/inbox` | List entries by status / created-since | `200` |
| `GET` | `/inbox/{id}` | Entry detail (body + tasks) | `200` |
| `PATCH` | `/inbox/{id}` | Update entry status / content | `200` |
| `GET` | `/inbox/tasks` | List tasks by status / entry | `200` |
| `PATCH` | `/inbox/{entryId}/tasks/{taskId}` | Update task status / content (approve, cancel, edit) | `200` |

> Tenancy is implicit (BRA401): every entry and task is scoped to the brain the
> API key resolves to. A record belonging to another brain always reads as
> `404 Not Found`, never `403`. No `brain_id` is ever accepted in a route or body.

---

## See also

- **BRA201** §8 — workflow `trigger` / `learning-mode` authoring
- **BRA405** — inbox system tools (`create_inbox_entry`, `add_inbox_task`, …)
- **BRA207** — learning-eval workflows (`trigger-mode` for `workflowrun:complete`)
- **BRA204** — `{{inboxEntry.*}}` / `{{task.*}}` / `{{#inboxTasks}}` template tags
