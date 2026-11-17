---
name: System Prompts, Tools, and Guardrails
code: BRA224
version: 1
description: When a run loops in its thinking or cannot do the job, decide
  whether the system prompt clashes with the workflow, a tool is missing, or
  the request is outside a guardrail. Load when a workflow stalls, repeats
  itself, or cannot do what it was asked.
---

# System Prompts, Tools, and Guardrails

A thinking loop is the model being unable to decide the next step. It shows
up in the thinking, not only in repeated tool calls. The usual cause is that
the job, as given, cannot be done.

Read the thinking. A line that says the model has no way to do what was asked
is evidence of a missing capability. The model will not always notice the
gap. The trace will. How thinking is stored: **BRA225**, **BRA403**.

## System prompt

A system prompt is shared base context for the workflows that use it. It is
the wrong place for instructions that only some workflows need.

For every workflow, ask two questions. Is this the right system prompt? Does
this workflow need one at all?

A short workflow whose own instructions are enough — a lookup, a summary, a
single transformation — is better with no system prompt, or with a short one
written for that kind of job. A prompt that requires skills, tools, or a
procedure the workflow does not use will stall the model. Use one prompt for
larger jobs that share that base, and a simpler prompt, or none, for the
short workflows. Authoring `system-prompt-code`: **BRA217**.

## Missing tool or guardrail

The workflow is the job. It also carries the data the caller supplied. A
caller can ask for something this workflow is not allowed to do. That is a
guardrail, not a missing tool.

Ask which of these is true.

- **Guardrail.** The request is out of bounds on purpose. Do not add the
  tool. The workflow should know how to answer when it hits that boundary.
- **Forgotten tool.** The job needs it, the definition already exists, and
  it is safe to offer. Add it to the workflow. A create without an update,
  or a list without a read, produces duplicates and loops, because the only
  available action is the wrong one.
- **No definition, or no endpoint.** The capability is not in the schema, or
  the API does not exist. That change is outside the schema. Raise a task
  and assign it to a person (**BRA405**, **BRA408**). Do not invent the
  behaviour in the workflow.

We do not give the model every tool it asks for. The question is whether to
change the job, or to give it the tools the job requires.

## See also

- **BRA217** — `system-prompt-code`, `tools`, `available-tools`
- **BRA214** — what a tool definition must contain
- **BRA222** — the wider order of diagnosis
- **BRA411** / **BRA415** — discovering and promoting tools already in the pool
