---
name: "Execution API: Workflow Execution & Telemetry"
code: BRA403
version: 24
description: How to list a brain's workflows (with pending inbox-task counts),
  run them synchronously (SSE streaming) or asynchronously (fire-and-forget
  with callback), pass optional run variables for {{input.*}} template tags
  and pre-called input-tools (BRA409), hold a multi-turn chat session against
  a single run, stop an in-flight turn without closing the session
  (POST /runs/{id}/stop), and retrieve run telemetry and unit-of-work telemetry
  via the Execution API — including model, thinking mode, temperature, sampling
  parameters and turn totals on run
  telemetry. Documents when a run settles Failed (max-turns exhausted, final
  max_tokens with no retries). Also notes when a Completed, Failed, or
  AwaitingInput run becomes eligible for learning evals (BRA207). Documents
  SSRF rules for async callbackUrl (allowed-callback-domains).
---

# Execution API: Workflow Execution & Telemetry

See BRA401 for authentication and conventions. See BRA402 for entity and unit-of-work setup.

---

## List workflows

### `GET /workflows`

Returns every workflow in the brain, ordered by name, with a count of linked inbox tasks that are still pending (status is not `COMPLETED` or `CANCELLED`).

Response `200 OK`:

```json
[
  {
    "name": "Chat",
    "code": "WF-CHAT",
    "version": 2,
    "description": "Open chat session",
    "pendingTasks": 3
  }
]
```

| Field | Notes |
|---|---|
| `name` | Workflow display title |
| `code` | Stable deploy key used to run the workflow |
| `version` | Integer schema version |
| `description` | Optional |
| `pendingTasks` | Linked inbox tasks not `COMPLETED` or `CANCELLED` (`PENDING`, `AWAITING_APPROVAL`, `RUNNING`, `FAILED`) |

---

## Workflow Execution

Workflows are addressed by **code** and resolved to the active workflow within the brain's scope. Both run endpoints share the same execution engine and the same request body.

### Shared request body

All fields are optional:

```json
{
  "inputMessage": "Draft a proposal",
  "entityId": "3f0c...",
  "unitOfWorkId": "7b2d...",
  "variables": {
    "widget_reference": "WID-001",
    "current_date": "2026-07-30"
  },
  "callbackUrl": "https://app.example.com/brain/callbacks/run-complete",
  "caller_jwt": "eyJhbGc...",
  "resumeRunId": "44291...",
  "resumeMethod": "exclude_tools"
}
```

| Field | Notes |
|---|---|
| `inputMessage` | The triggering user message. |
| `entityId` | Optional runtime scope. |
| `unitOfWorkId` | Optional runtime scope for template tags (e.g. `{{#unitOfWork.context}}`). Nothing is hard-coded into the prompt; workflows that need the logs declare those tags in Instructions. |
| `variables` | Optional string→string map persisted on the `WorkflowRun` and exposed to templates as `{{input.<key>}}`. Omit or leave null for unchanged behaviour. Keys and values are plain strings (no type coercion). See [Run variables](#run-variables-input) and **BRA409**. |
| `callbackUrl` | Honoured on the async path only. Must be `https` (or `http://localhost` / `http://127.0.0.1` in Development). Subject to the brain's shared outbound host allowlist — see [Async callback SSRF rules](#async-callback-ssrf-rules). |
| `caller_jwt` | Optional opaque JWT forwarded as `Authorization: Bearer` on outbound calls for connectors with `auth-type: caller-jwt`. Never logged or echoed. Nested `run_workflow` / workflow-tool child runs inherit it. The brain does not validate the token. Clerk JWTs often expire in ~60 seconds — on `/run/async`, tool dispatch may happen after expiry and the downstream API will 401. |
| `resumeRunId` | **Sync only.** Optional id of a finished chat run whose conversation is copied onto this new run. Ignored on `/run/async`. See [Resume a finished session](#resume-a-finished-session). |
| `resumeMethod` | **Sync only.** `exclude_tools` (default when `resumeRunId` is set), `all_messages`, or `summary`. Omitted `resumeRunId` starts a fresh run and ignores `resumeMethod`. |

### Run variables (`variables` → `{{input.*}}`)

Pass caller-defined named strings into a workflow at invocation time. Both sync
and async endpoints accept the same optional `variables` object.

**What happens**

1. The map is persisted on the run before execution starts.
2. Template tags `{{input.<key>}}` resolve to those values in workflow
   Instructions, system prompts, tool response markdown, and `input-tools`
   parameter mappings (see **BRA217** / **BRA409**).
3. Nested `run_workflow` / workflow-tool child runs inherit the parent's
   variables automatically.
4. Missing keys render blank (never an error). Omitting `variables` leaves
   `WorkflowRun.Variables` null — fully backward-compatible.

**Example — sync with variables**

```http
POST /workflows/WF-INPUT-VARIABLES/run/sync
Authorization: Bearer {brainApiKey}
Content-Type: application/json

{
  "inputMessage": "Produce the briefing.",
  "variables": {
    "topic": "Telos Brain Execution API",
    "seed_question": "What is the Execution API used for in one sentence?"
  }
}
```

**Example — async with variables**

```http
POST /workflows/WF-WIDGET/run/async
Authorization: Bearer {brainApiKey}
Content-Type: application/json

{
  "variables": {
    "widget_reference": "WID-001"
  },
  "callbackUrl": "https://app.example.com/brain/callbacks/run-complete"
}
```

Wire the same keys in the workflow schema:

- Instructions: `Working on {{input.topic}}.`
- Pre-called tools (`input-tools`):

  ```yaml
  input-tools:
    - variable: widget_information
      tool: get_widget_details
      parameters:
        widget_reference: "{{input.widget_reference}}"
  ```

Full authoring guide: **BRA409**. Template taxonomy: **BRA204** §3.6.

### `POST /workflows/{code}/run/sync` — synchronous (SSE)

Streams the result in real time as **Server-Sent Events**. The run is created at status `Running` (never `Queued`). Live progress events (`status`, `thinking`, `tool_call`, `tool_result`, and on Claude `text`) are emitted while the loop works. On non-streaming providers the final reply is still chunked as `text` deltas after the loop.

Rather than terminating, a successful sync run is **left open** as a chat session at status `AwaitingInput`, so you can continue the same conversation turn by turn (see [Chat Sessions](#chat-sessions)). It transitions to `Failed` on error or client disconnect. The stream begins with a `run_started` event carrying the run id you continue or close the session with.

Response headers:

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

While a turn is running, the server writes SSE **comment** keep-alives (`: keepalive`) every 15 seconds whenever no progress event has been sent. Comments are not `data:` events and must be ignored. They keep the HTTP connection active so Azure (and other proxies) do not close an idle SSE stream during a long tool call. A real client disconnect still fails the run.

Event stream:

```
data: {"type": "run_started", "runId": "44291..."}
data: {"type": "status", "phase": "model"}
data: {"type": "thinking", "delta": "I should look up the customer first."}
data: {"type": "tool_call", "name": "crm_lookup", "id": "toolu_01...", "input": {"customerId": "42"}}
data: {"type": "tool_result", "name": "crm_lookup", "id": "toolu_01...", "ok": true, "output": {"name": "Acme"}}
data: {"delta": "We", "type": "text"}
data: {"delta": " currently", "type": "text"}
data: {"delta": " have...", "type": "text"}
data: {"type": "done"}
```

Progress events are emitted **while the turn is running** so a harness chat UI can show steps (model wait, thinking, tool calls) instead of a silent wait. Unknown event types should be ignored so older clients stay compatible.

| `type` | When | Fields |
|---|---|---|
| `run_started` | Immediately, before work begins | `runId` |
| `status` | Each time the model is about to be called | `phase` (`model`) |
| `thinking` | While the provider is generating reasoning | `delta` (token-level on Claude streaming; one blob on non-streaming providers) |
| `tool_call` | Immediately before a tool is dispatched | `name`, `id`, `input` (JSON object when the arguments are JSON, otherwise a string) |
| `tool_result` | After that tool returns | `name`, `id`, `ok`, `output` (JSON object when the result is JSON, otherwise a string) |
| `text` | While the provider is writing the reply (Claude streaming), or after the loop if the call was non-streaming | `delta` |
| `error` | Terminal failure | `message` |
| `done` | Always last | — |

On Claude, `thinking` and `text` deltas are the provider's own tokens as they are generated. `tool_call` / `tool_result` still fire around tool dispatch. Non-streaming providers keep the previous behaviour (thinking as one blob after the call; reply text chunked at the end).

On failure, a terminal error event precedes `done`:

```
data: {"type": "error", "message": "Workflow 'foo' was not found."}
data: {"type": "done"}
```

> **Note:** because SSE commits a `200` response and headers before the first token, an unknown workflow code on the sync path surfaces as a terminal `error` event rather than a `404`. Use the async path if you need a hard `404` for an unresolved code.

### `POST /workflows/{code}/run/async` — asynchronous (fire-and-forget)

Creates the run at status `Queued`, enqueues a background job, and returns immediately.

Response `202 Accepted`:

```json
{
  "runId": "44291...",
  "status": "queued"
}
```

Returns `404 Not Found` if the workflow code does not resolve within the brain.

The engine executes the run in the background (`Queued → Running → Completed / Failed`). A run settles `Failed` (not `Completed`) when the model hits a terminal limit — the workflow's `max-turns` cap is exhausted, or the final `output-tokens` attempt stops with `max_tokens` and no further retry remains — as well as on transport / API errors and cancellation. If a `callbackUrl` was supplied, the engine **POSTs** the payload to that URL when the run finishes. The request URL is `callbackUrl` unchanged (an existing query string stays; the payload is not added to it). The body is the JSON payload, with `Content-Type: application/json; charset=utf-8`.

```http
POST https://app.example.com/brain/callbacks/run-complete
Content-Type: application/json; charset=utf-8

{
  "runId": "44291...",
  "status": "complete",
  "response": "We currently have three open proposals."
}
```

`status` is `complete` or `failed`. `response` is the assembled final assistant text: the same text the sync path streams as `text` deltas, or the error message when the run fails. It is JSON `null` when the run produced no reply. Quotes, backslashes and newlines inside `response` are JSON-escaped so the body stays valid.

Delivery is attempted up to three times. A non-success HTTP status or a transport error is retried; a persistent failure is logged and does **not** change the run status. The final attempt is stored on the run as a `role=callback` message (the URL, whether it succeeded, the HTTP status, and the receiver's body or the error when there was no HTTP response). That message is shown on the run and included in telemetry. It is not sent back to the model. If no `callbackUrl` is given, nothing is posted and no callback message is written.

#### Async callback SSRF rules

Outbound `callbackUrl` delivery is SSRF-hardened (same rules as declared-tool
`api.path` webhooks — see BRA213):

| Rule | Behaviour |
|---|---|
| Scheme | `https` required. In Development only, `http://localhost` and `http://127.0.0.1` (any port) are also allowed. |
| Host allowlist | Optional per-brain list from `brain-compose.yml` `allowed-callback-domains`. When set, the callback host must match an entry exactly (case-insensitive; no wildcards). When omitted, any public host is allowed. |
| IP blocklist | Always applied after DNS resolution: private, loopback, link-local, and cloud metadata ranges (e.g. `169.254.169.254`) are blocked in all environments. |
| Redirects | Automatic redirect following is disabled on the callback HTTP client. |
| Rejection | An unsafe `callbackUrl` marks the run `Failed` and no outbound POST is made. A `SYSTEM_CHANGE` inbox entry (`PROCESSED`) is created with the denied URL, reason, workflow code, and run id. |

Configure harness callback hosts in `brain-compose.yml` before using async runs:

```yaml
allowed-callback-domains:
  - app.example.com
```

---

## Chat Sessions

A synchronous run is a **chat session**: it is the *same* run for the whole conversation. Each turn appends to that run's message history, and the engine replays the run's own transcript as context on every subsequent turn — so the assistant remembers the conversation without you resending it. There is no separate session id: the **run id** (from the `run_started` event) *is* the session.

**Lifecycle.** A session moves through:

```
Running  →  AwaitingInput  ⇄  Running (next turn)  →  Completed
                    │
                    └──────────→  Completed (explicit close, or inactivity timeout)
```

After each successful turn the run settles at `AwaitingInput` (open) with an `expiresDateUtc`. It is closed to `Completed` when you call `complete`, or automatically when its inactivity timeout passes (see [Session timeout](#session-timeout)). Manual **Run eval** can target a `Completed`, `Failed`, or `AwaitingInput` run (BRA207). Automatic evals still fire on `Completed`.

**Cost and billing.** Each turn (including a sync call that leaves the run `AwaitingInput`) recalculates the run's LLM `CostCents` from its messages. OpenRouter turns that stored provider-billed `usage.cost` on every token-bearing message use that sum; otherwise cost is tokens × date-effective `LlmPrices` for the run's model (`openrouter` + catalogue id for OpenRouter; `telosbrain` + `xai/grok-4.6` for `telosbrain/xai/grok-4.6`, seeded at **2×** official xAI grok-4.6 short-context list prices and debited from organisation brain credit). Daily time-based charges include open sessions: the night job bills `RunSeconds - BilledSeconds` and then raises `BilledSeconds`, so a chat that stays open is charged that day and a later continuation only bills the unbilled remainder.

### `POST /runs/{runId}/messages` — continue a session

Posts the next user message to an open session and streams the turn as SSE, in the identical wire format to `/workflows/{code}/run/sync` (including a leading `run_started`). The run reuses its own entity / unit-of-work scope; the only per-turn input is the message.

```json
{
  "inputMessage": "And what about next quarter?"
}
```

Validated before the stream commits, so these are real HTTP statuses:

| Status | When |
|---|---|
| `200` (stream) | The session is open (`AwaitingInput`) and the turn is streaming. |
| `404 Not Found` | No run with that id belongs to the brain. The body is `{ "error": "No run with id '…' was found." }`. |
| `409 Conflict` | The run cannot take another message. The body is `{ "error": "<code>", "message": "…" }` **before** any SSE headers. |

`409` codes:

| `error` | When | `message` |
|---|---|---|
| `run_mid_turn` | Status is `Running` (or `Queued`): a turn is already in progress. | The run has a turn in progress; wait for it to finish. |
| `run_not_continuable` | Status is `Completed` or `Failed`. The session is closed. | `This session is closed and cannot be continued.` |

These are HTTP JSON bodies, not SSE `error` events. On `run_not_continuable`, start a **new** sync run with `resumeRunId` (see below). Do not post another message to the closed run. `run_mid_turn` is the same token `POST /runs/{runId}/complete` returns when a turn is still in progress.

A turn started here re-arms the session's `expiresDateUtc` from the moment it finishes, so an actively used session never times out.

### Resume a finished session

A finished run stays finished. To keep talking, start a new sync run that copies the previous conversation:

```http
POST /workflows/{code}/run/sync
Content-Type: application/json

{
  "inputMessage": "And what about next quarter?",
  "resumeRunId": "44291...",
  "resumeMethod": "exclude_tools"
}
```

`resumeMethod` defaults to `exclude_tools` when `resumeRunId` is set and the method is omitted. Omit `resumeRunId` — or send it empty — and the call starts a fresh run with no copied history, even if `resumeMethod` is present. That is the request existing clients already send. The new run uses the **requested** workflow: system prompt and instructions come from that workflow, not from the previous run. Entity, unit of work, inbox entry, variables, and `caller_jwt` come from this request only. `ParentWorkflowRunId` is not set.

| Source status | Result |
|---|---|
| `Completed` or `Failed` | Allowed. History is copied, then the new `inputMessage` is the first turn. |
| `AwaitingInput` | `409` `{ "error": "run_still_open", "message": "This session is still open. Continue it with POST /runs/{id}/messages." }` |
| `Running`, `Queued`, or `Scheduled` | `409` `{ "error": "run_mid_turn", "message": "This session has a turn in progress." }` |
| Unknown id, or another brain | `404` `{ "error": "No run with id '…' was found." }` |
| `resumeRunId` omitted or empty | Fresh run. `resumeMethod` is ignored. Same path as a request that predates resume. |
| Unknown `resumeMethod` when `resumeRunId` is set | `400` `{ "error": "invalid_resume_method", "message": "resumeMethod must be exclude_tools, all_messages, or summary." }` |

| `resumeMethod` | What is copied |
|---|---|
| `exclude_tools` (default) | User and assistant text only. Tool calls, tool results, thinking, and failed attempts are dropped. Compaction summaries are kept as `<context_summary>`. |
| `all_messages` | Active `user`, `assistant`, `tool`, and `compaction` rows, including tool name and tool-call id. The first turn of the new run replays those tool pairs. Later turns on the new run use the normal text transcript. |
| `summary` | The brain's `COMPACTION` workflow summarises the whole previous transcript. The summary is stored as a `compaction` message on the **new** run. The previous run is not modified (`CompactedAtUtc` is not set). If there is no COMPACTION workflow, or summarisation fails after one retry, the call returns `400` `{ "error": "resume_summary_failed", "message": "The previous session could not be summarised." }` and no chat run is created. Compaction cost sits on the nested COMPACTION run. |

Rows already superseded by compaction are not copied. The previous run's system prompt and workflow instructions are not copied.

Client flow: while the session is `AwaitingInput`, keep calling `POST /runs/{runId}/messages`. When that returns `run_not_continuable`, drop the old run id as the session handle and `POST /workflows/{code}/run/sync` with `resumeRunId` set to it. A `Failed` source is resumed the same way as a `Completed` one. An open session is not resumed; continue it instead.

### `POST /runs/{runId}/stop` — stop an in-flight turn

Stops the current turn if it is `Queued` or `Running` and leaves the session **open** (`AwaitingInput`) so you can send another message. This is not the same as `complete`: the run is not closed and is not evaluated automatically.

Response `200 OK`:

```json
{
  "runId": "44291...",
  "status": "AwaitingInput"
}
```

| Status | When |
|---|---|
| `200 OK` | The turn was stopped, or the run had already settled (idempotent). |
| `404 Not Found` | No run with that id belongs to the brain. |

Stopping aborts the in-flight turn, including any model request that is still in progress. Any assistant text already written stays on the run. The session timeout is re-armed from the moment of the stop. A connected SSE stream ends once the turn leaves `Queued` / `Running`.

A client disconnect on `/run/sync` or `/messages` (closed socket) still **fails** the run. Comment keep-alives keep the stream alive through long tool waits; they do not change this contract. If a turn may run longer than the *client* can stay connected, use `/run/async`.

To close the session after stopping, call `POST /runs/{runId}/complete`.

### `POST /runs/{runId}/complete` — close a session

Closes an open session, transitioning it to `Completed` so it becomes eligible for
learning evaluation (**BRA207**). If the brain has a `type: EVAL` workflow
whose `workflowrun:complete` trigger carries a learning-mode qualifier
(BRA351), an eval is enqueued automatically. Omit the trigger (or leave it
unqualified) for manual-only — use the admin UI **Run eval** button on the
run detail page (or `POST /brains/{instance}/runs/{runId}/eval` on the
Management API). The button is type-based and ignores the trigger.

Response `200 OK`:

```json
{
  "runId": "44291...",
  "status": "Completed"
}
```

Idempotent: closing an already-`Completed` session also returns `200`. Returns `404 Not Found` when no such run belongs to the brain, and `409 Conflict` `{ "error": "run_mid_turn", "message": "…" }` when a turn is still in progress — call `stop` first if you need to close immediately.

### Session timeout

An open session that is neither continued nor closed is swept to `Completed` once its `expiresDateUtc` passes, so abandoned chats do not linger. The inactivity window is set per workflow via the `session-timeout` frontmatter field (in minutes; see **BRA217**); when a workflow declares none, the engine default of **30 minutes** applies. The window is measured from the end of the most recent turn and re-armed on every continuation.

---

## Telemetry

### `GET /runs/{id}/telemetry` — run telemetry (OTEL)

Returns the full telemetry for a workflow run, projected to the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/). Each persisted turn becomes a span; token usage maps to `gen_ai.usage.*` and tool calls to `gen_ai.tool.*`.

Response `200 OK`:

```json
{
  "runId": "44291...",
  "workflowId": "9c1a...",
  "entityId": "3f0c...",
  "unitOfWorkId": "7b2d...",
  "workflowName": "Sales chat",
  "workflowPath": "workflows/sales-chat.md",
  "entityName": "Acme",
  "unitOfWorkName": "Onboard",
  "status": "Completed",
  "createdAt": "2026-07-09T07:10:00Z",
  "completedAt": "2026-07-09T07:10:04Z",
  "resource": {
    "gen_ai.system": "telos-brain",
    "gen_ai.request.model": "anthropic/claude-sonnet-4-6",
    "telos.thinking.mode": "effort",
    "gen_ai.request.temperature": 0.2,
    "gen_ai.request.top_p": 0.9,
    "gen_ai.request.top_k": 40,
    "gen_ai.request.min_p": 0.05,
    "gen_ai.request.presence_penalty": -0.5,
    "telos.workflow.name": "Sales chat",
    "telos.workflow.code": "WF-SALES",
    "telos.workflow.path": "workflows/sales-chat.md",
    "telos.entity.name": "Acme",
    "telos.unit_of_work.name": "Onboard"
  },
  "totals": {
    "gen_ai.usage.input_tokens": 1820,
    "gen_ai.usage.output_tokens": 430,
    "cacheRatePercent": 85,
    "gen_ai.embeddings.count": 2,
    "telos.turns.used": 3,
    "telos.turns.max": 15
  },
  "spans": [...]
}
```

Resource / totals extensions (Telos-specific, alongside GenAI semantic conventions):

| Attribute | Notes |
|---|---|
| `gen_ai.request.model` | Fully-qualified model the run executed against (`provider/model`) |
| `gen_ai.embeddings.count` | Embeddings generated in the run (always present on `totals`). OTEL standardises `gen_ai.embeddings.dimension.count` (vector size) only; this is the count of embeddings produced. |
| `cacheRatePercent` | Whole percent: cache reads / (uncached input + cache reads + cache writes). Null when all three are zero. |
| `gen_ai.request.temperature` | Workflow sampling temperature. Omitted when the workflow does not set one, so the provider default is not reported as `0` or `null`. `0` is included when the workflow set it. Read from the current workflow, the same way as thinking mode. |
| `gen_ai.request.top_p` | Workflow top-p. Omitted when unset. `0` is included when set. Read from the current workflow. Reported even when the provider request omitted the field. |
| `gen_ai.request.top_k` | Workflow top-k. Omitted when unset. `0` is included when set. Read from the current workflow. |
| `gen_ai.request.min_p` | Workflow min-p. Omitted when unset. `0` is included when set. Read from the current workflow. Not an official OTEL key. |
| `gen_ai.request.presence_penalty` | Workflow presence penalty. Omitted when unset. `0` is included when set. Read from the current workflow. |
| `telos.thinking.mode` | Workflow thinking mode (`none` \| `adaptive` \| `extended` \| `effort`) |
| `telos.workflow.name` | Workflow title (falls back to code). Also on the payload as `workflowName`. |
| `telos.workflow.code` | Workflow deploy code |
| `telos.workflow.path` | Brain-root-relative schema path of the executed workflow (e.g. `workflows/ask-for-advice.md`). Prefers the path captured at deploy; falls back to `workflows/{code}.md`. Also on the payload as `workflowPath`. |
| `telos.entity.name` | Entity name the run executed against (nullable) |
| `telos.unit_of_work.name` | Unit-of-work title the run executed against (nullable) |
| `telos.turns.used` | Completed assistant loop steps (excludes retries / `max_tokens` attempts) |
| `telos.turns.max` | Effective max-turns cap for the run (workflow value or engine default 10) |

Span attributes:

| Attribute | Notes |
|---|---|
| `gen_ai.message.role` | `user` \| `assistant` \| `tool` \| `compaction` \| `callback` |
| `gen_ai.message.content` | Message content |
| `gen_ai.usage.input_tokens` / `output_tokens` | Per-turn token counts |
| `gen_ai.usage.cache_read_input_tokens` / `cache_creation_input_tokens` | Present only when the provider reports prompt-cache usage |
| `gen_ai.tool.name` / `gen_ai.tool.call.id` | Present only on tool-call and tool-result turns |
| `gen_ai.tool.duration_ms` | Wall-clock milliseconds the tool took, on the tool-result turn only. Omitted when the row did not capture a duration. |
| `telos.callback.url` / `telos.callback.outcome` / `telos.callback.status_code` / `telos.callback.response` | Present on `role=callback` turns written after an async webhook POST. `outcome` is `succeeded` or `failed`. `status_code` is the receiver's HTTP status (omitted when the call produced no HTTP response). `response` is the receiver's body, or the error text when there was no HTTP response. These rows are not sent back to the model. |
| `gen_ai.embeddings.count` | Embeddings generated for this turn (usually the assistant tool-call row). Omitted when unused. |
| `gen_ai.response.finish_reason` | Provider stop reason on assistant turns (`end_turn` \| `tool_use` \| `max_tokens` \| ...) |
| `gen_ai.request.max_tokens` | The output token cap the attempt ran with; doubles per output-token retry |
| `error.message` | Present only on a failed / truncated assistant attempt |

An output-token retry (see **BRA217** `output-tokens`, the ordered per-attempt cap list) is not merged away: each attempt is its own assistant span carrying its `finish_reason` (`max_tokens` on a truncated attempt), the `max_tokens` cap it used and its consumed tokens, so a three-cap list that keeps truncating yields two truncated attempt spans before the final one. If that final attempt is still `max_tokens`, the run status is `Failed`.

Returns `404 Not Found` if the run does not belong to the brain.

### `GET /units-of-work/{id}/telemetry` — unit-of-work telemetry

Merges the unit of work's context and data logs into a single sequential stream.

**Ordering:** by event `date`, then record type (Context before Data) for identical timestamps, then `createdAt`.

Response `200 OK` — array of entries. Fields not applicable to a record type are `null`:

```json
[
  {
    "recordType": "Context",
    "date": "2026-07-09T07:06:00Z",
    "source": "agent",
    "title": "Draft created",
    "message": "Initial draft generated from template",
    "type": null,
    "body": null,
    "createdAt": "2026-07-09T07:06:00Z"
  },
  {
    "recordType": "Data",
    "date": "2026-07-09T07:06:30Z",
    "source": "tool:crm_lookup",
    "title": null,
    "message": null,
    "type": "tool_response",
    "body": "{ ... }",
    "createdAt": "2026-07-09T07:06:30Z"
  }
]
```

Returns `404 Not Found` if the unit of work does not belong to the brain.

---

## Endpoint Summary

| Method | Path | Purpose | Success |
|---|---|---|---|
| `POST` | `/workflows/{code}/run/sync` | Run workflow, stream SSE, open a chat session | `200` (stream) |
| `POST` | `/workflows/{code}/run/async` | Queue workflow run | `202` |
| `POST` | `/runs/{id}/messages` | Continue an open chat session (stream SSE) | `200` (stream) |
| `POST` | `/runs/{id}/stop` | Stop an in-flight turn and leave the session open | `200` |
| `POST` | `/runs/{id}/complete` | Close an open chat session | `200` |
| `GET` | `/runs/{id}/telemetry` | Run telemetry (OTEL GenAI) | `200` |
| `GET` | `/units-of-work/{id}/telemetry` | Merged context/data telemetry | `200` |