---
name: Workflows
code: BRA217
version: 9
description: How to author workflow markdown — frontmatter, types, triggers,
  tools / available-tools, input-tools, LLM execution settings, session
  timeout, and external agent deployment. Load this when creating or editing a
  workflow.
---

# Workflows

A workflow is a **single, self-contained markdown file**. The frontmatter is the
header and the wiring (tools + skills); the markdown body is the instructions.
The conventional path is `workflows/{code}.md`; grouping folders are allowed
(`workflows/jobs/{code}.md`) and `create_schema_file` creates them on the fly.
Overview and versioning: **BRA201**. Models: **BRA210**. Inbox trigger runtime:
**BRA404**. Learning-eval authoring: **BRA207**. Skill-declared tool promotion:
**BRA215**. Template tags: **BRA204**. Run variables / input-tools how-to:
**BRA409**.

```markdown
---
name: Review Blueprint                 # REQUIRED (workflow title)
code: WF-REVIEW                        # REQUIRED (unique workflow code)
description: Reviews a blueprint submission and posts findings to the ticket.  # optional
version: 1.1                           # optional (see **BRA201** versioning)
type: RUNNABLE                         # optional; one of TOOL | RUNNABLE | TRIGGERED | EVAL | SYSTEM | SIMULATION | COMPACTION | MCP (default RUNNABLE)
# trigger: inbox:SKILL_UPDATE           # optional; TRIGGERED only — scalar or YAML list
# trigger: inbox:SKILL_UPDATE:low      # optional learning-mode qualifier: low|medium|high
# trigger: inbox:SKILL_UPDATE:high:10  # optional weight threshold (positive integer)
# trigger: inbox:*:high:10:SKILL_UPDATE  # optional routing-code filter after weight
# trigger: workflowrun:complete:high    # EVAL only — automatic, any subject workflow at high
# trigger: workflowrun:complete:WF-REVIEW:high  # EVAL only — automatic, one subject workflow + mode
# trigger:
#   - inbox:SKILL_UPDATE
#   - inbox:WORKFLOW_UPDATE:medium
# model: anthropic/claude-sonnet-4-6   # optional; provider/model (see BRA210)
# failover-workflow: WF-BACKUP         # optional; other workflow code in this brain (BRA455)
# model: openai/gpt-4o                 # OpenAI
# model: telosbrain/xai/grok-4.6       # Telos-hosted Grok (brain credit, 2× xAI list)
# model: xai/grok-4.5                  # xAI / Grok (your key)
# deployment-type: elevenlabs_conversational_ai  # optional; project this workflow as an external agent
# elevenlabs-agent-id: agt_xxx         # optional; written back after first ElevenLabs create — omit on first deploy

# Injected tools — included in the LLM tool declarations for every turn.
# Referenced by tool NAME (the tool's `name`).
tools:
  - find_available_skills
  - get_skill
  - find_available_tools
  - activate_tools
  - add_ticket_comment

# Available tools — in the workflow's searchable permission envelope only.
# Surfaced via find_available_tools. Promoted for this run when activate_tools
# is called with their names, or when get_skill loads a skill that lists them
# under its own tools: field (see **BRA215** and **BRA415**). Not injected by
# default. Omit the key entirely when unused (existing workflows stay valid).
available-tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
  - create_skill
  - create_schema_file

# Pre-called tools — executed automatically at run startup before the
# AI's first turn. Parameter values support Template Service tags such as
# {{input.*}} (from the Execution API `variables` map). Omit entirely when unused.
# input-tools:
#   - variable: widget_information
#     tool: get_widget_details
#     parameters:
#       widget_reference: "{{input.widget_reference}}"

# Skills referenced by CODE.
#  - injected-skills are inlined into the prompt at run time.
#  - available-skills can be loaded on demand at runtime via skill tools.
injected-skills:
  - EP101
available-skills:
  - OP201
---

# Instructions

1. Load the ticket details for the referenced blueprint submission.
2. Assess the blueprint against the embedded skill guidance.
3. Summarise the findings clearly, calling out any issues or risks.
4. Post the summary back to the ticket as a comment using `add_ticket_comment`.
```

Rules:

- `name` and `code` are required; the markdown **body must not be empty** (it's
  the instructions).
- `type` (case-insensitive) must be one of `TOOL`, `RUNNABLE`, `TRIGGERED`, `EVAL`, `SYSTEM`, `SIMULATION`, `COMPACTION`, `MCP`;
  omitted defaults to `RUNNABLE`. `TOOL` = callable by another workflow (e.g. exposed via a `workflow` tool), `RUNNABLE` = executed manually, `TRIGGERED` = fired when an inbox / unit-of-work `trigger` matches, `EVAL` = workflow-run learning eval (the **Run eval** button; `trigger` is automatic-only — **BRA207**), `SYSTEM` = never invoked directly; referenced by other workflows via `system-prompt-code` to supply the system prompt. `SIMULATION` = tool-response synthesis for simulation interception; the active workflow of this type (not a specific code) handles intercepted API/MCP tools on a simulation run. `COMPACTION` = context summariser for auto-compaction and the `compact_context` system tool; at most one active per brain. `MCP` = publishable Brain MCP endpoint.
- Well-known `trigger` values include `inbox:<RoutingType>` / `inbox:*` (inbox
  learning loop), `unitofwork:complete` (unit-of-work learning eval), and
  `workflowrun:complete[:<workflow-code>|*][:<learning-mode>]` (workflow-run
  learning eval, **BRA207**).
- **Inbox triggers** (two stages — full rules in **BRA404**):
  - **Entry create:** `inbox:<RoutingType>` or `inbox:*` (optional
    `:low|medium|high` learning-mode qualifier, optional `:<weight-threshold>`
    positive integer, optional `:<routing-code>`) selects which `TRIGGERED`
    workflows get a `PENDING` inbox task when an entry is created. Qualifiers
    use `off < low < medium < high` (brain mode must meet or exceed the
    qualifier; unqualified inbox triggers always fire). A weight threshold
    fires only when the entry's `Weight` meets or exceeds that value; absent
    defaults to no weight gate. Weight is evaluated at entry creation only —
    clustering that later raises a source entry's weight does not re-fire that
    entry's triggers. A fifth-segment routing code (only valid after a weight
    threshold) matches that `RoutingType` exactly; omit it or use `*` to match
    all routing types.
  - **Task auto-run:** once a task exists, the **workflow linked on that task**
    is authoritative. If that workflow has an inbox trigger whose learning-mode
    qualifier, optional weight threshold, and optional fifth-segment routing
    code are satisfied (weight / routing are the parent entry's **current**
    values), the task auto-runs (`PENDING → RUNNING`); otherwise it moves to
    `AWAITING_APPROVAL`. The second-segment routing pattern is not re-checked
    at this stage. `trigger-mode` does **not** control inbox task approval.
    Apply-learning workflows that should only auto-run on a well-evidenced
    signal use `inbox:*:high:10` (canonical: `WF-INBOX-ENTRY-CONTEXT`); add
    `:<RoutingType>` when the workflow should handle only one routing code.
- For `type: EVAL`, `trigger` is **automatic only**. A `low|medium|high`
  qualifier enables automatic enqueue (`workflowrun:complete:high`, or
  `workflowrun:complete:WF-REVIEW:high`). Unqualified patterns and an omitted
  trigger never auto-run. The admin **Run eval** button is type-based — it
  shows on every evaluable run when an `EVAL` workflow exists, and ignores
  the trigger. `trigger-mode` is ignored for eval. A third segment that is
  `low|medium|high` is the learning-mode qualifier with an implicit `*`
  workflow filter; any other third segment is a workflow code. Use a YAML
  list for several automatic patterns (OR). Use `{{run.telemetry}}` (BRA204)
  for OTEL GenAI telemetry of the subject run. Full authoring guide:
  **BRA207**. Canonical example: `workflows/WF-EVAL-RUN.md`.
- `tools` (injected) and `available-tools` (searchable / promotable) reference
  tools by their `name`. `injected-skills` / `available-skills` reference skills
  by their `code`. These lists accept a single string or a list. Make sure the
  referenced names/codes exist.
- A workflow with only `tools` and no `available-tools` is unchanged — every
  listed tool is treated as injected.
- On deploy, `tools` become the workflow's injected tools and `available-tools`
  become the searchable pool. Extracting the schema writes those two lists back
  out.
- `input-tools` (optional) declares tools to call automatically at run
  startup. See **Pre-called tools** below.

## 1. Pre-called tools (`input-tools`)

Use `input-tools` when a workflow needs structured context fetched before the
model's first turn — for example loading a record by a reference passed in the
Execution API `variables` map. Full how-to (API + schema + worked example
`WF-INPUT-VARIABLES`): **BRA409**. Run-body field: **BRA403**.

```yaml
input-tools:
  - variable: widget_information
    tool: get_widget_details
    parameters:
      widget_reference: "{{input.widget_reference}}"
  - variable: entity_summary
    tool: get_entity_details
    parameters:
      entity_id: "{{input.entity_id}}"
```

| Field | Required | Meaning |
| ----- | -------- | ------- |
| `variable` | yes | Name under which the result is injected |
| `tool` | yes | Tool name (declared tool or system tool) |
| `parameters` | no | String map of argument values; supports `{{…}}` tags |

**Runtime behaviour**

1. After run `variables` are available as `{{input.*}}`, each entry runs in
   declaration order.
2. Parameter values are rendered as templates, then the named tool is called.
3. Successful results are injected as a user-role context block:

   ```xml
   <pre_called_tool name="widget_information">…tool result…</pre_called_tool>
   ```

4. On failure (tool missing, dispatch error, or exception) the run **continues**.
   A warning block is injected instead:

   ```xml
   <pre_called_tool name="widget_information" status="error">Tool call failed: …</pre_called_tool>
   ```

5. Results are capped at 200,000 characters (same limit as LLM-loop tool
   results). Extract omits the `input-tools` key entirely when there are no
   rows — never writes `input-tools: []`.

**Worked example — API variables → pre-called tool**

Caller:

```json
POST /workflows/WF-WIDGET/run/sync
{
  "variables": { "widget_reference": "WID-001" }
}
```

Workflow frontmatter:

```yaml
input-tools:
  - variable: widget_information
    tool: get_widget_details
    parameters:
      widget_reference: "{{input.widget_reference}}"
```

The model sees `<pre_called_tool name="widget_information">…</pre_called_tool>`
before its first turn, with no extra LLM tool call required to fetch the widget.

## 2. Choosing a model (`model`, see **BRA210**)

Set `model` to one `provider/model-name`, or an ordered comma-separated list
of interchangeable models (e.g.
`local_1/Qwen/Qwen3.6-35B-A3B, openrouter/qwen/qwen3.6-35b-a3b`).
A single id is a list of one. Providers, codes, credentials, and stepping
are in **BRA210**. Bare names (no prefix) default to Anthropic.
When `model:` is set, that list is the whole run — a missing credential
skips that entry; the brain default is not appended.
Omit `model` to use the brain default list (`llm-model` /
`DEFAULT_LLM_MODEL` / Settings). If that is also unset, the run fails —
leftover cloud keys are not a silent default.

## 2a. Fail over to another workflow (`failover-workflow`)

`failover-workflow` names another workflow **code** in the same brain. It is not a model, and it is not a second run.

Use it when the backup work needs a different prompt, tools, or instructions. Interchangeable models belong on `model:` as a comma-separated list (**BRA210**). A single model is a list of one.

The rewrite runs only after that list is exhausted on the **first** LLM attempt, and only when no assistant turn with tokens has been stored. The same `runId` is kept. The original input message, variables, entity, unit of work, caller JWT, initiation method, and callback URL stay. Messages from the failed attempt are deleted. The failover workflow then starts from scratch, including its own `input-tools`. Side effects those original input-tools already had are not rolled back.

One hop. If the failover workflow's list is also exhausted, the run fails. A missing code, or a code that names this same workflow, fails the run. There is no fall-through to `DEFAULT_LLM_MODEL` or a leftover provider key.

Omit the key when unused. Extract never writes `null`.

## 3. LLM execution settings (optional)

A workflow may declare fine-grained control over how the conversant runs it.
All fields below are **optional** and **kebab-case**; omit any of them to keep
its default. Omitted `thinking` does not send a reasoning field, so Grok keeps
its API default of `high`.

`max-turns` and `output-tokens` apply to every supported provider. `caching`
applies on providers that support prompt caching (Anthropic, xAI); OpenAI
ignores it. `thinking` and `thinking-effort` apply on every provider (see
**BRA210** §6). `thinking-budget` applies on Anthropic and on OpenRouter
`extended`; other providers have no thinking token budget. All three are
validated on deploy. `temperature` applies on every provider: omit it to keep
that provider's own default (the field is not sent). `top-p`, `top-k`, `min-p`,
and `presence-penalty` are the same kind of optional sampling control: omit a
field to leave it off the request. A set value is sent only to providers whose
API has that parameter (**BRA210** §6). `auto-compaction` applies
on every provider: Claude uses server-side `compact_20260112`; OpenAI / xAI run
the brain's `COMPACTION` workflow client-side when the prompt-token threshold is
reached.

```markdown
---
name: Chat
code: WF-CHAT
type: RUNNABLE
model: anthropic/claude-sonnet-4-6

auto-compaction: 100000        # off (default) | input-token trigger (Claude server-side; OpenAI/xAI client-side via COMPACTION workflow)
output-tokens: 2048, 4096, 16384  # 4096 (default) | ordered per-attempt output caps (Claude max_tokens); each value is the next retry when a turn stops at the cap
caching: automatic             # (default: hand-crafted per-block markers) | none (suppress all) | automatic (provider automatic prompt cache)
max-turns: 50                  # 10 (default) | tool-use loop cap in turns
thinking: adaptive             # none (default) | adaptive (model decides) | extended (manual budget) | effort (adaptive + explicit effort)
thinking-budget: 24000         # per-mode default | thinking token budget (extended budget_tokens, or adaptive/effort headroom over the output cap)
thinking-effort: low           # low (default for effort mode) | medium | high | xhigh | max | adaptive-thinking effort (cost lever)
temperature: 0.2               # omit to keep the provider default | number from 0 to 2 inclusive
top-p: 0.9                     # omit | number from 0 to 1 inclusive (sent on every provider)
top-k: 40                      # omit | whole number from 0 to 10000 (Claude, OpenRouter, local)
min-p: 0.05                    # omit | number from 0 to 1 inclusive (OpenRouter, local)
presence-penalty: -0.5         # omit | number from -2 to 2 inclusive (OpenAI, Azure, OpenRouter, local)
max-recursion-depth: 5         # 5 (default) | max nesting depth for recursive workflow invocations
max-runs-per-hour: 50          # 50 (default) | max executions of this workflow per rolling hour
---
```

| Field | Default when omitted | Accepted values |
| ----- | -------------------- | --------------- |
| `auto-compaction` | off (no compaction) | a positive integer (input-token trigger; Claude API enforces a 50000 minimum; OpenAI/xAI use the same threshold client-side) |
| `output-tokens` | `4096` (a single attempt) | a positive integer, or an ordered comma-separated list of positive integers (e.g. `2048, 4096, 16384`) |
| `caching` | hand-crafted per-block markers | `none` \| `automatic` |
| `max-turns` | `10` | a positive integer |
| `thinking` | `none` | `none` \| `adaptive` \| `extended` \| `effort` |
| `thinking-budget` | per-mode default (extended `10000`; adaptive/effort `0`, i.e. no headroom) | a positive integer of at least `1024` (tokens) |
| `thinking-effort` | `low` for `effort` mode; `adaptive` omits it (API default `high`) | `low` \| `medium` \| `high` \| `xhigh` \| `max` |
| `temperature` | omit the field; each provider keeps its own default | a number from `0` to `2` inclusive (for example `0.2` or `1`) |
| `top-p` | omit the field | a number from `0` to `1` inclusive, at most three decimal places |
| `top-k` | omit the field | a whole number from `0` to `10000` inclusive |
| `min-p` | omit the field | a number from `0` to `1` inclusive, at most three decimal places |
| `presence-penalty` | omit the field | a number from `-2` to `2` inclusive, at most two decimal places |
| `max-recursion-depth` | `5` | a positive integer |
| `max-runs-per-hour` | `50` | a positive integer |

Notes:

- `output-tokens` is an ordered list of per-attempt output caps: the first value
  is the initial attempt and each subsequent value is used for the next retry, so
  the number of values is the number of attempts (a single value means no
  retries). A retry fires only when a turn stops with Claude's
  `stop_reason: "max_tokens"` and another cap remains. Every attempt is recorded
  as its own assistant turn in run telemetry (each with its `stop_reason`, the
  output cap it ran with, and its consumed tokens), so a three-cap list that keeps
  truncating leaves two truncated attempt rows before the final one. Failed
  attempts' tokens count towards the run totals because they were genuinely
  billed.
- `thinking: extended` uses a manual thinking budget — prefer `adaptive` or
  `effort` on newer models where a manual budget is not accepted.
- `thinking-budget` only applies when a thinking mode is on and is otherwise
  ignored. For `extended` it is the manual `budget_tokens` (default `10000`). For
  `adaptive` / `effort` it is headroom added on top of `output-tokens` and defaults
  to `0` — because max_tokens caps thinking + response combined, by default
  thinking shares the `output-tokens` budget with the reply; set `thinking-budget`
  to reserve extra room so a large amount of thinking cannot truncate the reply.
- `thinking-effort` is the real thinking-cost lever (Anthropic bills tokens
  actually generated, not `output-tokens`, which is only a ceiling). It applies to
  the `adaptive` / `effort` modes and is ignored otherwise. Lower effort thinks
  less — cheaper, faster, and prioritises the response; higher effort reasons more.
  The `effort` mode defaults to `low`; `adaptive` omits it so the mapped effort
  is `high`. Omitted `thinking` sends nothing, so Grok stays at its API default
  of `high` (**BRA210** §6). Prefer `adaptive` + `thinking-effort` over `extended`
  on newer models, where a manual `budget_tokens` is rejected. Only Anthropic and
  OpenRouter `extended` send `thinking-budget`. Elsewhere `extended` maps to
  effort `high`.
- `caching` / `thinking` / `thinking-budget` / `thinking-effort` / `temperature` /
  `top-p` / `top-k` / `min-p` / `presence-penalty` are validated on deploy; an
  invalid value is a hard error.
- `temperature` is sent on every model call for that workflow when set, and
  omitted entirely when unset. Do not rely on a platform default being written
  into the request. Nested `run_workflow` children use their own workflow and
  do not inherit the parent's temperature. Claude accepts `0`–`1` and may
  reject a higher value at the API. Claude thinking has historically required
  temperature to be omitted or set to `1`; combining another value with
  `thinking` may fail at Anthropic. The field is still sent.
- `top-p`, `top-k`, `min-p`, and `presence-penalty` follow the same omit-when-unset
  rule, and `0` is a real value that is sent. They are not inherited by nested
  `run_workflow` children. A set value is included on a provider request only
  when that API has the parameter; otherwise it is left off that call and still
  stored, shown, and reported in telemetry. See **BRA210** §6. Claude 4.7 and
  later may reject non-default `top_p` / `top_k` at the API; the fields are
  still sent. A local runner that rejects an unknown sampling field may fail
  the call.
- `max-recursion-depth` caps how deep workflows may nest via `run_workflow` or
  workflow-typed tools. Depth starts at `1` for a top-level run and increments by
  `1` for each nested invocation. An invocation that would exceed the cap is
  refused before a `WorkflowRun` is created.
- `max-runs-per-hour` caps how many times this workflow may execute in a rolling
  one-hour window. High-frequency system workflows (for example `WF-EVAL`) should
  set an elevated value so they are not throttled under load. Omit the field to
  use the default of `50`.

## 3a. Entity and unit-of-work scope (`entity`, `unit-of-work`)

Optional. Omitted settings keep today's behaviour: Execution API `entity_id` /
`unit_of_work_id` are stored as supplied, and an MCP `tools/call` stays unscoped.

When set, a top-level run reads the named parameter and writes the matched id
onto the run before the instructions are rendered. Execution API runs read
`variables`. MCP runs read the tool-call arguments (those arguments are not
copied into the run's variables). Nested `run_workflow` children and heartbeat
successors inherit the parent's ids and do not match again.

```yaml
entity: clients:slug:slug
unit-of-work: jobs:reference:reference
```

Each value is `type-code:parameter-name:match-key`. Two parts are allowed
(`clients:entityId`); the match key then defaults to `id`.

- `type-code` is the entity type or unit-of-work type code on this brain.
- `parameter-name` is the incoming key. The match is case-sensitive. A missing
  or blank value leaves that id unset.
- `id` matches `Entities.Id` or `UnitsOfWork.Id`. Any other match key, including
  `reference`, is an entity variable or unit-of-work variable key. The built-in
  8-character Reference column is not used. Names are not used.
- The first match wins (`CreatedAt`, then `Id`). A miss does not fail the run
  and does not fall back to a caller-supplied id.
- Unit of work is resolved only after an entity is in scope, and only among
  that entity's units of work.
- A malformed value fails deploy. An unknown type code does not: it matches
  nothing at run time.
- These keys do not read an inbox entry. An inbox task that should keep the
  entry's entity uses `scope` below, not `entity`.

## 3b. Inbox task scope (`scope`)

An inbox task run is brain-scoped unless its workflow opts in. Set this on the
one workflow that should see the entity blueprint. That is usually
`WF-REVIEW-BLUEPRINT`. Leave it off every other task on the same entry.

```yaml
scope: inherit-entry
```

The only accepted value is `inherit-entry`. Anything else fails deploy. Omit
the key and the task stays brain-scoped, which is what existing workflows do.

What the engine does:

1. `create_inbox_entry` stores the entity id and unit-of-work id already on the
   run that calls it. `entity_name` and `unit_of_work_name` stay labels. A name
   is never looked up, so two entities with the same name cannot be mixed up.
2. When the task runs, a workflow with `scope: inherit-entry` copies those ids
   onto the run. Blueprint tools then use the same tiers as any other scoped
   run. A unit-of-work blueprint for that type wins if one exists. Otherwise an
   entity blueprint for the entity's type. Otherwise the brain blueprints.
3. If the workflow opts in and the entry has no entity id, the task stays
   brain-scoped. That covers an older entry, and a creating run that was itself
   brain-scoped. The server logs a warning that names the entry and the task.
4. The run records where the scope came from. `inherited-entry` means the ids
   were copied. `none` means the task stayed brain-scoped. On the run page this
   shows as "Inherited from the inbox entry" or "Brain-scoped". Other runs leave
   the field empty.
5. Opting in does not write a blueprint entry. If that tier has more than one
   blueprint, `add_blueprint_entry` still needs `blueprint` (a code or a title).

Do not also set `entity` or `unit-of-work` on a workflow that uses
`scope: inherit-entry`. The inherited ids are the ones the run keeps.

Example. The brain has an entity type `projects`, a blueprint `projects` with
`scope: entity:projects`, and brain blueprints `company` and `memory`. The
review task is:

```markdown
---
name: Review blueprint
code: WF-REVIEW-BLUEPRINT
type: TRIGGERED
trigger: inbox:MEMORY_UPDATE
scope: inherit-entry
tools:
  - add_blueprint_entry
  - search_blueprint_entries
---
```

Complete a unit of work on a project. The eval run is already scoped to that
project, and `create_inbox_entry` stores the project id on the inbox entry.
The `WF-REVIEW-BLUEPRINT` task then sees the `projects` blueprint. A second
task workflow on the same entry, without `scope`, still sees only `company`
and `memory`.

If the entry shows `EntityName: Security` but has no entity id, the review task
still sees only `company` and `memory`. Check the warning, then re-run the eval
from a scoped run so the entry is created with an id. Renaming the entry does
not fix it.

## 4. Chat session settings (optional)

A workflow run started synchronously (`POST /workflows/{code}/run/sync`, see
BRA403) is left open as a **chat session** the caller can continue turn by turn.
`session-timeout` controls how long an open session may sit idle before it is
automatically closed. It is **optional** and **kebab-case**.

```markdown
---
name: Chat
code: WF-CHAT
type: RUNNABLE

session-timeout: 15            # 30 (default) | minutes an open chat session may idle before it auto-closes
---
```

| Field | Default when omitted | Accepted values |
| ----- | -------------------- | --------------- |
| `session-timeout` | `30` (minutes) | a positive integer (minutes) |

Notes:

- The timeout is measured from the end of the most recent turn and re-armed on
  every continuation, so an actively used session never times out; only an
  abandoned one is swept closed.
- Closing a session (by timeout or an explicit `complete`) transitions the run to
  `Completed`, at which point it becomes eligible for evaluation. An open session
  is not evaluated.

## 5. External agent deployment (optional)

A workflow may declare itself as a **deployable external agent**. These fields
are **optional** and **kebab-case**. Omit both keys for a normal Brain-only
workflow. Full connector authoring: **BRA209**.

```markdown
---
name: Voice agent
code: WF-VOICE
type: RUNNABLE
deployment-type: elevenlabs_conversational_ai
# elevenlabs-agent-id: agt_xxx   # omit on first deploy; extract after create
---
```

| Field | Default when omitted | Accepted values |
| ----- | -------------------- | --------------- |
| `deployment-type` | none (Brain-only) | free text; first convention value is `elevenlabs_conversational_ai` |
| `elevenlabs-agent-id` | none | the `agent_id` returned by ElevenLabs after first create |

Notes:

- `deployment-type` is a discriminator, not a check-constrained enum. Presence
  of `elevenlabs_conversational_ai` is what the ElevenLabs deployment handler
  matches on.
- The brain must have **one** connector with `type: elevenlabs` (BRA209). The
  handler reads the `api-key` parameter's `secret:` when set, otherwise
  `CONNECTOR_{connectorId}_CLIENT_SECRET`, as the `xi-api-key`.
- On first deploy, omit `elevenlabs-agent-id`. The handler `POST`s
  `/v1/convai/agents/create` and writes the returned id back on the workflow.
  Run `brain extract` afterwards so the id is in the YAML; later deploys
  `PATCH` the existing agent.
- A later YAML deploy that omits `elevenlabs-agent-id` will clear the stored
  id (absent optional fields persist as null).
- Injected skill codes are resolved to full markdown at deploy time and
  appended to `workflow.Instructions` — ElevenLabs has no Telos skill codes.
- Omit both keys entirely when unused — never write `null` or empty string.

---
