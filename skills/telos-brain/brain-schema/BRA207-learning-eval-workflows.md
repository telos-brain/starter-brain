---
name: Learning Eval Workflows
code: BRA207
version: 3
description: How to author TRIGGERED learning-eval workflows that grade a
  completed unit of work or workflow run, inject telemetry via template tags,
  persist a 0–100 score with set_run_grading, and create inbox learnings with
  create_inbox_entry — including manual Run eval and automatic enqueue modes
  (BRA091 / BRA112).
---

# Learning Eval Workflows

Learning evals sit in the middle learning loop: after work finishes, an eval
workflow grades the session, records a **0–100 quality score** on the subject
run, extracts actionable learnings, and posts each one to the **inbox**. Nothing
is applied automatically.

There are two eval surfaces:

| Surface | Trigger | Telemetry in instructions | Typical code |
| --- | --- | --- | --- |
| Unit of work | `unitofwork:complete` | `{{#unitOfWork.context}}` / `{{#unitOfWork.data}}` (BRA204) | `WF-EVAL` |
| Workflow run | `workflowrun:complete` | `{{run.telemetry}}` + `{{run.reference}}` (BRA204) | `WF-EVAL-RUN` |

This skill focuses on **authoring** those workflows in the brain schema. For the
`set_run_grading` tool contract, see **BRA406**. For the inbox lifecycle after
learnings land, see **BRA404** / **BRA405**. For run OTEL shape, see **BRA403**.
For all template tags, see **BRA204**.

---

## 1. Manual workflow-run eval (recommended starting point)

Use this when you want an admin to click **Run eval** on a Completed or Failed
run in the brain UI, rather than grading every run automatically.

### 1.1 Frontmatter checklist

Create a markdown file under `workflows/` (e.g. `wf-eval-run.md`):

```markdown
---
name: Learning Eval (Run)
code: WF-EVAL-RUN
type: TRIGGERED
version: 1
description: Grades a completed workflow run, records the score, and creates inbox learnings.
model: anthropic/claude-sonnet-4-6
system-prompt-code: <your-system-prompt-workflow-code>

trigger: workflowrun:complete
trigger-mode: manual

output-tokens: 4096, 8192
caching: automatic
max-turns: 15
thinking: effort
max-runs-per-hour: 500

tools:
  - create_inbox_entry
  - set_run_grading
---
```

| Field | Required value | Why |
| --- | --- | --- |
| `type` | `TRIGGERED` | Eval is event-driven, not a chat session |
| `trigger` | `workflowrun:complete` | Matches the run-eval Hangfire path (BRA091) |
| `trigger-mode` | `manual` (or omit — null is treated as manual) | Shows **Run eval** on the run detail page; does **not** auto-enqueue |
| `tools` | must include `create_inbox_entry` and `set_run_grading` | System tools (BRA405 / BRA406) — in-process |
| `max-runs-per-hour` | elevated (e.g. `500`) | Avoids throttling under batch review |

### 1.2 Instructions body

Inject the subject run's reference and OTEL telemetry, then instruct the model
to grade, create learnings, and **persist the score**:

```markdown
# Instructions

You are evaluating a single completed workflow run. Ground every claim in the
telemetry below. The input message is only a short trigger.

## Subject run

Reference (use this exact value for set_run_grading): {{run.reference}}

## Run telemetry

{{run.telemetry}}

1. Reconstruct what was asked, what the agent did (including tool calls), and
   how it turned out.
2. Assign an integer grade 0–100 against expected outcomes.
3. Identify discrete, actionable learnings. If nothing to improve, create no
   entries.
4. For each learning, call `create_inbox_entry` exactly once with:
   - `title`, `body`, `routing_type`
   - optional `status: PROCESSED` when the finding should not fire inbox
     trigger workflows (typical for grade-linked findings)
5. Call `set_run_grading` exactly once with:
   - `run_reference` — the subject reference shown above ({{run.reference}})
   - `grading` — the integer 0–100
   - `inbox_entry_reference` — optional; the primary learning's reference
6. Reply with a one- or two-line grade summary and how many learnings you recorded.
```

Canonical reference workflow: **`WF-EVAL-RUN`** in `workflows/WF-EVAL-RUN.md`.

### 1.3 System tools

#### `create_inbox_entry` (BRA405)

**In-process** via `IInboxService` — no outbound HTTP. Declare under
`tools/inbox/create-inbox-entry.yml`.

| Parameter | Notes |
| --- | --- |
| `title`, `body`, `routing_type` | Required |
| `source` | Optional producing-system label |
| `status` | Optional: `PENDING` (default — triggers fire, then auto-`PROCESSED`) or `PROCESSED` (skip triggers) |

#### `set_run_grading` (BRA406)

Persists the quality score on the **subject** WorkflowRun. Declare under
`tools/inbox/set-run-grading.yml`.

| Parameter | Notes |
| --- | --- |
| `run_reference` | Required — use **`{{run.reference}}`**, not a shortened Guid from telemetry |
| `grading` | Required — integer 0–100 |
| `inbox_entry_reference` | Optional — links the grade tag to an InboxEntry |

### 1.4 Deploy and run

1. Deploy the brain schema (CLI or Management API) so the workflow and tools are
   uploaded. Bump `version` when changing an existing eval.
2. Open a **Completed** or **Failed** workflow run in the admin UI
   (`/brains/{instance}/runs/{runId}`).
3. Click **Run eval** (visible when at least one `workflowrun:complete` workflow
   with manual / null `trigger-mode` exists).
4. When the eval finishes: learnings appear in the inbox; the run shows a
   traffic-light grade when `set_run_grading` succeeded.

Re-evaluation is allowed — the button can be used again on the same run
(overwrites the previous grade).

---

## 2. Automatic workflow-run eval

Same as §1, but set:

```markdown
trigger: workflowrun:complete
trigger-mode: automatic
```

**Behaviour:**

- When a workflow run reaches `Completed` (one-shot finish, session `complete`,
  or inactivity timeout), every matching **automatic** eval is enqueued.
- Runs of workflows that themselves have `trigger: workflowrun:complete` are
  **not** auto-evaluated (prevents eval-of-eval loops).
- Prefer fixing known issues before enabling automatic mode — continuous evals
  against an unfixed problem spam the inbox with the same learning.

The admin **Run eval** button is driven only by **manual** (or null) evals. You
may keep both a manual and an automatic eval workflow if you need both paths.

---

## 3. Unit-of-work eval (existing path)

Still supported and independent of run evals:

```markdown
---
code: WF-EVAL
type: TRIGGERED
trigger: unitofwork:complete
# trigger-mode is not used by the UoW Hangfire path today
---
```

Instructions use unit-of-work logs, not OTEL. `set_run_grading` applies to
**WorkflowRuns** — use it on the run-eval path (`WF-EVAL-RUN`), not the UoW
convention path, unless you also have a subject run reference.

Enqueued when `POST /units-of-work/{id}/complete` succeeds. The engine looks up
workflow code **`WF-EVAL`** by convention. Prefer a separate code
(`WF-EVAL-RUN`) for run evals so the two paths do not collide.

---

## 4. Template scopes used by evals

| Tag | Scope | When it resolves |
| --- | --- | --- |
| `{{run.reference}}` | `run` | Subject run's 8-character AI-facing reference (for `set_run_grading`) |
| `{{run.telemetry}}` | `run` | Compacted OTEL GenAI JSON for the subject Completed run |
| `{{#unitOfWork.context}}` / `{{#unitOfWork.data}}` | `unitOfWork` | Eval run request carries `UnitOfWorkId` |

Unresolvable tags render blank (BRA204 silent-blank contract). Always put the
telemetry blocks in the Instructions, not only in the short trigger message.

**Important:** `runId` inside `{{run.telemetry}}` is a **shortened Guid**, not
`WorkflowRuns.Reference`. Always use `{{run.reference}}` for `set_run_grading`.

---

## 5. Grading guidance (prompt design)

Good eval instructions:

- Require evidence from the injected telemetry / logs
- Ask for an explicit **integer 0–100** and call `set_run_grading` once
- Ask for **discrete** learnings (one inbox entry each)
- Allow "nothing to improve" with zero `create_inbox_entry` calls — still call
  `set_run_grading` when you have a score
- Map each learning to a single `routing_type`
- Use a capable model and enough `output-tokens` / `thinking` for reflection

Suggested score bands (aligned with the admin UI traffic-light tag):

| Band | Meaning |
| --- | --- |
| 80–100 | Met outcomes cleanly |
| 50–79 | Partial / recoverable issues |
| 0–49 | Missed outcomes or serious errors |

Avoid:

- Dumping the same learning twice
- Inventing failures not present in the logs
- Passing a shortened Guid as `run_reference`
- Auto-applying schema or code changes from the eval (humans apply via inbox)

---

## 6. Related skills and examples

| Resource | Role |
| --- | --- |
| **BRA201** §8 | Workflow frontmatter (`type`, `trigger`, `trigger-mode`, tools) |
| **BRA204** | `run.reference`, `run.telemetry`, `unitOfWork.*` tag taxonomy |
| **BRA403** | OTEL run telemetry shape; session close → eligible for eval |
| **BRA404** | Inbox HTTP surface; inbox trigger stages (entry create vs task auto-run) |
| **BRA405** | Inbox system tools (`create_inbox_entry`, `add_inbox_task`, …) |
| **BRA406** | `set_run_grading` contract and operator surfaces |
| `workflows/WF-EVAL-RUN.md` | Canonical manual run-eval workflow |
| Salesmate `wf-eval.md` / `wf-eval-run.md` | Sample brain copies |
