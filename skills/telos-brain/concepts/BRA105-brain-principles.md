---
name: Brain Principles
code: BRA105
version: 3
description: High-level principles for organising a Telos Brain — skills,
  tools, configuration, memory, and LLM budgets. Always inject when triaging
  learnings or editing the brain.
---

# Brain Principles

- **Organise, don't accumulate.** Structure matters as much as content. Prefer
  high-quality, well-organised information. When you add text, consider if you need to remove something
  or split the skill.
- **Progressive disclosure.** Skills have codes and live in categories. Keep
  each skill short; split when one skill covers too much. Reference related
  skills by code rather than inlining them — that is how further depth is
  loaded on demand.
- **Tools are mini-skills.** Prefer fixing the tool definition (description,
  parameter names/descriptions, response and error templates with variables and
  conditionals) over teaching the tool only in a skill. Tool defs sit in the
  system prompt — keep them tight, but make them sufficient.
- **Don't overfit one run.** Workflows repeat. Take the learning, but do not
  reconfigure for a single eval or flip-flop between one-off fixes.
- **Categories are the lens.** Skill-book and blueprint category descriptions
  decide what belongs. Extract signal; discard noise. Skills hold transferable
  practices and processes — not personal or customer data. Memory holds scoped
  facts; match brain vs entity scope.
- **Manage the LLM; don't let the LLM manage us.** We give the model a budget
  and we enforce it. Use the boundaries we have: daily and monthly spend limits,
  max turns, output-token and thinking-token budgets, and run-rate caps (per
  hour or per minute). Tokens are an allocation, not an entitlement — raise a
  limit only when the work needs more, never because the model asked.
