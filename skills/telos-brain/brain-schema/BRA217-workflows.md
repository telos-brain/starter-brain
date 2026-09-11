---
name: Workflows
code: BRA217
version: 1
description: How to author workflow markdown — frontmatter, types, triggers,
  tools / available-tools, input-tools, LLM execution settings, session
  timeout, and external agent deployment. Load this when creating or editing a
  workflow.
---

# Workflows

A workflow is a **single, self-contained markdown file**. The frontmatter is the
header and the wiring (tools + skills); the markdown body is the instructions.
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
type: RUNNABLE                         # optional; one of TOOL | RUNNABLE | TRIGGERED | SYSTEM | SIMULATION | COMPACTION (default RUNNABLE)
# trigger: inbox:SKILL_UPDATE           # optional; TRIGGERED only — scalar or YAML list
# trigger: inbox:SKILL_UPDATE:low      # optional learning-mode qualifier: low|medium|high
# trigger: inbox:SKILL_UPDATE:high:10  # optional weight threshold (positive integer)
# trigger: inbox:*:high:10:SKILL_UPDATE  # optional routing-code filter after weight
# trigger: workflowrun:complete         # run-eval: any subject workflow
# trigger: workflowrun:complete:high    # run-eval: any workflow, only when learning mode is high
# trigger: workflowrun:complete:WF-REVIEW:high  # run-eval: one subject workflow + mode
# trigger:
#   - inbox:SKILL_UPDATE
#   - inbox:WORKFLOW_UPDATE:medium
# model: anthropic/claude-sonnet-4-6   # optional; provider/model (see BRA210)
# model: openai/gpt-4o                 # OpenAI
# model: xai/grok-4.5                  # xAI / Grok
# deployment-type: elevenlabs_conversational_ai  # optional; project this workflow as an external agent
# elevenlabs-agent-id: agt_xxx         # optional; written back after first ElevenLabs create — omit on first deploy

# Injected tools — included in the LLM tool declarations for every turn.
# Referenced by tool NAME (the tool's `name`).
tools:
  - find_available_skills
  - get_skill
  - find_available_tools
  - add_ticket_comment

# Available tools — in the workflow's searchable permission envelope only.
# Surfaced via find_available_tools, or promoted mid-run when get_skill loads a
# skill that lists them under its own tools: field (see **BRA215**). Not injected by
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
- `type` (case-insensitive) must be one of `TOOL`, `RUNNABLE`, `TRIGGERED`, `SYSTEM`, `SIMULATION`, `COMPACTION`;
  omitted defaults to `RUNNABLE`. `TOOL` = callable by another workflow (e.g. exposed via a `workflow` tool), `RUNNABLE` = executed manually, `TRIGGERED` = fired when `trigger` matches, `SYSTEM` = never invoked directly; referenced by other workflows via `system-prompt-code` to supply the system prompt. `SIMULATION` = tool-response synthesis for simulation interception; the active workflow of this type (not a specific code) handles intercepted API/MCP tools on a simulation run. `COMPACTION` = context summariser for auto-compaction and the `compact_context` system tool; at most one active per brain.
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
- For `workflowrun:complete`, `trigger-mode: automatic` enqueues on Completed
  when the optional subject-workflow filter and learning-mode qualifier match;
  `trigger-mode: manual` (or omitted) is kicked off from the admin **Run eval**
  button (workflow filter applies; learning-mode qualifier is ignored — operator
  override). A third segment that is `low|medium|high` is the learning-mode
  qualifier with an implicit `*` workflow filter (`workflowrun:complete:high`);
  any other third segment is a workflow code (`workflowrun:complete:WF-REVIEW`).
  Add a fourth segment for both (`workflowrun:complete:WF-REVIEW:high`). Use a
  YAML list for several subject workflows (OR). Use `{{run.telemetry}}` (BRA204)
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

Set `model` to a `provider/model-name` string (e.g. `anthropic/claude-sonnet-4-6`,
`openai/gpt-4o`, `xai/grok-4.5`). Supported providers, example model codes, and
credential mapping are listed in **BRA210**. Bare model names (no prefix)
default to Anthropic. Omit `model` to use the brain default (`llm-model` /
`DEFAULT_LLM_MODEL` / Settings). If that is also unset, the run fails — leftover
cloud keys are not used as a silent default.

## 3. LLM execution settings (optional)

A workflow may declare fine-grained control over how the conversant runs it.
All fields below are **optional** and **kebab-case**; omit any of them to keep
its default. Omitting all of them reproduces the historic behaviour exactly, so
existing workflows need no changes.

`max-turns` and `output-tokens` apply to every supported provider. `caching`
applies on providers that support prompt caching (Anthropic, xAI); OpenAI
ignores it. `thinking`, `thinking-budget`, and `thinking-effort` are
**Claude-oriented** — validated on deploy when present, but ignored at run time
on OpenAI / xAI (see **BRA210** §5). `auto-compaction` applies on every
provider: Claude uses server-side `compact_20260112`; OpenAI / xAI run the
brain's `COMPACTION` workflow client-side when the prompt-token threshold is
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
  The `effort` mode defaults to `low`; `adaptive` omits it so Anthropic's API
  default (`high`) applies. Prefer `adaptive` + `thinking-effort` over `extended` on
  newer models, where a manual `budget_tokens` is rejected.
- `caching` / `thinking` / `thinking-budget` / `thinking-effort` are validated on
  deploy; an invalid value is a hard error.
- `max-recursion-depth` caps how deep workflows may nest via `run_workflow` or
  workflow-typed tools. Depth starts at `1` for a top-level run and increments by
  `1` for each nested invocation. An invocation that would exceed the cap is
  refused before a `WorkflowRun` is created.
- `max-runs-per-hour` caps how many times this workflow may execute in a rolling
  one-hour window. High-frequency system workflows (for example `WF-EVAL`) should
  set an elevated value so they are not throttled under load. Omit the field to
  use the default of `50`.

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
