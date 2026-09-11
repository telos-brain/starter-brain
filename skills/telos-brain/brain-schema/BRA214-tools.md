---
name: Brain Schema Tools
code: BRA214
version: 1
description: How to author tool groups and tool YAML — api, mcp, system,
  workflow, and native tools, plus parameters (secrets, entity / unit-of-work /
  input bindings, URL path tokens, and outbound types). Load this when creating
  or editing a tool.
---

# Brain Schema Tools

Tools are organised into **groups**. The compose file points to a group
manifest (`tools.yml`); the manifest points to individual tool files. Overview
and versioning: **BRA201**. Compose entity / unit-of-work variables that
parameters can bind to: **BRA213**. Connectors used by API/MCP tools:
**BRA209**. Secrets: **BRA202**. Schema system tools: **BRA203**.

## 1. Tool group manifest (`tools/<group>/tools.yml`)

```yaml
name: Tickets                          # REQUIRED
description: Tools for reading and updating tickets.   # REQUIRED
tools:
  - add-ticket-comment.yml           # paths relative to this manifest
```

## 2. Tool definition (one file per tool)

Every tool declares **exactly one** execution block: `api`, `mcp`, `system`,
`workflow`, or `native`. Declaring zero or more than one is a hard error.

Common fields for all tools:

```yaml
name: add_ticket_comment               # REQUIRED — the AI-facing tool name
version: 1.0                           # optional (see **BRA201** versioning); defaults to 1
description: >-                         # REQUIRED
  Adds a comment to an existing ticket…
```

**API tool** (executed by calling an HTTP endpoint):

```yaml
api:
  method: POST                         # optional (maps to httpMethod)
  path: https://go.telosready.com/tool-api/add-ticket-comment   # REQUIRED (the webhook URL)
```

Write `{parameter-name}` in `api.path` to put that parameter in the URL
(see Parameters below). Relative paths on a connector work the same way
(`path: /v5/entities/{code}`).

The `api.path` host must satisfy the shared outbound allowlist in **BRA213**
(`allowed-callback-domains`) when that list is configured. `https` is required
outside Development; private and metadata IP ranges are always blocked.

**MCP tool** (invoked via an MCP server tool):

```yaml
mcp:
  tool: search                         # REQUIRED — MCP tool name on the server
  # Provide at least one outbound target (connector and/or direct URL / server):
  connector: my-mcp-connector          # optional — named connector (BRA209)
  server-url: https://mcp.example.com  # optional — direct MCP server URL
  server: my-mcp-server                # optional — legacy server label (still accepted)
```

A tool may use a connector, a direct `server-url` / `server`, or both. Connector
auth and URL resolution are in **BRA209**.

**System tool** (executed by an in-brain system tool, not an external call):

```yaml
system:
  tool: find_available_skills          # REQUIRED (the system tool name)
```

The schema system tools — which let a running brain inspect and edit its own
configuration-as-code files — are documented in BRA203. The inbox system tools —
list / get / update entries and tasks by **reference** (never UUID) — are
documented in BRA405. Run grading (`set_run_grading`) is documented in BRA406;
learning-eval authoring that uses it is in BRA207. Image URL transcription
(`transcribe_image`) is documented in BRA412.

**Workflow tool** (routes to another workflow in the same brain):

```yaml
workflow:
  code: WF-ASK-QUESTION                # REQUIRED (the target workflow's code)

parameters:
  - name: question                     # AI-facing name → {{input.question}}
    description: The question to answer.
    type: string
    required: true
```

The tool's `parameters` are resolved and passed into a fresh workflow-run for the
target workflow in two ways:

1. **Template variables (preferred)** — each resolved parameter is available in
   the target workflow's Instructions as `{{input.<name>}}` (see **BRA204** §3.6).
   Hardcoded `value:` params and `entity:`-bound params are included; `secret:`
   params are never forwarded into a child prompt.
2. **Markdown input message (legacy)** — the same values are also rendered as
   `## <name>\n<value>` sections on the child run's input message so older
   workflows that read the markdown still work.

That child run uses its own `model` and `tools`, inherits the calling run's
brain / entity / unit-of-work scope, and its final reply is returned as the tool
result. The target workflow is normally `type: TOOL`. Nesting is capped (depth 5)
to prevent runaway recursion.

**Example — target workflow Instructions using the param:**

```markdown
# Instructions

Answer this question directly and concisely:

{{input.question}}
```

**Native tool** (a built-in capability of the LLM, e.g. web access):

```yaml
native:
  type: web_search                     # REQUIRED — capability key (web_search | web_fetch)
```

A native tool is enabled directly on the model and executed by the provider. Unlike every other type it makes no outbound call and takes **no `parameters`**. Add it to a workflow's `tools` list by `name`; at run time it is passed to the model as a built-in capability. Supported keys: `web_search`, `web_fetch`.

## 3. Worked example: `web_search` and `web_fetch`

Native tools are authored like any other tool — as a tool group with one file per tool — then referenced from `brain-compose.yml` and enabled on the workflows that need them.

**Step 1 — Create the tool group manifest** (`tools/native/tools.yml`):

```yaml
name: Native tools
description: Provider-native (built-in) LLM capabilities such as web access.
tools:
  - web-search.yml
  - web-fetch.yml
```

**Step 2 — Create one file per native tool.** Each declares only `name`, `description`, and the `native` block — no `parameters`.

`tools/native/web-search.yml`:

```yaml
name: web_search
version: 1.0
description: >-
  Searches the web for up-to-date information and returns relevant results.
native:
  type: web_search
```

`tools/native/web-fetch.yml`:

```yaml
name: web_fetch
version: 1.0
description: >-
  Fetches the contents of a specific URL so the model can read the page.
native:
  type: web_fetch
```

**Step 3 — Register the group in `brain-compose.yml`** (unlisted files are not deployed):

```yaml
tools:
  - tools/native/tools.yml
```

**Step 4 — Enable the tools on any workflow that should use them**, by `name`:

```yaml
tools:
  - web_search
  - web_fetch
```

No endpoints, credentials, or MCP servers are involved — the capability is executed by the model provider itself.

---

## 4. Parameters

```yaml
parameters:
  - name: ticketReference              # REQUIRED — the AI-facing param name
    param: ticketReference             # optional — underlying key the call expects
                                       #   (defaults to `name`; legacy aliases:
                                       #    `api-param`, `targetKey`)
    description: >-                     # REQUIRED
      The ticket reference, e.g. "XXX037".
    type: string                       # outbound value type (see below)
    required: true                     # omit or false = optional; true = required
                                       #   in the LLM input schema

  # A parameter with a hardcoded `value` is FIXED and hidden from the LLM.
  # (legacy aliases: `api-value`, `apiValue`)
  - name: skillbooks
    param: skillbooks
    value: "ENG,OPS"                   # presence of `value` => not exposed to the LLM
    description: The skillbooks to search within.
    type: string

  # Use a non-string type when the target API expects a typed JSON value.
  # Agents always supply strings; the Tool Router coerces before dispatch.
  - name: order
    description: Sort order for the action.
    type: int
```

Key behaviour: a parameter is **exposed to the LLM only when it has no `value`,
`secret`, `entity`, `unitofwork`, `input` or `header`**. Set `value` to pin a
param and hide it. `name` and `description` are required on every parameter.

#### Putting a parameter in the URL

Any parameter can go in the URL. Write `{name}` in `api.path` — or `{param}`
if you set `param:`. The value is filled in when the tool runs and is **not**
also sent as a query string (GET) or JSON field (POST).

```yaml
name: get_nzbn_entity
description: Gets an NZBN entity by number.
api:
  method: GET
  path: https://api.business.govt.nz/gateway/nzbn/v5/entities/{nzbn}
parameters:
  - name: nzbn
    description: The NZBN number.
    required: true
```

At run time `{nzbn}` becomes the value the model supplied, e.g.
`…/entities/9429052360992`.

You can use more than one placeholder, including on a connector-relative path:

```yaml
api:
  method: GET
  connector: nzbn
  path: /v5/entities/{nzbn}/directors/{directorId}
parameters:
  - name: nzbn
    description: The NZBN number.
    required: true
  - name: directorId
    description: The director to return.
    required: true
```

If the URL token must differ from the AI-facing name, set `param:` to the
token:

```yaml
api:
  method: GET
  path: https://api.example.com/entities/{nzbn}
parameters:
  - name: nzbn_number
    param: nzbn
    description: The NZBN number.
    required: true
```

Optional: write `path: nzbn` on the parameter (instead of `param: nzbn`) to
mark that it belongs in the URL. Do not set both `path:` and `header:` on the
same parameter.

This works for ordinary parameters and for `secret:`, `entity:`,
`unitofwork:`, and `input:` parameters — put `{name}` or `{param}` in
`api.path` and the resolved value is placed there.

`required` is **false when omitted**. Set `required: true` for parameters the
model must supply (for example `url` on `transcribe_image`). Leave it off — or
set `required: false` — for optional parameters (for example `prompt`). The
flag is presented to the LLM in the tool input schema; the server does not
reject a missing argument at dispatch.

#### Parameter `type` (outbound coercion)

`type` declares the value type the Tool Router places on the outbound request
(POST JSON body or GET query string). Agents always emit strings; the router
converts the resolved value (model argument **or** hardcoded `value`) before
dispatch so typed APIs receive numbers/dates rather than `"1"`.

| `type`     | Coercion                                                         | Default |
| ---------- | ---------------------------------------------------------------- | ------- |
| `string`   | No-op — value stays a string                                     | yes (also when omitted) |
| `int`      | Parsed as an integer; strips `$`/`£`/`€`/… and commas; truncates decimals (`1.9`→`1`, `$1,234.99`→`1234`) |         |
| `decimal`  | Parsed with invariant culture (`.` as decimal separator); strips currency symbols (`$3.14`→`3.14`) |         |
| `date`     | Calendar date — multiple formats (see below)                     |         |
| `datetime` | Instant — multiple formats; emitted as UTC ISO-8601              |         |

**Date / datetime formats and timezones**

- Accepted date shapes include `yyyy-MM-dd`, `dd/MM/yyyy`, `MM/dd/yyyy`,
  `yyyy/MM/dd`, and dotted/dashed variants. Ambiguous values prefer **day-first**
  (British): `01/02/2026` → 1 February 2026.
- Accepted datetime shapes include ISO-8601 with `T` or a space, with optional
  fractional seconds, and with or without a `Z` / `±HH:MM` offset.
- When a datetime includes an explicit `Z` or offset, that instant is honoured.
- When a datetime has **no** timezone, it is interpreted in the brain's
  `TIMEZONE` environment variable (IANA id, e.g. `Pacific/Auckland`) — the same
  source as `{{now.local*}}`. If `TIMEZONE` is unset or unrecognised, UTC is used.
- Outbound `datetime` values are always serialised as UTC.
- If a `date` parameter is given a datetime string, the calendar date is taken
  in the brain timezone after resolving the instant (so a late UTC evening can
  become the next local day).

Parse failures return a clear tool error to the agent (e.g. could not convert
parameter `order` to type `int`) and the HTTP call is not made. Headers always
remain strings regardless of `type`. `required` is omitted/`false` by default
and is presented to the model only — the server does not reject a missing
argument at dispatch.

#### Injecting a secret / API key (api tools)

An `api` tool can authenticate to its endpoint by injecting a **stored brain
environment variable** — without the secret living in the schema. Three extra
fields drive this (all hide the parameter from the LLM):

```yaml
parameters:
  - name: authorization
    description: API key injected as the Authorization bearer token.
    header: Authorization              # send as this HTTP HEADER
    secret: ACME_API_KEY               # value = this brain env variable, decrypted at dispatch
    value: "Bearer {secret}"           # optional template; {secret} => the decrypted value
```

- `secret:` names a brain environment variable (uploaded from `.env`); its
  decrypted value is injected at dispatch. If the variable is not set, the
  parameter is omitted (logged and skipped), never sent as a placeholder.
- `value:` (with `secret:`) is a template where `{secret}` is replaced by the
  decrypted value; with no `value:`, the raw secret is injected as-is.
- `header:` chooses **where** the value goes:
  - **with** `header:` → sent as that named HTTP header;
  - **without** `header:` → sent in the request payload, i.e. the **query
    string** for a GET tool or the **JSON body** for a POST tool — unless
    `api.path` contains `{name}` or `{param}`, in which case it goes in the
    URL instead (see **Putting a parameter in the URL** above).
- Do not set both `header:` and `path:` on the same parameter.

Only `api` tools inject secrets — `mcp`/`system`/`workflow`/`native` tools make
no authenticated outbound HTTP call, so these fields have no effect there.

See **BRA202** for how environment variables are uploaded/encrypted, the
well-known provider key names, resolution order, and full worked examples
(header, query and body).

#### Binding a parameter to an entity variable

A parameter can pull its value from the **current entity's** stored data rather
than a secret, a hardcoded value or the model. Declare an `entity:` field naming
a variable key that the entity's type declares in `brain-compose.yml` (**BRA213**):

```yaml
parameters:
  - name: organisation_id
    description: The external organisation id for the current entity.
    param: organisationId              # underlying key the endpoint expects
    entity: organisationId             # inject the current entity's value for this variable
```

- `entity:` names an **entity variable key**. At dispatch the router looks up the
  value for that key on the run's current entity (the entity the workflow run is
  scoped to) and injects it under `param` (the target key).
- Like `secret` and `value`, an `entity`-bound parameter is **hidden from the
  LLM**.
- `header:` still chooses placement: with `header:` the value is sent as that
  HTTP header; without it, it goes in the query string (GET) or JSON body
  (POST) — or in the URL if `api.path` contains `{name}` or `{param}`.
- **If the current entity has no value for the key** (or the run has no entity in
  scope), the parameter is **omitted** from the request — never sent blank. Set
  the value via the Execution API (BRA402) so it resolves.

`api` / `system` tools inject the bound value into the outbound call / executor
arguments. `workflow` tools expose it as `{{input.<name>}}` (and the legacy
markdown input). The binding is ignored by `native` tools (which take no
parameters).

#### Binding a parameter to a unit-of-work variable

A parameter can equally pull its value from the **current unit of work's** stored
data. Declare a `unitofwork:` field naming a variable key that the unit of work's
type declares in `brain-compose.yml` (**BRA213**):

```yaml
parameters:
  - name: job_id
    description: The external job id for the current unit of work.
    param: jobId                       # underlying key the endpoint expects
    unitofwork: jobId                  # inject the current unit of work's value for this variable
```

- `unitofwork:` names a **unit-of-work variable key**. At dispatch the router
  looks up the value for that key on the run's current unit of work (the unit of
  work the workflow run is scoped to) and injects it under `param`.
- Semantics match `entity:` exactly: the parameter is **hidden from the LLM**,
  `header:` still chooses placement (or put `{name}` / `{param}` in
  `api.path` to send it in the URL), and **if the current unit of work has no
  value for the key** (or the run has no unit of work in scope) the parameter
  is **omitted** from the request rather than sent blank.
- A single tool may mix `entity:`- and `unitofwork:`-bound parameters; each
  resolves against its own scope.

#### Binding a parameter to a workflow input variable

A parameter can pull its value from the **current run's input bag** — the same
keys as `{{input.*}}` (Execution API `variables` plus workflow-tool /
`run_workflow` parameters). Declare an `input:` field naming that key:

```yaml
parameters:
  - name: userId
    description: Acting user id injected from this workflow run.
    param: userId
    input: userId
```

- `input:` names a key in the merged input bag (case-insensitive). At dispatch
  the router injects that value under `param`.
- Like `entity` and `unitofwork`, an `input`-bound parameter is **hidden from
  the LLM**.
- **If the run has no value for the key** (or the value is blank), the call
  **fails** with a clear error — it is not sent empty. Pass the key as an
  Execution API `variables` entry or as a workflow-tool / `run_workflow`
  parameter.
- Resolution order when several bindings are set on one parameter: `secret` →
  `entity` → `unitofwork` → `input` → hardcoded `value` → model argument.

#### End-to-end example: an authenticated API tool that uses a variable

Putting the group manifest and parameter rules together — a complete, authenticated API tool from scratch.

**Step 1 — Declare the secret in `.env`** (uploaded on deploy, encrypted at
rest; never commit the real file):

```bash
# .env  (next to brain-compose.yml)
ACME_API_KEY=sk_live_xxx
```

**Step 2 — Create the tool group manifest** (`tools/acme/tools.yml`):

```yaml
name: Acme
description: Tools for creating and reading Acme widgets.
tools:
  - create-widget.yml
```

**Step 3 — Define the tool** (`tools/acme/create-widget.yml`). One injected
secret (hidden from the LLM) plus one model-supplied parameter:

```yaml
name: create_widget
version: 1
description: Creates a widget in Acme via its HTTP API.
api:
  method: POST
  path: https://api.acme.example.com/widgets
parameters:
  # Injected secret — hidden from the model, sent as the Authorization header.
  - name: authorization
    description: Acme API key, injected as the Authorization bearer token.
    header: Authorization
    secret: ACME_API_KEY
    value: "Bearer {secret}"
  # Exposed — the model supplies this in the JSON body.
  - name: name
    description: The display name of the widget to create.
    type: string
    required: true
```

**Step 4 — Register the group in `brain-compose.yml`** (unlisted files are not
deployed):

```yaml
tools:
  - tools/acme/tools.yml
```

**Step 5 — Enable the tool on a workflow**, by `name`:

```yaml
tools:
  - create_widget
```

At run time, when the model calls `create_widget` with `{ "name": "Sprocket" }`,
the dispatched request is:

```
POST https://api.acme.example.com/widgets
Authorization: Bearer sk_live_xxx

{ "name": "Sprocket" }
```

To authenticate via a query parameter instead (a GET endpoint keyed by
`?api_key=…`), drop `header:` and set `method: GET` — see BRA202 §3.3.

---
