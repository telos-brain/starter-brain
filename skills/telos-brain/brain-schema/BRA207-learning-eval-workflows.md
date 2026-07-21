---
name: Learning Eval Workflows
code: BRA207
version: 2
description: How to author TRIGGERED learning-eval workflows that grade a
  completed unit of work or workflow run, inject telemetry via template tags,
  and create inbox learnings with the create_inbox_entry system tool — including
  manual Run eval and automatic enqueue modes (BRA091).
---

# Learning Eval Workflows

Learning evals sit in the middle learning loop: after work finishes, an eval
workflow grades the session, extracts actionable learnings, and posts each one
to the **inbox** as a `PENDING` entry for human review. Nothing is applied
automatically.

There are two eval surfaces:

| Surface | Trigger | Telemetry in instructions | Typical code |
| --- | --- | --- | --- |
| Unit of work | `unitofwork:complete` | `{{#unitOfWork.context}}` / `{{#unitOfWork.data}}` (BRA204) | `WF-EVAL` |
| Workflow run | `workflowrun:complete` | `{{run.telemetry}}` — full OTEL GenAI JSON (BRA204) | `WF-EVAL-RUN` |

This skill focuses on **authoring** those workflows in the brain schema. For the
inbox lifecycle after learnings land, see **BRA404**. For run OTEL shape, see
**BRA403**. For all template tags, see **BRA204**.

---

## 1. Manual workflow-run eval (recommended starting point)

Use this when you want an admin to click **Run eval** on a completed run in the
brain UI, rather than grading every run automatically.

### 1.1 Frontmatter checklist

Create a markdown file under `workflows/` (e.g. `wf-eval-run.md`):

```markdown
---
name: Learning Eval (Run)
code: WF-EVAL-RUN
type: TRIGGERED
version: 1
description: Grades a completed workflow run and creates inbox learnings.
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
---
```

| Field | Required value | Why |
| --- | --- | --- |
| `type` | `TRIGGERED` | Eval is event-driven, not a chat session |
| `trigger` | `workflowrun:complete` | Matches the run-eval Hangfire path (BRA091) |
| `trigger-mode` | `manual` (or omit — null is treated as manual) | Shows **Run eval** on the run detail page; does **not** auto-enqueue |
| `tools` | must include `create_inbox_entry` | System tool (BRA405) — in-process inbox create |
| `max-runs-per-hour` | elevated (e.g. `500`) | Avoids throttling under batch review |

### 1.2 Instructions body

Inject the subject run's OTEL telemetry with the `run` scope, then instruct the
model to grade and call `create_inbox_entry` once per learning:

```markdown
# Instructions

You are evaluating a single completed workflow run. Ground every claim in the
telemetry below. The input message is only a short trigger.

## Run telemetry

{{run.telemetry}}

1. Reconstruct what was asked, what the agent did (including tool calls), and
   how it turned out.
2. Grade against expected outcomes.
3. Identify discrete, actionable learnings. If nothing to improve, create no
   entries and say so.
4. For each learning, call `create_inbox_entry` exactly once with:
   - `title` — one-line summary
   - `body` — markdown: observation, why it matters, recommended change
   - `routing_type` — one of `SKILL_UPDATE`, `WORKFLOW_UPDATE`, `TOOL_UPDATE`,
     `MEMORY_UPDATE`, `SYSTEM_CHANGE`
5. Reply with a one- or two-line grade summary and how many learnings you recorded.
```

Canonical reference workflow: **`WF-EVAL-RUN`** in `workflows/WF-EVAL-RUN.md`.

### 1.3 System tool: `create_inbox_entry` (preferred)

**`create_inbox_entry` is an in-process system tool** (BRA405). The Tool Router
resolves it without any outbound HTTP call — no API key, no host URL, no
`localhost` / connection-refused failures.

Declare it in YAML under `tools/inbox/`:

```yaml
name: create_inbox_entry
system:
  tool: create_inbox_entry
```

| Kind | Creates inbox entries? | How |
| --- | --- | --- |
| **System tool `create_inbox_entry`** (preferred) | Yes | In-process via `IInboxService` (same semantics as `POST /inbox`) |
| Execution API `POST /inbox` (BRA404) | Yes | For external harnesses / HTTP clients — not needed inside eval workflows |

Minimum parameters the model must supply:

| Parameter | Notes |
| --- | --- |
| `title` | Required |
| `body` | Required (markdown) |
| `routing_type` | Required for triage (`SKILL_UPDATE`, …) |
| `source` | Optional producing-system label |

Entries are created in **PENDING** status for human review (BRA404). Matching
`inbox:*` / `inbox:<RoutingType>` TRIGGERED workflows still get tasks atomically.

### 1.4 Deploy and run

1. Deploy the brain schema (CLI or Management API) so the workflow and tool are
   uploaded. Bump `version` when changing an existing eval.
2. Open a **Completed** workflow run in the admin UI (`/brains/{instance}/runs/{runId}`).
3. Click **Run eval** (visible when at least one `workflowrun:complete` workflow
   with manual / null `trigger-mode` exists).
4. Learnings appear in the inbox when the eval finishes.

Re-evaluation is allowed — the button can be used again on the same run.

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

Instructions use unit-of-work logs, not OTEL:

```markdown
{{#unitOfWork.context}}
### {{date}} {{time}} — {{title}} ({{source}})
{{body}}
{{/unitOfWork.context}}
{{#unitOfWork.data}}
### {{date}} {{time}} — {{title}} ({{source}})
{{body}}
{{/unitOfWork.data}}
```

Enqueued when `POST /units-of-work/{id}/complete` succeeds. The engine looks up
workflow code **`WF-EVAL`** by convention. Prefer a separate code
(`WF-EVAL-RUN`) for run evals so the two paths do not collide.

---

## 4. Template scopes used by evals

| Tag | Scope | When it resolves |
| --- | --- | --- |
| `{{run.telemetry}}` | `run` | Eval run request carries `SubjectRunId` (the Completed run being graded) |
| `{{#unitOfWork.context}}` / `{{#unitOfWork.data}}` | `unitOfWork` | Eval run request carries `UnitOfWorkId` |

Unresolvable tags render blank (BRA204 silent-blank contract). Always put the
telemetry blocks in the Instructions, not only in the short trigger message.

---

## 5. Grading guidance (prompt design)

Good eval instructions:

- Require evidence from the injected telemetry / logs
- Ask for **discrete** learnings (one inbox entry each)
- Allow "nothing to improve" with zero `create_inbox_entry` calls
- Map each learning to a single `routing_type`
- Use a capable model and enough `output-tokens` / `thinking` for reflection

Avoid:

- Dumping the same learning twice
- Inventing failures not present in the logs
- Auto-applying schema or code changes from the eval (humans apply via inbox)

---

## 6. Related skills and examples

| Resource | Role |
| --- | --- |
| **BRA201** §8 | Workflow frontmatter (`type`, `trigger`, `trigger-mode`, tools) |
| **BRA204** | `run.telemetry`, `unitOfWork.*` tag taxonomy |
| **BRA403** | OTEL run telemetry shape; session close → eligible for eval |
| **BRA404** | Inbox entries created by `create_inbox_entry` |
| **BRA202** | Secret injection for the inbox API tool |
| `workflows/WF-EVAL-RUN.md` | Canonical manual run-eval workflow |
| Salesmate `wf-eval.md` / `wf-eval-run.md` | Sample brain copies |
