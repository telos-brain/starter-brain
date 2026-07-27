---
name: Inbox System Tools
code: BRA405
version: 6
description: The in-brain system tools for operating the learning-signal inbox
  — create_inbox_entry, list_inbox_entries, get_inbox_entry, update_inbox_entry,
  list_inbox_tasks, add_inbox_task and update_inbox_task. All identity is by
  8-character reference (never UUID); workflows are selected by code. Covers how
  PENDING tasks auto-run from the linked workflow's inbox trigger (see BRA404).
  For persisting a run quality score, see BRA406 (set_run_grading). For listing
  Organisation members and assigning tasks by email, see BRA408 (list_users,
  assign_task_to_user).
tools:
  - create_inbox_entry
  - list_inbox_entries
  - get_inbox_entry
  - update_inbox_entry
  - list_inbox_tasks
  - add_inbox_task
  - update_inbox_task
---

# Inbox System Tools

BRA404 covers the **Execution API** HTTP surface for inbox intake (paths still
use Guids). This skill covers the complementary **AI-facing system tools**: a
running brain creating, listing, reading and updating its own inbox entries and
tasks **in-process** — no webhook URL or API key.

**Identity rule (BRA084):** every tool parameter and every list/detail field that
identifies an entry or task uses an **8-character lowercase alphanumeric
reference** (`[a-z0-9]{8}`). Do **not** pass UUIDs. Linked workflows are named by
**`workflow_code`**, not `workflow_id`.

These are ordinary `system` tools (BRA201 §5.2). Declarations live under
`tools/inbox/` in the brain schema (and sample brains). This skill lists them in
frontmatter `tools:` so a workflow that keeps them under `available-tools` can
promote them via `get_skill` (same pattern as BRA203 / `WF-SKILL-UPDATE`).

**Run grading** (`set_run_grading`) is declared in the same `tools/inbox/` group
but documented in **BRA406** — eval workflows typically call both
`create_inbox_entry` and `set_run_grading` (BRA207).

---

## The seven inbox tools

| Tool | Purpose | Key parameters | Returns |
|---|---|---|---|
| **`create_inbox_entry`** | Create a learning signal | `title`, `body`, `routing_type`, optional `source`, `status` | Confirmation with new entry **reference** |
| **`list_inbox_entries`** | List entries (optional filters) | `status`, `routing_type` | CSV keyed by `Reference` |
| **`get_inbox_entry`** | Full entry + tasks | `inbox_entry_reference` | Markdown |
| **`update_inbox_entry`** | Status / routing type | `inbox_entry_reference`, `status`, `routing_type` | Confirmation |
| **`list_inbox_tasks`** | Tasks for one entry | `inbox_entry_reference` | CSV keyed by `Reference` |
| **`add_inbox_task`** | Create a task | `inbox_entry_reference`, `workflow_code`, `instructions`, optional `assigned_to` | Confirmation with new task reference |
| **`update_inbox_task`** | Status / action; approve | `inbox_task_reference`, `status`, `action` | Confirmation |

Intended flows:

- **Learning evals:** `create_inbox_entry` (once per learning) then
  `set_run_grading` (BRA406 / BRA207)
- **Triage / review:** list entries → get entry → list/add/update tasks

---

## Parameter reference (no UUIDs)

| Tool | Parameter | Notes |
|---|---|---|
| `create_inbox_entry` | `title`, `body`, `routing_type` | Required |
| `create_inbox_entry` | `source` | Optional producing-system label |
| `get_inbox_entry` | `inbox_entry_reference` | Required |
| `update_inbox_entry` | `inbox_entry_reference` | Required; plus `status` and/or `routing_type` |
| `list_inbox_tasks` | `inbox_entry_reference` | Required |
| `add_inbox_task` | `inbox_entry_reference` | Required |
| `add_inbox_task` | `workflow_code` | Optional — workflow **code**, not id |
| `add_inbox_task` | `instructions` | Optional — instructions-only task action |
| `add_inbox_task` | `assigned_to` | Optional — email of an active Organisation member; omit/null leaves unassigned (BRA408) |
| `update_inbox_task` | `inbox_task_reference` | Required |
| `update_inbox_task` | `status` / `action` | At least one required |

Deprecated / invalid for these tools: `id`, `inbox_entry_id`, `workflow_id`,
`task_id`.

---

## `create_inbox_entry`

Creates an inbox entry in-process via `IInboxService` (same path as
`POST /inbox`). Prefer this over a declared HTTP tool that `POST`s to a host
URL — no outbound network call, no API key, no localhost.

Required: `title`, `body`, `routing_type` (`SKILL_UPDATE`, `WORKFLOW_UPDATE`,
`TOOL_UPDATE`, `MEMORY_UPDATE`, `SYSTEM_CHANGE`, or a brain-defined value).
Optional: `source`, `status`.

| `status` | Behaviour |
| --- | --- |
| omitted / `PENDING` | Stage-1 inbox trigger matching runs (entry `routing_type` + learning mode — BRA404), then the entry auto-promotes to `PROCESSED` |
| `PROCESSED` | Created without firing inbox triggers — preferred for grade-linked findings that should not spawn tasks |

Returns a confirmation including the new entry's **reference**, **Depth**, and
how many triggered tasks were created. Those tasks start `PENDING`; auto-run vs
`AWAITING_APPROVAL` is decided later from each task's linked workflow (BRA404
stage 2).

**Depth** (mirrors `WorkflowRun.RecursionDepth`):

| How the entry is created | Depth |
| --- | --- |
| Management / Execution API, email, Granola, etc. | `1` |
| `create_inbox_entry` from a run **not** linked to an inbox entry | `1` |
| `create_inbox_entry` from a run whose `InboxEntryId` is set (workflow triggered from an inbox entry/task) | parent entry `Depth + 1` |

So a top-level inbox signal is depth 1; a learning created by a workflow that
ran from that entry is depth 2; and so on.

Canonical YAML: `tools/inbox/create-inbox-entry.yml`.

---

## `list_inbox_entries`

Optional `status` (`PENDING`, `PROCESSED`, `COMPLETED`) and `routing_type`.
Returns CSV:

`Reference,Date,Source,Title,Status,RoutingType,CreatedAt`

Use `get_inbox_entry` for the body.

## `get_inbox_entry`

Requires `inbox_entry_reference`. Returns markdown with entry scalars, full body,
and a task list (each task by **reference**, with `workflow` as a **code**).

## `update_inbox_entry`

Requires `inbox_entry_reference` and at least one of `status` or `routing_type`.
Entry lifecycle is `PENDING → PROCESSED → COMPLETED` (BRA113). Terminal
`COMPLETED` cannot be overwritten.

## `list_inbox_tasks`

Requires `inbox_entry_reference`. Returns CSV for that entry only:

`Reference,Status,WorkflowCode,Action,Response,CreatedAt,UpdatedAt`

## `add_inbox_task`

Requires `inbox_entry_reference`. Optional `workflow_code` (resolved server-side),
`instructions` (stored as the task action — instructions-only; entry content
reaches triggered workflows via `{{inboxEntry.*}}`, see BRA204), and
`assigned_to` (email of an active Organisation member — same validation as
`assign_task_to_user` in **BRA408**; omit or pass null to leave unassigned).
Creates status `PENDING` and returns the new task's **reference**.

**Auto-run:** the Hangfire processor decides from the **workflow named by
`workflow_code`**, not from the entry's routing type. If that workflow has an
`inbox:…` trigger whose learning-mode qualifier is satisfied, the task moves
`PENDING → RUNNING` and runs. Otherwise it moves to `AWAITING_APPROVAL` for
human sign-off. Full rules: **BRA404** (Inbox triggers — two stages). Workflow
`trigger-mode` does not control this path.

To assign an **existing** task, use `assign_task_to_user` (**BRA408**) instead
of recreating it.

## `update_inbox_task`

Requires `inbox_task_reference` and at least one of `status` or `action`.
Forward-only task lifecycle. Setting `status` to `RUNNING` from
`AWAITING_APPROVAL` **approves** the task and runs the linked workflow
synchronously (settles `COMPLETED` or `FAILED`; never left stranded in
`RUNNING`).

---

## Wiring them into a workflow

1. Include the `tools/inbox/` group in `brain-compose.yml`.
2. Either inject under workflow `tools:`, or list under `available-tools:` and
   load this skill (`BRA405`) via `get_skill` to promote them.

Minimal declaration for create:

```yaml
name: create_inbox_entry
version: 1
description: >-
  Creates a single inbox entry recording one learning signal.
system:
  tool: create_inbox_entry
parameters:
  - name: title
    param: title
    type: string
    required: true
  - name: body
    param: body
    type: string
    required: true
  - name: routing_type
    param: routing_type
    type: string
    required: true
```

---

## Safety and scope

- **Brain-scoped, always.** The harness injects the brain; it is never a
  parameter. Cross-brain references read as not found.
- **References only.** Agents must copy `Reference` values from list/get /
  create output — never invent Guids.
- **Creating entries** uses the `create_inbox_entry` **system** tool (this
  skill). The Execution API `POST /inbox` (BRA404) remains available for
  external harnesses; eval workflows should prefer the system tool.

---

## See also

- **BRA404** — Execution API inbox (HTTP; Guid paths) and **inbox trigger stages**
  (entry create vs task auto-run, learning-mode qualifiers)
- **BRA201** §8 — authoring `trigger: inbox:…` / learning-mode on workflows
- **BRA207** — learning-eval workflows that call `create_inbox_entry`
- **BRA406** — `set_run_grading` (persist 0–100 score on a WorkflowRun)
- **BRA204** — `{{inboxEntry.*}}` / `{{task.*}}` / `{{#inboxTasks}}` template tags
- **BRA408** — `list_users` / `assign_task_to_user` (Organisation members and
  task assignment)
- **WF-INBOX-ENTRY-CONTEXT** — canonical TRIGGERED inbox workflow pattern
