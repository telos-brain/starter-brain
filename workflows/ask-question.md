---
name: Ask Question
code: WF-ASK-QUESTION
description: >-
  Answers a single question by searching the brain's memory blueprint and
  returning only information that answers the question.
version: 1
model: anthropic/claude-sonnet-4-6

# TOOL: not run manually — exposed via tools/execution/qa/ask-question.yml and
# invoked as its own workflow-run. Parameters arrive as {{input.query}}.
type: TOOL

system-prompt-code: WF-SYSTEM-PROMPT

output-tokens: 2048, 4096
caching: automatic
max-turns: 8

# Blueprint tools only — this workflow must ground answers in memory.
tools:
  - search_blueprint_entries
  - get_blueprint_entry
  - list_blueprint_entries
---

# Instructions

You are answering a single question supplied by the calling tool as
`{{input.query}}`.

**Question**

{{input.query}}

1. Read the question carefully and identify exactly what is being asked.
2. Search the memory blueprint with `search_blueprint_entries` using a clear
   query derived from the question. If results are thin, try
   `list_blueprint_entries` and open promising titles with `get_blueprint_entry`.
3. For each relevant hit, call `get_blueprint_entry` to read the full content
   before relying on it.
4. Answer using **only** information found in those blueprint entries. Do not
   invent facts, pad with general knowledge, or speculate beyond what the
   entries support.
5. If the blueprint does not contain enough information to answer, say so
   plainly and state what is missing — do not guess.
6. Return only the answer — no preamble, no restatement of the question, no
   tool commentary.
