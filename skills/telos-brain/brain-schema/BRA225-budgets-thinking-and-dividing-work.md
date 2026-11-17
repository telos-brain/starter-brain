---
name: Budgets, Thinking, and Dividing Work
code: BRA225
version: 1
description: How to set output-token budgets and thinking so a run can be
  diagnosed, and how to split a large job across turns or a sub-workflow.
  Load when a run hits an output limit, returns nothing, or tries to do a
  long series of tool calls in one chat turn.
---

# Budgets, Thinking, and Dividing Work

Cost defaults for frontier models stay in **BRA212**: a short output cap, a
deliberate retry, and thinking effort kept in check. This skill is the other
case — cheaper models, invisible thinking, and jobs that are too big for one
turn.

## Output budget follows the model

Output budget matters most on an expensive model. Keep that cap low, and add
a retry only when truncation is expected.

A cheaper or less capable model, including a small hosted model and a local
model, gets more of its quality from thinking. It thinks whether or not the
workflow captures that thinking. `thinking: none` leaves that text unstored.
A turn that stops at the output cap with an empty reply is often thinking
that filled the cap and was never written down. You cannot diagnose what you
cannot read.

For those models:

- Set `thinking: adaptive` so the thinking is captured.
- Set `thinking-effort` to `low`, `medium`, `high`, or `xhigh` when you want
  less or more of that thinking. `adaptive` without an effort uses the
  provider default (**BRA217**).
- Give a higher output budget, because thinking shares the cap with the
  reply. A practical starting list is `10000, 25000, 50000`: the first cap
  is the normal attempt, and the later values are retries when a turn stops
  at the cap.

Frontier models stay on the short budget in **BRA212**. Do not copy the high
budget onto them.

## A large job is a design problem

Raising the budget is not the fix when a chat is handed a job that needs a
long series of tool calls. One turn that tries to do all of them will exhaust
output or context.

Batch the tool calls. In an asynchronous workflow, one batch ends the turn
and the workflow can continue with the next batch. In a chat, a batch ends
the turn and the person is waiting, so the parent cannot grind through the
whole job itself.

For a large chat job, add a sub-workflow and call it through a workflow tool
(**BRA214**). The parent divides the work. The child does one portion.

Choose the execution mode from the dependency.

- **Synchronous** when a later call must see the earlier result before it
  starts.
- **Asynchronous** when the portions are independent and can run together.

The child's parameters are the contract. How to shape them: **BRA223**.

## See also

- **BRA212** — cost defaults, caching, cheaper models, spend limits
- **BRA217** — `output-tokens`, `thinking`, `thinking-effort`, `max-turns`
- **BRA210** — how each provider applies thinking
- **BRA214** — workflow tools and `execution-mode`
- **BRA403** — thinking text on the OTEL span
