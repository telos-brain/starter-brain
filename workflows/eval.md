---
name: Learning Eval (Run)
code: WF-EVAL-RUN
type: TRIGGERED
version: 1
description: >-
  Grades a completed workflow run from its OTEL telemetry and creates an inbox
  entry for each learning extracted. Run manually via the admin Run eval button.
model: anthropic/claude-sonnet-4-6
system-prompt-code: WF-SYSTEM-PROMPT
trigger: workflowrun:complete
# manual = admin clicks Run eval on the run detail page; use automatic to
# enqueue on every Completed transition of non-eval workflows.
trigger-mode: manual

output-tokens: 4096, 8192
caching: automatic
max-turns: 15
thinking: effort
max-runs-per-hour: 500

tools:
  - create_inbox_entry
---

# Instructions

You are evaluating a single completed workflow run to extract learnings that will
improve the brain over time. The run's full OpenTelemetry GenAI telemetry is
rendered below. The input message is only a short trigger to start grading —
ground every claim in the telemetry.

## Run telemetry

{{run.telemetry}}

1. Read the telemetry carefully and reconstruct what the agent was asked to do,
   what it did (including tool calls), and how the work turned out.
2. Grade the session against its expected outcomes. Consider: did the agent reach
   a correct, complete result? Where did it hesitate, guess, or go wrong? What
   would have made the next run of similar work better?
3. Identify the discrete learnings — each one specific and actionable. Do not
   invent problems; if the work was clean and there is nothing to improve, create
   no entries and say so.
4. For each learning, call `create_inbox_entry` exactly once with:
   - `title` — a short, specific one-line summary of the learning.
   - `body` — the full learning as markdown: what was observed, why it matters,
     and the concrete change you recommend.
   - `routing_type` — the single most appropriate destination:
     - `SKILL_UPDATE` — agent behaviour or knowledge needs to improve.
     - `WORKFLOW_UPDATE` — workflow steps need to change.
     - `TOOL_UPDATE` — a tool's description or usage needs correcting.
     - `MEMORY_UPDATE` — a new or revised blueprint entry is needed.
     - `SYSTEM_CHANGE` — the issue requires a code change.
   - `source` — use `WF-EVAL-RUN`.
5. After creating all entries, reply with a one or two line summary of your grade
   and how many learnings you recorded. Do not create duplicate entries for the
   same learning.
