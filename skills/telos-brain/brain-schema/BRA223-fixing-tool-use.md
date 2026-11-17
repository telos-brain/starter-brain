---
name: Fixing Tool Use
code: BRA223
version: 1
description: When a model misuses a tool or cannot use the result, decide
  whether to fix the data the tool returns, the tool definition, or raise a
  task because the endpoint is outside the brain schema. Keep workflows short.
  Load when a run repeats a tool call, is confused by a result, or fails a
  parameter.
---

# Fixing Tool Use

When a model does not use a tool successfully, change the tool. Do not teach
the workaround in the workflow. Workflows stay short. A tool definition is a
standalone mini-skill: what the tool does, how to set its parameters, and how
to read what comes back. It speaks to the model, which is the caller. It does
not describe the implementation. Shape and tone: **BRA214**. Principle:
**BRA105**.

A repeated call is a sign the result was not usable. The workflow was often
fine. Do not add an instruction that says to call the tool only once.

## What we control

Ask whether the result is inside our control.

- **The tool definition is always inside our control.** It is part of the
  brain schema.
- **The data is inside our control when we own the software behind the
  tool.** A connector file does not say this. Ownership does. When we own
  it, and the model did not get what it asked for, improve the data the tool
  returns. A clearer result beats a longer workflow.
- **A third-party endpoint we do not own is outside the schema.** The fix is
  a task assigned to a person who can change that system. Do not cover it in
  the workflow. Creating and assigning that task: **BRA405**, **BRA408**.
  Assignment is an email the caller supplies after listing members. It does
  not by itself stop the task from running. A task runs on its own only when
  its linked workflow is eligible; otherwise it waits at `AWAITING_APPROVAL`
  (**BRA404**).

## Definition or data

Improve the **data** when the payload is confusing, incomplete, or scoped to
the wrong thing. The model then stops compensating.

Improve the **definition** when the model needs to know how to call the tool
or how to interpret a result it already returns clearly. Put on the parameter
the values that are actually accepted, and what a result means. If a failed
call comes back with the allowed values, that information belongs on the
parameter description, so the next run does not have to fail to learn it.
The definition should be enough for a successful call and a correct reading
of the result.

Trace backwards before editing the tool that failed. The failing call may
have been given a bad target by an earlier result. Fix the earlier result.

## Errors and parameters

Prefer a tool that succeeds.

- A mandatory parameter must be described well enough that the model is not
  guessing. Allowed values and the shape of the value belong in that
  description.
- An optional parameter with a bad value that does not matter should be
  dropped, or replaced with a supported value. The response says what was
  ignored or substituted.
- A genuine error is plain text. It says what was wrong and what to do next.
  It is not an HTML error page, and it is not a status code on its own.

What is mandatory stays strict. What is optional and harmless is absorbed.
The model should not be left to invent formats.

## Passing work to another workflow

A workflow tool's parameters are the contract the child receives. Each one
is available in the child as `{{input.<name>}}`, and the child workflow has
to use it (**BRA204**, **BRA214**).

Use separate parameters when the child needs specific fields. Use one
free-form parameter when the caller should be free to shape the request. A
free-form parameter is a choice to be flexible. It is not a substitute for a
field the child must have.

## See also

- **BRA214** — tool YAML, descriptions, parameters, workflow tools
- **BRA218** — audit and approval, which is separate from a bad result
- **BRA222** — when this diagnosis applies, and when the problem is the job
