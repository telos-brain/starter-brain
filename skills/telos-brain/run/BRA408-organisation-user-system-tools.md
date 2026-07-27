---
name: Organisation User System Tools
code: BRA408
version: 1
description: The in-brain system tools for discovering Organisation members and
  assigning InboxTasks — list_users and assign_task_to_user. Scoped to the
  Brain's Organisation via harness-injected BrainId. list_users returns CSV of
  name, email, and free-text role profile; assign_task_to_user sets
  InboxTasks.AssignedTo by member email. For creating a task already assigned,
  see add_inbox_task assigned_to in BRA405.
tools:
  - list_users
  - assign_task_to_user
---

# Organisation User System Tools

Agents often need to know **who** is in the Organisation and hand an InboxTask
to a named person. These two system tools cover that without outbound HTTP
(Clerk is never called from a system tool — name/email come from local Member
rows; free-text role profiles come from org Membership).

These are ordinary `system` tools (BRA201 §5.2). Declarations live under
`tools/organisation/` in the brain schema. This skill lists them in frontmatter
`tools:` so a workflow that keeps them under `available-tools` can promote them
via `get_skill` (same pattern as BRA405 / BRA406).

**Scope:** always the Organisation that owns the current Brain
(`BrainId → Brains.OrganisationId`). Organisation id is never a tool parameter.

---

## The two tools

| Tool | Purpose | Key parameters | Returns |
|---|---|---|---|
| **`list_users`** | List active Organisation members | *(none)* | CSV `name,email,role` |
| **`assign_task_to_user`** | Assign an InboxTask by email | `task_reference`, `email` | Confirmation with task reference |

Intended flow:

1. Call `list_users` to discover members and their role profiles.
2. Pick an email (or use one the operator already named).
3. Call `assign_task_to_user` with the task's 8-character **reference** and that
   email — or pass `assigned_to` when creating via `add_inbox_task` (BRA405).

---

## `list_users`

| | |
| --- | --- |
| **Purpose** | Discover who can be assigned InboxTasks in this Brain's Organisation |
| **Parameters** | None — Organisation is resolved from harness `BrainId` |
| **YAML** | `tools/organisation/list-users.yml` |

### Returns

CSV with header row:

```
name,email,role
Alice Smith,alice@example.com,Handles triage and expert review
Bob Jones,bob@example.com,
```

| Column | Source |
| --- | --- |
| `name` | `Members.DisplayName` (falls back to email when blank) |
| `email` | `Members.EmailAddress` |
| `role` | `Memberships.RoleDescription` — free-text **org-scoped** profile for agents; empty string when unset. **Not** the access-control role (`Admin` / `Member`) |

Only **active** memberships (`IsDeleted = 0`) are returned.

### When to use

Before assigning work: load this skill, call `list_users`, match a person by
role description or email, then assign.

---

## `assign_task_to_user`

| | |
| --- | --- |
| **Purpose** | Set `InboxTasks.AssignedTo` to an active member's email |
| **YAML** | `tools/organisation/assign-task-to-user.yml` |

### Parameters

| Parameter | Required | Notes |
| --- | --- | --- |
| `task_reference` | Yes | 8-character lowercase alphanumeric InboxTask reference (`[a-z0-9]{8}`). Never a UUID. |
| `email` | Yes | Email of an **active** Organisation member. Normalised to lowercase on store. |

### Behaviour

1. Resolve the task by `task_reference` within the current Brain (not found →
   error).
2. Resolve the Brain's Organisation.
3. Validate `email` is a well-formed address of an active member of that
   Organisation. Unknown, inactive, or out-of-org emails return a clear error
   and **write nothing**.
4. Write the normalised email to `InboxTasks.AssignedTo`.

Assignment is **storage-only** — no email, push, or in-app notification.

### Returns

Plain confirmation, e.g. `Inbox task abcd1234 assigned to alice@example.com.`

### Errors (plain English)

- Invalid / missing `task_reference`
- Task not found in this Brain
- Missing `email`
- Email not a valid address, or not an active Organisation member

---

## Creating a task already assigned

Prefer `add_inbox_task` with optional `assigned_to` (same email validation) when
you are creating the task in the same step — see **BRA405**. Use
`assign_task_to_user` when the task already exists.

---

## Wiring them into a workflow

1. Include the `tools/organisation/` group in `brain-compose.yml`.
2. Either inject under workflow `tools:`, or list under `available-tools:` and
   load this skill (`BRA408`) via `get_skill` to promote them.

---

## Safety and scope

- **Organisation-scoped via Brain.** Never pass Organisation id or Member id.
- **Active members only.** Soft-deleted memberships are excluded.
- **References for tasks.** Agents must copy `task_reference` from list/get /
  create output — never invent Guids.
- **Role profile is AI-readable.** `Memberships.RoleDescription` is injected
  into prompts via `list_users`; operators edit it in Admin → Users with a
  helper note that the text is included verbatim in agent prompts.

---

## See also

- **BRA405** — inbox system tools (`add_inbox_task` optional `assigned_to`,
  `list_inbox_tasks`, …)
- **BRA404** — Execution API inbox and trigger stages
- **BRA204** — `{{inboxEntry.*}}` / `{{#inboxTasks}}` template tags
