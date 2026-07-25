---
name: Create Frame of Reference
code: WF-FRAME-OF-REFERENCE
description: >-
  Builds a grounded frame of reference for a given context using the brain
  glossary and blueprint memory — problem statement, frame, domain model,
  bigger picture, and considerations.
version: 1
model: anthropic/claude-sonnet-4-6

# TOOL: callable as a pre-step by other workflows (e.g. via run_workflow or the
# create_frame_of_reference workflow-tool). Parameters arrive as {{input.*}}.
type: TOOL

system-prompt-code: WF-SYSTEM-PROMPT

output-tokens: 2048, 4096
caching: automatic
max-turns: 12

# Glossary is built from {{#blueprint.entries}} (title + version/centrality).
# Retrieval uses blueprint search — ask_question is a separate workflow-tool.
tools:
  - search_blueprint_entries
  - get_blueprint_entry
  - list_blueprint_entries
---

# Instructions

You are building a **frame of reference** for the context below. A frame of
reference is a structured foundation of shared understanding — context,
terminology, mental models, and operational assumptions — so later work reasons
from the same conceptual map.

The original problem may use the **wrong language**. For example, "the task
modal won't go away" might actually be a tooltip displayed incorrectly. Map the
input to real concepts in this brain's glossary and memory, or explicitly call
out ambiguity when the mapping is unclear.

## Context

{{input.context}}

## Glossary

Blueprint entry titles and centrality (`version`) in the current scope.
Higher version ≈ updated more often — a relevance heuristic, not a guarantee
of importance. If the list is empty, proceed without it — do not error.

<glossary>
{{#blueprint.entries}}
{{entry.title}}, {{entry.version}}
{{/blueprint.entries}}
</glossary>

Prefer `search_blueprint_entries` / `list_blueprint_entries` /
`get_blueprint_entry` to ground yourself when needed.

## Process

1. Read the context and the glossary. Identify which glossary titles (if any)
   are most relevant. Use centrality as a tiebreaker when titles look equally
   related. Cap deep retrieval at **3–5** targeted tool calls — do not load
   every high-centrality entry.
2. For selected titles or topics, call `search_blueprint_entries` with a focused
   query derived from the context. Open promising hits with
   `get_blueprint_entry` before relying on them. You may also
   `list_blueprint_entries` when search is thin.
3. Using the context, glossary, and retrieved memory, produce **exactly** these
   five sections and nothing else:

### 1. Problem statement
Summarise the information provided as a problem statement — as concise as
possible.

### 2. Frame
Analyse the problem to identify its contextual lens: the environment it
operates in, the processes it is embedded in, dependencies on other elements,
and constraints that govern its behaviour. Output **only a single contextual
lens statement** that synthesises these findings.

### 3. Domain model
Describe the domain model this problem exists within, using concepts from the
glossary and retrieved blueprint entries (and other known details from memory).

### 4. Bigger picture
If the frame and model describe an inner frame, state the outer frame it sits
inside — as concise as possible. If there is no meaningful outer frame, say so
briefly.

### 5. Considerations
A single list of peripheral things that may be important to consider.

**Test your frame:** it should enable clearer reasoning about the context. If
concepts are ambiguous or the input language does not match the glossary, say
so under Considerations (or in the Frame when the lens itself is unclear).

Return **only** the five sections above — no preamble, no tool commentary, no
extra closing remarks.
