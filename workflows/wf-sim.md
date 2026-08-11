---
name: Simulate Tool Response
code: WF-SIM
version: 1
type: SIMULATION
description: Synthesises a plausible tool response from historical sample call/response pairs for simulation runs.
model: anthropic/claude-sonnet-4-6

# No tools: / available-tools: — the LLM only synthesises from pre-fetched samples.
# get_sample_tool_results is a system tool; pre-called here so samples arrive as
# context before the first turn (ToolRouter passes tool_name / parameters / count).
input-tools:
  - variable: sample_tool_results
    tool: get_sample_tool_results
    parameters:
      tool_name: "{{input.tool_name}}"
      parameters: "{{input.parameters}}"
      count: "{{input.count}}"

output-tokens: 4096
caching: automatic
max-turns: 3
---

# Instructions

You are synthesising a **simulated** tool response for a tool call that is being
intercepted during a simulation run. Live tools must not be invoked.

## Intercepted call

- **Tool name:** `{{input.tool_name}}`
- **Parameters:** `{{input.parameters}}`

## Historical samples

Look for the `<pre_called_tool name="sample_tool_results">` block in context.
Inside it, each historical pair is wrapped as:

```xml
<tool_call>
  <parameters>{ ... }</parameters>
  <response>{ ... }</response>
</tool_call>
```

If that block is missing, empty, or reports an error, invent a minimal plausible
response consistent with the tool name and parameters alone.

## Your task

1. Analyse the example `<parameters>` / `<response>` pairs (prefer exact-parameter
   matches when present; otherwise generalise from the most recent examples).
2. Synthesise **one** plausible response for the intercepted call above.
3. Return **only** the synthesised response string — no XML wrapper, no
   commentary, no preamble, and no explanation of how you derived it.
