---
name: Improving a Brain from a Learning
code: BRA222
version: 2
description: How to turn an eval learning into the right change. A learning
  entry is one data point, not a fix. Triage raises weight; a task is created
  only when the problem is trusted. Choose the workflow, the tool definition,
  the data the tool returns, or a task for a person. Load when triaging a
  learning or deciding what to change after a bad run.
---

# Improving a Brain from a Learning

An eval does two jobs. It records a grade, so performance can be tracked over
time, and it raises a learning entry. The grade is the score. The entry is
the data point. Authoring the eval is **BRA207**. Weight, clustering, and
when a task may be created are **BRA414**.

## A learning is not a conclusion

One run is out of context. The entry says something did not go well. It does
not say whether that was a one-off, a recurring problem, or still unknown.
Do not choose a fix from a single entry. Do not rewrite a workflow to cover
one trace.

Triage groups related entries and raises their weight. That work stays in
problem territory: a recurring problem becomes visible, and its weight goes
up. A task is created only when the weight is high enough to act
(**BRA414**). Until then, leave it.

A task is one distinct change. It may run on its own, wait for approval, or
be assigned to a person. Anything that has to change outside the brain schema
— an API, a missing endpoint, a bug in software the schema only calls — is a
task for a person. The schema can change a workflow, a skill, a tool
definition, and the instructions those carry. It cannot change an endpoint
the brain does not own.

## Read the trace before the symptom

The call that failed is often not the cause. Walk back to the result that
led the model to make that call. A later error is frequently the result of
an earlier tool response that handed the model the wrong thing to do next.

Read the thinking as well as the tool calls. The model is in a stream of
consciousness. It may not know that something is missing. The trace shows
it. An eval receives the same run document as Export telemetry, and each
message's `thinking` is the full captured text. If the workflow had thinking
off, the model may still have thought, and that text was never stored. Turn
thinking to `adaptive` before treating an empty, truncated turn as
unexplained (**BRA225**).

## Where the fix belongs

Ask these in order. Stop at the first layer that owns the problem.

1. **Was the job possible?** A thinking loop — the model cannot decide what
   to do next — often means the job cannot be done with what it was given.
   That may be a missing tool, a guardrail, or a system prompt that does not
   fit the workflow. **BRA224**.
2. **Did a tool return something the model could not use?** Prefer a better
   result, then a better tool definition. Do not add workflow text to explain
   a bad result. **BRA223**.
3. **Is the job too big for one turn?** A long run of tool calls that exhausts
   output or context is a design problem. Divide it. **BRA225**.

## Iterate

Fixing one problem exposes the next. That is the point of a high learning
rate. Eval, change the layer that owns the problem, then eval again. Do not
stack instructions onto the workflow for each new symptom. Keep the workflow
short. Put durable knowledge of how to use a tool on the tool definition,
which is a mini-skill for the model (**BRA105**, **BRA214**).

## See also

- **BRA105** — tools are mini-skills; do not overfit one run
- **BRA207** — authoring the eval that grades and raises the entry
- **BRA223** — tool result, tool definition, or a task for a person
- **BRA224** — impossible jobs, system prompts, missing tools, guardrails
- **BRA225** — output budgets, visible thinking, sub-workflows
- **BRA403** — OTEL telemetry, including thinking text
- **BRA405** / **BRA408** — creating a task and assigning it to a person
