---
name: Review Blueprint
code: WF-REVIEW-BLUEPRINT
description: >-
  Applies one blueprint memory write from a review_blueprint inbox task —
  searches for a close match, then either merges into an existing entry or
  creates a new one. Never both.
version: 3
model: anthropic/claude-sonnet-4-6

# Tasks are created by WF-REVIEW-UOW / WF-TRIAGE via add_inbox_task with
# workflow_code WF-REVIEW-BLUEPRINT. An inbox: trigger with :low enables
# Hangfire stage-2 auto-run when brain learning-mode is low or higher (BRA404).
# Routing segment is ignored at stage 2; MEMORY_UPDATE matches stage-1 entries
# that arrive already classified as memory learnings.
type: TRIGGERED
trigger: inbox:MEMORY_UPDATE:low

system-prompt-code: WF-SYSTEM-PROMPT

output-tokens: 4096, 8192
caching: automatic
max-turns: 15
thinking: effort
max-runs-per-hour: 500

tools:
  - search_blueprint_entries
  - get_blueprint_entry
  - add_blueprint_entry
  - update_blueprint_entry
---

# Instructions

You apply **exactly one** blueprint write for a single `review_blueprint` task.
Fully autonomous — do not ask questions or wait for confirmation. Do **not**
update the inbox entry status.

Blueprint scope (entity vs brain-global) is resolved automatically from the run
context — never pass scope to tools.

## This task

**Reference:** `{{task.reference}}`
{{#if task.action}}**Instructions:** {{task.action}}{{/if}}

{{#if task.expertOpinion}}
### Expert Opinion

The following expert input has been provided for this task:

{{task.expertOpinion}}
{{/if}}

## Task to process

Parse **category** and **concept** from **this task's** instructions. Format
(em dash):

```text
review blueprint: {category name} — {short concept description}
```

After the first ` — ` (space-em-dash-space):

- **Category** = text between `review blueprint:` and the em dash (trim)
- **Concept** = text after the em dash (trim) — this is the concept description;
  use a short title derived from it when creating (first phrase / main noun
  phrase, not the whole essay)

If you cannot parse a category and concept, stop with no tool writes and say so
in one line.

## Optional entry context

Use the parent entry body only as supporting evidence for the merge/create —
the task instructions remain authoritative for category and concept.

- **Reference:** {{inboxEntry.reference}}
- **Title:** {{inboxEntry.title}}

{{#if inboxEntry.body}}
<entry-body>
{{inboxEntry.body}}
</entry-body>
{{/if}}

## Process (one write only)

1. Call `search_blueprint_entries` with:
   - `query` = the concept description
   - `category` = the parsed category name
   - `max_results` = a small number (e.g. 5)
2. If search returns candidates that look like a **close match** (same concept,
   not merely the same category): call `get_blueprint_entry` for the best match
   (`category` + exact `title`). Then call **`update_blueprint_entry` once**:
   - Merge the new concept information into the existing content
   - Preserve existing knowledge — do not wholesale replace
   - `old_str` must appear exactly once; include enough context to be unique
3. If there is **no** close match: call **`add_blueprint_entry` once** with:
   - `category` = parsed category
   - `title` = short concept name (unique within the category)
   - `content` = markdown grounded in the concept description (and entry body
     if helpful)
4. **Never** call both `update_blueprint_entry` and `add_blueprint_entry` in the
   same run. **Never** perform a second write after the first succeeds.
5. Reply in one or two lines: created vs updated, category, and entry title.

## Rules

- Always search (and load a candidate) before creating — no duplicate titles /
  duplicate concepts
- Prefer update when an existing entry already covers the same concept
- Prefer create when results are only loosely related or empty
- Prefer a missed write over inventing facts not supported by the task /
  entry
- Do not edit skills, workflows, tools, or inbox status
