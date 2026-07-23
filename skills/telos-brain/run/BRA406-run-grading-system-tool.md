---
name: Run Grading System Tool
code: BRA406
version: 1
description: The set_run_grading in-process system tool — how eval workflows
  persist a 0–100 quality score on a WorkflowRun (and optionally link an
  InboxEntry) so the admin console can show traffic-light grades and grading
  trends. Identity is by 8-character run reference from {{run.reference}}.
tools:
  - set_run_grading
---

# Run Grading System Tool

Eval workflows decide the score; this tool **only stores** it. After the model
applies its rubric, call `set_run_grading` once per subject run so operators can
see the grade on the run list / detail pages and on the workflow Grading tab
chart.

This is an ordinary `system` tool (BRA201 §5.2). Declaration lives under
`tools/inbox/set-run-grading.yml`. This skill lists it in frontmatter `tools:`
so a workflow that keeps it under `available-tools` can promote it via
`get_skill`.

For authoring the surrounding eval workflow, see **BRA207**. For creating the
linked inbox finding, see **BRA405** (`create_inbox_entry`).

---

## `set_run_grading`

| | |
| --- | --- |
| **Purpose** | Persist `WorkflowRuns.Grading` (0–100) and optional `InboxEntryId` |
| **When** | After the eval has scored the subject run (usually once, near the end) |
| **YAML** | `tools/inbox/set-run-grading.yml` |

### Parameters

| Parameter | Required | Notes |
| --- | --- | --- |
| `run_reference` | Yes | Subject run's **8-character reference** (`[a-z0-9]{8}`). In a `workflowrun:complete` eval, pass the literal value from **`{{run.reference}}`**. Do **not** use a shortened Guid from `{{run.telemetry}}` (`runId`) — that is not the AI-facing reference. |
| `grading` | Yes | Integer **0–100** inclusive (string or number JSON both accepted). |
| `inbox_entry_reference` | No | Reference of an InboxEntry that explains the grade (typically the primary learning just created). If supplied but not found, the tool errors and **writes nothing**. |

### Behaviour

- Single `UpdateRun`: sets `Grading` and optionally `InboxEntryId`.
- Brain scoping is automatic (EF global filter); never pass `BrainId` or UUIDs.
- Re-calling overwrites the previous grade / linked entry for that run.
- The tool does **not** invent a rubric — scoring lives in the eval instructions.

### Returns

A short confirmation, e.g. `Grading set to 85 on run 'rungrade'. Inbox entry 'abcd1234' associated.`

---

## Recommended eval call order

1. Read `{{run.telemetry}}` and apply the rubric (BRA207 §5).
2. Optionally create learnings with `create_inbox_entry` (BRA405). For a
   finding that only explains the grade (no inbox trigger workflows), prefer
   `status: PROCESSED`.
3. Call `set_run_grading` once with:
   - `run_reference`: `{{run.reference}}` (paste the rendered value into the
     tool call — template tags are resolved in Instructions, not inside tool
     argument JSON unless your harness substitutes them)
   - `grading`: the integer score
   - `inbox_entry_reference`: the reference returned by `create_inbox_entry`
     when you want the grade tag to open that finding

Canonical workflow: **`WF-EVAL-RUN`** (`workflows/WF-EVAL-RUN.md`).

---

## What operators see

| Surface | Behaviour |
| --- | --- |
| Run list / detail | Traffic-light **GradingTag** when `Grading` is non-null (green ≥ 80, orange 50–79, red &lt; 50). Click opens the linked InboxEntry when `InboxEntryId` is set. |
| Workflow detail → Grading tab | Daily average of graded runs as a time-series chart (`GET …/workflows/{code}/grading`). |

Ungraded runs (`Grading` null) show no tag and do not appear in the chart.

---

## Related skills

| Resource | Role |
| --- | --- |
| **BRA207** | Authoring learning-eval workflows that grade and call this tool |
| **BRA204** | `{{run.reference}}` / `{{run.telemetry}}` template tags |
| **BRA405** | `create_inbox_entry` and other inbox system tools |
| **BRA403** | Run OTEL shape and when a run becomes eval-eligible |
