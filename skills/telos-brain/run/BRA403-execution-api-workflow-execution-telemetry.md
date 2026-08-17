---
name: "Execution API: Workflow Execution & Telemetry"
code: BRA403
version: 12
description: How to list a brain's workflows (with pending inbox-task counts),
  run them synchronously (SSE streaming) or asynchronously (fire-and-forget
  with callback), pass optional run variables for {{input.*}} template tags
  and pre-called input-tools (BRA409), hold a multi-turn chat session against
  a single run, and retrieve run telemetry and unit-of-work telemetry via the
  Execution API — including model, thinking mode and turn totals on run
  telemetry. Documents when a run settles Failed (max-turns exhausted, final
  max_tokens with no retries). Also notes when a Completed run becomes
  eligible for learning evals (BRA207). Documents SSRF rules for async
  callbackUrl (allowed-callback-domains).
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
  "callbackUrl": "https://app.example.com/brain/callbacks/run-complete"
}
```

| Field | Notes |
|---|---|
| `inputMessage` | The triggering user message. |
| `entityId` | Optional runtime scope. |
| `unitOfWorkId` | Optional runtime scope for template tags (e.g. `{{#unitOfWork.context}}`). Nothing is hard-coded into the prompt; workflows that need the logs declare those tags in Instructions. |
| `variables` | Optional string→string map persisted on the `WorkflowRun` and exposed to templates as `{{input.<key>}}`. Omit or leave null for unchanged behaviour. Keys and values are plain strings (no type coercion). See [Run variables](#run-variables-input) and **BRA409**. |
| `callbackUrl` | Honoured on the async path only. Must be `https` (or `http://localhost` / `http://127.0.0.1` in Development). Subject to the brain's shared outbound host allowlist — see [Async callback SSRF rules](#async-callback-ssrf-rules). |

### Run variables (`variables` → `{{input.*}}`)

Pass caller-defined named strings into a workflow at invocation time. Both sync
and async endpoints accept the same optional `variables` object.

**What happens**

1. The map is persisted on the `WorkflowRun` as JSON before execution starts
   (so Hangfire async jobs can read it from the database).
2. Template tags `{{input.<key>}}` resolve to those values in workflow
   Instructions, system prompts, tool response markdown, and `input-tools`
   parameter mappings (see **BRA201** §8.0a / **BRA409**).
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

Streams the result in real time as **Server-Sent Events**, mirroring the Claude streaming wire format. The run is created at status `Running` (never `Queued`).

Rather than terminating, a successful sync run is **left open** as a chat session at status `AwaitingInput`, so you can continue the same conversation turn by turn (see [Chat Sessions](#chat-sessions)). It transitions to `Failed` on error or client disconnect. The stream begins with a `run_started` event carrying the run id you continue or close the session with.

Response headers:

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

Event stream:

```
data: {"type": "run_started", "runId": "44291..."}
data: {"delta": "We", "type": "text"}
data: {"delta": " currently", "type": "text"}
data: {"delta": " have...", "type": "text"}
data: {"type": "done"}
```

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

The engine executes the run in the background (`Queued → Running → Completed / Failed`). A run settles `Failed` (not `Completed`) when the model hits a terminal limit — the workflow's `max-turns` cap is exhausted, or the final `output-tokens` attempt stops with `max_tokens` and no further retry remains — as well as on transport / API errors and cancellation. If a `callbackUrl` was supplied, a webhook is POSTed on completion:

```http
POST https://app.example.com/brain/callbacks/run-complete
Content-Type: application/json

{
  "runId": "44291...",
  "status": "complete"
}
```

`status` is `complete` or `failed`. Webhook delivery is retried on transient failure; a persistent failure is logged but does not fail the run. Results are always retrievable via telemetry.

#### Async callback SSRF rules

Outbound `callbackUrl` delivery is SSRF-hardened (same rules as declared-tool
`api.path` webhooks — see BRA201 §4.3):

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

After each successful turn the run settles at `AwaitingInput` (open) with an `expiresDateUtc`. It is closed to `Completed` when you call `complete`, or automatically when its inactivity timeout passes (see [Session timeout](#session-timeout)). A run is only evaluated once it reaches a terminal status, so an open session is not evaluated until it closes.

**Cost and billing.** Each turn (including a sync call that leaves the run `AwaitingInput`) recalculates the run's LLM `CostCents` from its messages. Daily time-based charges include open sessions: the night job bills `RunSeconds - BilledSeconds` and then raises `BilledSeconds`, so a chat that stays open is charged that day and a later continuation only bills the unbilled remainder.

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
| `404 Not Found` | No run with that id belongs to the brain. |
| `409 Conflict` | The run has a turn in progress, or is `Completed` / `Failed` and can no longer be continued. |

A turn started here re-arms the session's `expiresDateUtc` from the moment it finishes, so an actively used session never times out.

### `POST /runs/{runId}/complete` — close a session

Closes an open session, transitioning it to `Completed` so it becomes eligible for
learning evaluation (BRA207 / BRA091). If the brain has a `workflowrun:complete`
workflow with `trigger-mode: automatic`, an eval is enqueued. If only
`trigger-mode: manual` (or omitted) is configured, use the admin UI **Run eval**
button on the run detail page (or `POST /brains/{instance}/runs/{runId}/eval` on
the Management API).

Response `200 OK`:

```json
{
  "runId": "44291...",
  "status": "Completed"
}
```

Idempotent: closing an already-`Completed` session also returns `200`. Returns `404 Not Found` when no such run belongs to the brain, and `409 Conflict` when the run cannot be closed in its current state (e.g. a turn is still in progress).

### Session timeout

An open session that is neither continued nor closed is swept to `Completed` once its `expiresDateUtc` passes, so abandoned chats do not linger. The inactivity window is set per workflow via the `session-timeout` frontmatter field (in minutes; see BRA201); when a workflow declares none, the engine default of **30 minutes** applies. The window is measured from the end of the most recent turn and re-armed on every continuation.

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
  "status": "Completed",
  "createdAt": "2026-07-09T07:10:00Z",
  "completedAt": "2026-07-09T07:10:04Z",
  "resource": {
    "gen_ai.system": "telos-brain",
    "gen_ai.request.model": "anthropic/claude-sonnet-4-6",
    "telos.thinking.mode": "effort"
  },
  "totals": {
    "gen_ai.usage.input_tokens": 1820,
    "gen_ai.usage.output_tokens": 430,
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
| `telos.thinking.mode` | Workflow thinking mode (`none` \| `adaptive` \| `extended` \| `effort`) |
| `telos.turns.used` | Completed assistant loop steps (excludes retries / `max_tokens` attempts) |
| `telos.turns.max` | Effective max-turns cap for the run (workflow value or engine default 10) |

Span attributes:

| Attribute | Notes |
|---|---|
| `gen_ai.message.role` | `user` \| `assistant` \| `tool` |
| `gen_ai.message.content` | Message content |
| `gen_ai.usage.input_tokens` / `output_tokens` | Per-turn token counts |
| `gen_ai.usage.cache_read_input_tokens` / `cache_creation_input_tokens` | Present only when the provider reports prompt-cache usage |
| `gen_ai.tool.name` / `gen_ai.tool.call.id` | Present only on tool-call and tool-result turns |
| `gen_ai.response.finish_reason` | Provider stop reason on assistant turns (`end_turn` \| `tool_use` \| `max_tokens` \| ...) |
| `gen_ai.request.max_tokens` | The output token cap the attempt ran with; doubles per output-token retry |
| `error.message` | Present only on a failed / truncated assistant attempt |

An output-token retry (see BRA201 `output-tokens`, the ordered per-attempt cap list) is not merged away: each attempt is its own assistant span carrying its `finish_reason` (`max_tokens` on a truncated attempt), the `max_tokens` cap it used and its consumed tokens, so a three-cap list that keeps truncating yields two truncated attempt spans before the final one. If that final attempt is still `max_tokens`, the run status is `Failed`.

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
| `POST` | `/runs/{id}/complete` | Close an open chat session | `200` |
| `GET` | `/runs/{id}/telemetry` | Run telemetry (OTEL GenAI) | `200` |
| `GET` | `/units-of-work/{id}/telemetry` | Merged context/data telemetry | `200` |