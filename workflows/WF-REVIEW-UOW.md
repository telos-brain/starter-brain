---
name: Review Unit of Work
code: WF-REVIEW-UOW
description: >-
  On unit-of-work completion, detects domain concepts that fit blueprint
  categories and creates a PROCESSED inbox entry with one review_blueprint task
  per learning. Does not grade agent quality (see WF-EVAL-RUN).
version: 1
type: TRIGGERED
trigger: unitofwork:complete:low
model: anthropic/claude-sonnet-4-6
system-prompt-code: WF-SYSTEM-PROMPT

output-tokens: 4096, 8192
caching: automatic
max-turns: 30
thinking: effort
max-runs-per-hour: 500

tools:
  - create_inbox_entry
  - add_inbox_task
---

# Instructions

You extract **domain knowledge** learnings from a completed unit of work and
route them into blueprint memory. This is not a session grade — do **not**
create skill, workflow, tool, or system-change learnings (those belong to
`WF-EVAL-RUN`).

Only capture concepts that **clearly fit** one of the blueprint categories
below. Do not force-fit. If nothing clearly fits, create **no** inbox entry and
reply with a single line: `No blueprint learnings.`

## Blueprint categories

Categories in the current scope (entity-scoped when an entity is present on the
run; otherwise brain-global). Match learnings only to these:

<blueprint_categories>
{{#blueprint.categories}}
- **{{category.name}}** — {{category.description}}
{{/blueprint.categories}}
</blueprint_categories>

## Unit of work telemetry

<unit_of_work_context>
{{#unitOfWork.context}}
### {{date}} {{time}} — {{title}} ({{source}})
{{body}}

{{/unitOfWork.context}}
</unit_of_work_context>

<unit_of_work_data>
{{#unitOfWork.data}}
### {{date}} {{time}} — {{title}} ({{source}})
{{body}}

{{/unitOfWork.data}}
</unit_of_work_data>

## Process

1. Read the categories and the unit-of-work context/data carefully.
2. List candidate domain concepts (vocabulary, processes, client facts,
   operational patterns, financial terms, etc.) that are evidenced in the
   telemetry **and** clearly belong to a listed category. One concept may map to
   one category only. Prefer many precise tasks over a few vague ones — a rich
   unit of work may legitimately produce 20–30 tasks. Do not invent concepts
   that are not supported by the telemetry.
3. If the candidate list is empty: stop. Do not call any tools. Reply
   `No blueprint learnings.`
4. If there are candidates:
   a. Call `create_inbox_entry` **once** with:
      - `title` — short summary, e.g. `Blueprint learnings from unit of work`
      - `body` — markdown list of every learning you will task (category +
        concept), for auditability
      - `routing_type` — `MEMORY_UPDATE`
      - `status` — `PROCESSED` (required — skips entry-create inbox triggers;
        tasks below drive `WF-REVIEW-BLUEPRINT`)
      - `source` — `WF-REVIEW-UOW`
   b. From the tool result, take the new entry's **8-character reference**.
   c. For **each** candidate learning, call `add_inbox_task` once with:
      - `inbox_entry_reference` — that reference
      - `workflow_code` — `WF-REVIEW-BLUEPRINT`
      - `instructions` — exactly this format (em dash):
        `review blueprint: {category name} — {short concept description}`
        Example: `review blueprint: Business Concepts — billboard sites`
5. Reply with one or two lines: how many tasks you created. No preamble.
