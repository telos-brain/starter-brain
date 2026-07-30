---
name: Template Tag Taxonomy
code: BRA204
description: Canonical reference for double-curly-bracket template tags used in workflow Instructions and tool response-markdown / error-markdown. Covers the input scope from Execution API variables, workflow-tool / run_workflow parameters, and input-tools mappings.
version: 12
---

# Template Tag Taxonomy

This skill is the **single source of truth** for the template tag syntax used by
the template service. Use it when authoring or editing:

- Workflow `Instructions` markdown
- Tool YAML `response-markdown` and `error-markdown` fields

No other reference is required. Tags that cannot be resolved render as blank —
never an error — so typos produce empty output silently.

---

## 1. Why double-curly-bracket syntax

Template tags use `{{...}}` (not angle brackets or single braces) so they remain
**YAML-safe** when embedded in tool definitions and markdown. Angle-bracket or
single-curly alternatives conflict with YAML structure and Markdown parsing.

---

## 2. Tag types

### 2.1 Scalar substitution

```text
{{scope.field}}
{{scope.nested.field}}
```

Replaces the tag with the resolved string value, or blank if unresolvable.

**Example — entity name:**

```markdown
Working on **{{entity.name}}**.
```

**Example — entity variable key:**

```markdown
Organisation id: {{entity.organisationId}}
```

### 2.2 Iteration

```text
{{#collection}}
...body...
{{/collection}}
```

Repeats the body once per item. Nesting is supported to arbitrary depth.
Unresolvable or empty collections render nothing.

**Inside hierarchical loops**, prefer the **qualified** singular prefix
(`skillBook`, `category`, `skill`). **Inside unit-of-work record loops**, use
**short form** field names (`date`, `time`, `title`, `source`, `body`).

### 2.3 Conditionals

```text
{{#if condition}}
...body...
{{/if}}
```

Supported condition forms:

| Form | Meaning |
| --- | --- |
| `path` | Truthy when the value is non-null, non-blank, or a non-empty collection |
| `path == "value"` | String equality (ordinal) |
| `path != "value"` | String inequality (ordinal) |

String literals may use double or single quotes. There is no full expression
language (no `&&`, `||`, comparisons other than `==` / `!=`).

**Example:**

```markdown
{{#if result.status == "error"}}
The call failed: {{result.message}}
{{/if}}

{{#if entity.name}}
Entity in scope: {{entity.name}}
{{/if}}
```

---

## 3. Scopes

Only scopes referenced by tags in the content are loaded (lazy load). Adding a
new scope is a provider + DI registration — the parser itself does not change.

### 3.1 `entity`

| | |
| --- | --- |
| **Type** | Scalar + open key/value bag |
| **Sort** | N/A |
| **Requires** | `EntityId` on the run; otherwise the scope is empty |

| Field | Description |
| --- | --- |
| `entity.name` | Entity display name |
| `entity.{variableKey}` | Value of a runtime entity variable (any key set on the entity) |

**Example:**

```markdown
# Context
Customer: {{entity.name}}
Region: {{entity.region}}
```

---

### 3.2 `unitOfWork`

| | |
| --- | --- |
| **Type** | Two enumerable collections |
| **Sort** | Date ascending (then creation order) for both collections |
| **Requires** | `UnitOfWorkId` on the run; otherwise the scope is empty |

| Collection | Item fields (short form inside the loop) |
| --- | --- |
| `unitOfWork.context` | `date`, `time`, `title`, `source`, `body` |
| `unitOfWork.data` | `date`, `time`, `title`, `source`, `body` |

Notes:

- `date` is `yyyy-MM-dd`; `time` is `HH:mm` (from the record's `Date` timestamp).
- Context `body` is the narrative message; data `title` is the record type.
- There is **no** `unitOfWork.combined` scope yet (planned). Interleave by using
  both blocks when you need narrative and structured data together.

**Example — context log:**

```markdown
## Unit of work context
{{#unitOfWork.context}}
### {{date}} {{time}} — {{title}} ({{source}})
{{body}}

{{/unitOfWork.context}}
```

**Example — structured data log:**

```markdown
## Unit of work data
{{#unitOfWork.data}}
### {{date}} {{time}} — {{title}} ({{source}})
{{body}}

{{/unitOfWork.data}}
```

**Example — mixed (iteration + conditional):**

```markdown
{{#unitOfWork.context}}
- {{title}}{{#if body}}: {{body}}{{/if}}
{{/unitOfWork.context}}
```

---

### 3.2a `run` (BRA091)

| | |
| --- | --- |
| **Type** | Scalar bag |
| **Sort** | N/A |
| **Requires** | `WorkflowRunId` (subject run) on the template render context; otherwise the scope is empty |

| Field | Description |
| --- | --- |
| `run.reference` | Subject run's 8-character AI-facing reference (`WorkflowRuns.Reference`) for `set_run_grading` (BRA406) |
| `run.telemetry` | OTEL GenAI telemetry for the subject run as indented JSON (same shape as `GET /runs/{id}/telemetry`, compacted for eval prompts) |

Notes:

- Used by `workflowrun:complete` eval workflows. The subject run id is passed as
  `SubjectRunId` on the eval's `WorkflowRunRequest` — it is **not** the eval
  run's own id.
- When the subject run is missing, the `run` scope is empty (both tags blank).
- **`run.reference` is not a shortened Guid.** Telemetry's `runId` field is the
  first 8 hex characters of the UUID and must **not** be passed to
  `set_run_grading` — always use `{{run.reference}}`.
- Compacted for eval prompts: GUID values inside telemetry JSON are shortened to
  their first 8 hex characters, and `gen_ai.message.content` / `error.message`
  longer than 500 characters are truncated with an explicit marker
  (`...[truncated, full content length: N]`). The Execution API
  `GET /runs/{id}/telemetry` response stays full fidelity.
- Authoring guide for manual / automatic run evals: **BRA207**. Grading tool:
  **BRA406**.

**Example — learning eval instructions:**

```markdown
## Subject run
Reference: {{run.reference}}

## Run telemetry
{{run.telemetry}}
```

---

### 3.3 `skillBooks`

| | |
| --- | --- |
| **Type** | Three-level hierarchy (list at root) |
| **Sort** | Alphabetical by title at every level (books, categories, skills) |

| Level | Iteration | Fields (qualified form) |
| --- | --- | --- |
| Books | `{{#skillBooks}}...{{/skillBooks}}` | `skillBook.code`, `skillBook.title`, `skillBook.description` |
| Categories | `{{#skillBook.categories}}...{{/skillBook.categories}}` | `category.code`, `category.title`, `category.description`, `category.range` |
| Skills | `{{#category.skills}}...{{/category.skills}}` | `skill.code`, `skill.title` |

Notes:

- `category.code` and `category.range` are projected from the category index
  (e.g. index `100` → code `100`, range `100-199`). There is no separate
  code/range column in storage.

**Example — nested hierarchy:**

```markdown
# Available skills
{{#skillBooks}}
## {{skillBook.code}} — {{skillBook.title}}
{{skillBook.description}}

{{#skillBook.categories}}
### {{category.code}} {{category.title}} ({{category.range}})
{{category.description}}

{{#category.skills}}
- `{{skill.code}}`: {{skill.title}}
{{/category.skills}}

{{/skillBook.categories}}
{{/skillBooks}}
```

---

### 3.4 `now`

| | |
| --- | --- |
| **Type** | Scalar bag (always available) |
| **Sort** | N/A |
| **Requires** | Nothing — resolved at render time |

| Field | Format | Description |
| --- | --- | --- |
| `now.utcDate` | `yyyy-MM-dd` | Current UTC date |
| `now.utcTime` | `HH:mm` | Current UTC time |
| `now.utcDayOfWeek` | English name | Current UTC day of week (e.g. `Saturday`) |
| `now.localDate` | `yyyy-MM-dd` | Current local date |
| `now.localTime` | `HH:mm` | Current local time |
| `now.localDayOfWeek` | English name | Current local day of week |

Notes:

- **Local** uses the brain environment variable `TIMEZONE` (IANA id, e.g.
  `Pacific/Auckland`). When unset or unrecognised, local falls back to UTC.
- Values are captured when Instructions (or tool response templates) are
  rendered — including each chat continuation turn.

**Example:**

```markdown
## Clock
UTC: {{now.utcDayOfWeek}} {{now.utcDate}} {{now.utcTime}}
Local: {{now.localDayOfWeek}} {{now.localDate}} {{now.localTime}}
```

---

### 3.5 `result`

| | |
| --- | --- |
| **Type** | Open-ended flat key/value bag |
| **Sort** | N/A |
| **Source** | Caller-supplied tool/API response (e.g. tool `response-markdown`) |

| Field | Description |
| --- | --- |
| `result.{anyKey}` | Top-level key from the JSON response body |

Notes:

- The result scope is **flat**. Nested JSON objects/arrays are stringified.
- If the tool body is not valid JSON, the scope is `{ result: "<raw content>" }`
  — use `{{result.result}}` to print the whole body.
- Field names are unknown at parse time; consult the tool's response payload.

**Example — tool response-markdown:**

```yaml
response-markdown: |
  Status: {{result.status}}
  Id: {{result.id}}
error-markdown: |
  Failed: {{result.result}}
```

---

### 3.6 `input`

| | |
| --- | --- |
| **Type** | Open-ended flat key/value bag |
| **Sort** | N/A |
| **Source** | Merged from (1) Execution API `variables`, (2) workflow-tool / `run_workflow` parameters. See **BRA409**. |

| Field | Description |
| --- | --- |
| `input.{key}` | Value for that key from the merged bag |

The `input` scope is a **single flat bag**. Values come from two sources that
are merged at render time (tool / `run_workflow` parameters overlay API
variables when the same key appears in both):

| Source | How it is supplied | Persisted on `WorkflowRun`? |
| --- | --- | --- |
| Execution API `variables` | Optional string→string map on `POST …/run/sync` and `…/run/async` (BRA403) | Yes (JSON column) |
| Workflow-tool / `run_workflow` parameters | Resolved args when this workflow is invoked as a child | No (request only; overlays variables) |

Notes:

- **API variables** — pass `{ "variables": { "widget_reference": "WID-001" } }`
  on the run body. Use in Instructions as `{{input.widget_reference}}`, and in
  workflow `input-tools` parameter mappings the same way (BRA201 §8.0a).
- **Workflow-tool params** — declare parameters on the **workflow tool** YAML
  (`parameters:` under the tool that has `workflow: code: …`). Each `name`
  becomes an `input` key. Exposed (model-supplied) params, hardcoded `value:`
  params, and `entity:`-bound params are included. `secret:` params are
  **never** forwarded into the child prompt.
- Field names are case-insensitive (`{{input.question}}` matches `question`).
- When neither source supplies data, the scope is empty (tags render blank).
- Nested `run_workflow` / workflow-tool runs inherit the parent's API
  variables automatically.
- For workflow-tool children, the same values are still rendered as markdown on
  the child input message (`## name\nvalue`) for backwards compatibility —
  prefer `{{input.*}}` in new Instructions. See **BRA201** §5.2.

**Example — Execution API variables:**

```http
POST /workflows/WF-INPUT-VARIABLES/run/sync
Content-Type: application/json

{
  "variables": {
    "topic": "Telos Brain",
    "seed_question": "What is Telos Brain in one sentence?"
  },
  "inputMessage": "Produce the briefing."
}
```

```markdown
# Instructions

Topic: {{input.topic}}
```

**Example — workflow-tool parameters:**

```yaml
name: ask_question
workflow:
  code: WF-ASK-QUESTION
parameters:
  - name: question
    description: The question to answer.
    type: string
    required: true
```

```markdown
# Instructions

Answer this question directly and concisely:

{{input.question}}
```

**Example — `input-tools` parameter mapping (uses the same scope):**

```yaml
input-tools:
  - variable: seed_answer
    tool: ask_question
    parameters:
      question: "{{input.seed_question}}"
```

---

### 3.7 `blueprint`

| | |
| --- | --- |
| **Type** | Scalar + enumerable `categories` and `entries` |
| **Sort** | `blueprint.categories` by `Index` ascending; entries by title alphabetical within each category (same order in both lists) |
| **Resolution** | Automatic — authors do **not** pick the blueprint |

Template-scope resolution (EntityId only — UnitOfWork is not consulted):

1. Entity scoped blueprint when the run has an `EntityId` (from the entity type)
2. Brain-scoped blueprint when no `EntityId` is present
3. Empty (tags blank) if none match

| Path | Iteration | Fields |
| --- | --- | --- |
| `blueprint` | — | `blueprint.name`, `blueprint.description` |
| `blueprint.categories` | `{{#blueprint.categories}}...{{/blueprint.categories}}` | `category.name`, `category.description`, `category.entries` |
| `category.entries` | `{{#category.entries}}...{{/category.entries}}` | `entry.title`, `entry.version`, `entry.category` |
| `blueprint.entries` | `{{#blueprint.entries}}...{{/blueprint.entries}}` | `entry.title`, `entry.version`, `entry.category` |

Notes:

- `category.entries` and `blueprint.entries` share the **same entry field names**.
- `entry.category` is the parent category **name** (string), not a nested object.
- `entry.version` is the entry's centrality integer (default `1`).
- There is no `entry.code` — BlueprintEntry has no Code column.
- Empty categories still render (`category.name` / `category.description`); the
  `{{#category.entries}}` block simply produces no inner content.
- Prefer qualified prefixes inside loops (`{{entry.title}}`, `{{category.name}}`)
  so nested frames do not collide.

**Example — `blueprint.categories` with nested `category.entries`:**

```markdown
# Memory: {{blueprint.name}}
{{blueprint.description}}

{{#blueprint.categories}}
### {{category.name}}
{{category.description}}

{{#category.entries}}
- {{entry.title}} (v{{entry.version}}) — {{entry.category}}
{{/category.entries}}

{{/blueprint.categories}}
```

**Example — flat `blueprint.entries`:**

```markdown
# All entries
{{#blueprint.entries}}
- [{{entry.category}}] {{entry.title}} (v{{entry.version}})
{{/blueprint.entries}}
```

---

### 3.8 `inboxEntry`

| | |
| --- | --- |
| **Type** | Scalar bag |
| **Sort** | N/A |
| **Requires** | `InboxEntryId` on the run (set automatically for inbox-triggered workflows); otherwise the scope is empty |

| Field | Description |
| --- | --- |
| `inboxEntry.reference` | Short 8-character AI-facing identifier (`[a-z0-9]{8}`) |
| `inboxEntry.date` | Source-event timestamp (`yyyy-MM-dd HH:mm:ss UTC`) |
| `inboxEntry.source` | Free-text producing-system identifier (nullable) |
| `inboxEntry.title` | Entry title |
| `inboxEntry.body` | Full signal content (markdown) |
| `inboxEntry.status` | Lifecycle status (`PENDING`, `REVIEWING`, `APPLIED`, `DISMISSED`) |
| `inboxEntry.routingType` | Routing classification (nullable until triaged) |

Notes:

- Populated when a `TRIGGERED` workflow runs from an inbox task (Hangfire
  processor, `update_inbox_task` approval, or Execution API approve). Non-inbox
  runs leave this scope blank.
- **Do not** expect the entry body in the workflow input message. The task
  `Action` / input is **instructions-only**; entry content must be pulled in via
  these tags (see `WF-INBOX-ENTRY-CONTEXT`).

**Example:**

```markdown
# Inbox signal: {{inboxEntry.title}} ({{inboxEntry.reference}})

**Date:** {{inboxEntry.date}}
**Source:** {{inboxEntry.source}}
**Status:** {{inboxEntry.status}}
**Routing:** {{inboxEntry.routingType}}

## Body

{{inboxEntry.body}}
```

---

### 3.9 `inboxTasks`

| | |
| --- | --- |
| **Type** | Enumerable list at root |
| **Sort** | Creation order (oldest first) |
| **Requires** | `InboxEntryId` on the run; otherwise the scope is empty |

| Iteration | Item fields (short form inside the loop) |
| --- | --- |
| `{{#inboxTasks}}...{{/inboxTasks}}` | `reference`, `action`, `response`, `status`, `workflowCode`, `expertOpinion` |

Notes:

- Lists **all** tasks for the entry that triggered the run (not only the current
  task). Prefer `{{task.*}}` (below) when you need the triggering task alone.
- `workflowCode` is the linked workflow's `Code`, or blank when the task has no
  workflow (informational task).
- `action` is the task's instructions-only field (routing intent), not entry
  markdown.
- `response` is the final assistant reply from the linked workflow run, or blank
  until the task has executed.
- `expertOpinion` is human review notes, or blank when null.

**Example:**

```markdown
## Tasks on this entry
{{#inboxTasks}}
- `{{reference}}` — {{status}}{{#if workflowCode}} → {{workflowCode}}{{/if}}
  {{#if action}}Instructions: {{action}}{{/if}}
  {{#if response}}Response: {{response}}{{/if}}
  {{#if expertOpinion}}Expert opinion: {{expertOpinion}}{{/if}}
{{/inboxTasks}}
```

---

### 3.10 `task`

| | |
| --- | --- |
| **Type** | Scalar bag |
| **Sort** | N/A |
| **Requires** | `InboxTaskId` on the run (set automatically when a workflow is started from an inbox task); otherwise the scope is empty |

| Field | Description |
| --- | --- |
| `task.reference` | Short 8-character AI-facing identifier (`[a-z0-9]{8}`) |
| `task.action` | Instructions-only field (routing intent), not entry markdown |
| `task.response` | Final assistant reply from the linked run, or blank until executed |
| `task.status` | Lifecycle status (`PENDING`, `AWAITING_APPROVAL`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`) |
| `task.workflowCode` | Linked workflow `Code`, or blank when the task has no workflow |
| `task.expertOpinion` | Human expert review notes, or blank when null |

Notes:

- Populated for the **triggering** inbox task only (Hangfire processor,
  `update_inbox_task` approval, or Execution API approve). Sibling tasks remain
  available via `{{#inboxTasks}}`.
- Field names match the short-form item fields inside `{{#inboxTasks}}`, but are
  qualified with the `task.` prefix because this is a scalar bag, not a loop.
- Non-inbox runs leave this scope blank.

**Example:**

```markdown
## Current task

**Reference:** `{{task.reference}}`
**Status:** {{task.status}}
{{#if task.workflowCode}}**Workflow:** `{{task.workflowCode}}`{{/if}}
{{#if task.action}}**Instructions:** {{task.action}}{{/if}}

{{#if task.expertOpinion}}
### Expert Opinion

{{task.expertOpinion}}
{{/if}}
```

---

## 4. Contracts authors must know

1. **Silent blank** — unknown scopes, missing fields, null run ids, and typos all
   render as empty string. Do not expect an error signal for bad tags.
2. **Lazy load** — only scopes referenced in the content are fetched. Content with
   no `{{` tags passes through byte-identical (no scope providers called).
3. **Determinism** — respect the documented sort orders; do not assume another
   order when writing prompts that list collections.
4. **YAML block scalars** — for tool `response-markdown` / `error-markdown`, use
   `|` block literals so markdown stays YAML-safe.
5. **Where tags run**
   - Workflow Instructions (system prompt and invoking Instructions)
   - Tool `response-markdown` / `error-markdown` after a tool result

---

## 5. Quick reference

| Scope | Iteration | Scalar / item fields |
| --- | --- | --- |
| `entity` | — | `name`, `{variableKey}` |
| `unitOfWork` | `.context`, `.data` | `date`, `time`, `title`, `source`, `body` |
| `run` | — | `reference` (AI-facing run code for `set_run_grading`), `telemetry` (OTEL GenAI JSON for subject run) |
| `skillBooks` | root → `.categories` → `.skills` | `skillBook.*`, `category.*`, `skill.*` |
| `now` | — | `utcDate`, `utcTime`, `utcDayOfWeek`, `localDate`, `localTime`, `localDayOfWeek` |
| `result` | — | `{anyKey}` (flat) |
| `input` | — | `{key}` (flat; Execution API `variables` + workflow-tool / `run_workflow` params) |
| `blueprint` | `.categories` → `.entries`; `.entries` | `blueprint.name/description`; `category.name/description`; `entry.title/version/category` |
| `inboxEntry` | — | `reference`, `date`, `source`, `title`, `body`, `status`, `routingType` |
| `inboxTasks` | root | `reference`, `action`, `response`, `status`, `workflowCode`, `expertOpinion` |
| `task` | — | `reference`, `action`, `response`, `status`, `workflowCode`, `expertOpinion` |

| Tag | Syntax |
| --- | --- |
| Scalar | `{{scope.field}}` |
| Iteration | `{{#path}}...{{/path}}` |
| Conditional | `{{#if expr}}...{{/if}}` with `==`, `!=`, or truthiness |

---

## 6. Related examples

Reference workflows in `workflows/`:

| Workflow | Code | Demonstrates |
| --- | --- | --- |
| Skill Update | `WF-SKILL-UPDATE` | Nested `{{#skillBooks}}` → `{{#skillBook.categories}}` → `{{#category.skills}}`, plus find / load / create / update via schema tools |
| Unit of Work Context | `WF-UNIT-OF-WORK-CONTEXT` | Dual-block `{{#unitOfWork.context}}` and `{{#unitOfWork.data}}` with short-form fields; optional `{{entity.name}}` |
| Learning Eval (Run) | `WF-EVAL-RUN` | `{{run.reference}}` + `{{run.telemetry}}`; `set_run_grading` + `create_inbox_entry` (BRA207 / BRA406) |
| Inbox Entry Context | `WF-INBOX-ENTRY-CONTEXT` | `{{inboxEntry.*}}` / `{{task.*}}` scalars and `{{#inboxTasks}}` iteration for TRIGGERED inbox workflows; instructions-only input |
| Ask Question | `WF-ASK-QUESTION` | `{{input.question}}` from the `ask_question` workflow-tool parameters |
| Input Variables Demo | `WF-INPUT-VARIABLES` | Execution API `variables` → `{{input.*}}` plus `input-tools` pre-calling `ask_question` (BRA409) |

Also see tool `response-markdown` / `error-markdown` on declared tools (e.g.
salesmate `record_telemetry`) for the `result` scope. Full variables /
`input-tools` guide: **BRA409**.
