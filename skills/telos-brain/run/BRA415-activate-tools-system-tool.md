---
name: Activate Tools System Tool
code: BRA415
version: 1
description: The activate_tools system tool. After find_available_tools returns
  names, pass the ones you intend to call. They are promoted for the rest of
  the run, and onto the MCP server tool list for the rest of the session when
  the run belongs to an MCP workflow or was started from one.
tools:
  - activate_tools
---

# Activate Tools System Tool

`find_available_tools` returns names and descriptions. That text does not
make a tool callable. The provider only calls tools that are on the next
request. `activate_tools` is how you put a discovered tool there.

`get_skill` still promotes tools listed on the loaded skill when they are
also in the workflow's `available-tools`. This tool is the extra step for a
name no loaded skill declared.

This is an ordinary `system` tool (BRA214). Declaration lives under
`tools/system-tools/activate-tools.yml`. This skill lists it in frontmatter
`tools:` so a workflow that keeps it under `available-tools` can promote it
via `get_skill`. Workflows that inject `find_available_tools` should inject
`activate_tools` beside it. If it lived only in `available-tools`, the model
could not call it until something else had already promoted it.

**Scope:** always the current run (`WorkflowRunId` harness-injected). Never
pass a brain id or a workflow id.

---

## `activate_tools`

| | |
| --- | --- |
| **Purpose** | Promote named tools for the rest of this run |
| **When** | After `find_available_tools`, before calling a tool that is not already injected |
| **YAML** | `tools/system-tools/activate-tools.yml` |

### Parameters

| Name | Required | Description |
| --- | --- | --- |
| `tools` | yes | Comma-separated tool names, for example `xero_get_invoices, xero_get_invoice` |

### What is promoted

A name is activated only when it is in **this** workflow's `available-tools`
pool and it is not a system tool. The workflow still owns the permission
envelope.

| Outcome | Behaviour |
| --- | --- |
| Name in `available-tools` | Appended to this run's promoted tools. Call it on the **next** step. The definition (name, description, parameters) arrives on that request. |
| Name already in `tools:` (injected) | Reported as already callable. Not an error. |
| Unknown, or not in this workflow's available tools | Reported as not activated. Not written. |

A second call appends. It does not replace names already promoted, including
names `get_skill` promoted.

### MCP sessions

When the run's workflow is `type: MCP`, or the run was started from an MCP
workflow (including a nested job such as `oneoff_job`), the same names are
published on that MCP server's tool list. They stay there until the
workflow's `session-timeout` passes, and the window refreshes while the
session is used. They are not written into the workflow definition. When the
session expires, `tools/list` returns to the injected tools only, and the
session sweep deletes the stored row.

A nested job may publish a tool onto the parent MCP even when that tool is
not on the MCP workflow's own `available-tools`. The available-pool check is
the check on the workflow that called `activate_tools` or `get_skill`.

---

## Wiring it into a workflow

1. Include the `tools/system-tools/` group in `brain-compose.yml`.
2. Inject `activate_tools` under workflow `tools:`, next to
   `find_available_tools`.

---

## See also

- **BRA212** — keep domain tools in `available-tools` and activate the ones a turn needs
- **BRA215** — `get_skill` promotion from a skill's `tools:` list
- **BRA217** — `tools` and `available-tools` on a workflow
