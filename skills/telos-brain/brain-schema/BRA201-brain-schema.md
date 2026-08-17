---
name: Brain Schema
code: BRA201
version: 37
description: How to setup a brain schema using yml and markdown
---

# Authoring a Telos Brain Schema

This document describes the `brain-schema` format precisely enough for an AI
agent (or a person) to reproduce it correctly. A Telos Brain is defined as
**configuration-as-code**: a set of YAML and markdown files that the Telos Brain
CLI (`brain deploy`) parses and uploads to the Management API. The server never
sees the raw files — the CLI parses them into JSON and POSTs them.

Everything here is derived from the CLI parsers, so it matches deployment
behaviour exactly. Follow it literally.

---

## 1. Mental model

A brain is composed from a single entry-point manifest (`brain-compose.yml`)
that **points to** self-contained definitions for these kinds of thing:

| Concept       | What it is                                                        | Defined by                          |
| ------------- | ----------------------------------------------------------------- | ----------------------------------- |
| **Entities**  | Top-level things the brain reasons about (e.g. `Application`).     | Inline in `brain-compose.yml`.      |
| **Units of work** | A scoped piece of work operating across entities (e.g. `Ticket`). | Inline in `brain-compose.yml`.  |
| **Connectors** | Named external-service integrations (URL, auth, declared params). | Connector YAML (`connectors/{name}.yml`). See **BRA209**. |
| **Tools**     | Callable actions (HTTP API, MCP, or in-brain system tools).       | Tool-group folders (`tools.yml`).   |
| **Skills**    | Reusable knowledge/practices, grouped into skillbooks.            | Skillbook folders (`skillbook.yml`). |
| **Blueprints** | Long-form scoped knowledge (vision, architecture, concepts…).     | Blueprint folders (`blueprint.yml`). |
| **Workflows** | Runnable instructions that wire together tools + skills.          | A single markdown file per workflow. |

Two authoring styles are used, deliberately:

- **YAML** for *structured wiring* — manifests, endpoints, parameters,
  categories, scopes.
- **Markdown with YAML frontmatter** for *long-form content* — skills, blueprint
  entries, and workflow instructions. The frontmatter carries metadata; the
  markdown body is the content itself.

All referenced paths inside a manifest are **relative to that manifest's own
folder** (no `./` prefix needed).

---

## 2. Directory layout

A representative layout (names are conventional, not required — the compose file
is the source of truth for what gets deployed):

```
brain-schema/
  brain-compose.yml              # entry point — everything is referenced from here
  package.json                   # provides `npm run deploy`
  .env.example                   # template for deploy credentials (copy to .env)
  .gitignore                     # ignores .env, node_modules, brain.lock

  connectors/
    example-oauth2.yml           # one connector definition per file (BRA209)
    example-api-key.yml

  tools/
    tickets/
      tools.yml                  # tool group manifest
      add-ticket-comment.yml     # one tool definition per file

  skills/
    eng/
      skillbook.yml              # skillbook manifest (declares categories)
      backend/
        EP101-database-migrations.md   # one skill per markdown file
      frontend/
        EP201-component-design.md

  blueprints/
    product-brain/
      blueprint.yml              # blueprint manifest (declares scope + categories)
      vision-overview.md         # one entry per markdown file (tagged by category)
      system-architecture.md

  workflows/
    review-blueprint.md          # one workflow per markdown file (self-contained)
```

**Do not commit** `.env` (real credentials), `node_modules/`, or `brain.lock`
(local deploy state, akin to `terraform.tfstate`).

---

## 3. Deploy workflow

From the schema folder:

```bash
npm run deploy         # brain deploy .
npm run deploy:dry     # brain deploy . --dry-run  (parse + validate only, no API calls)
```

Key facts:

- The CLI resolves the compose file by looking for, in order:
  `brain-compose.yml`, `brain-compose.yaml`, `brain.yml`, `brain.yaml` — or you
  pass an explicit `.yml` path.
- On **first deploy** an instance name is required: `brain deploy . --instance <name>`.
  Thereafter it's remembered in `brain.lock`.
- **Instance name** must be a DNS-style slug: 3–63 chars, lowercase letters,
  digits and internal hyphens only, no leading/trailing hyphen.
- To duplicate an existing instance's configuration into a new slug (e.g.
  production → staging), use the Management API clone endpoint — see **BRA205**.
- To pull newer template configuration into a previously cloned instance without
  overwriting destination resources that are already ahead, use update-from —
  see **BRA206** (same version-precedence rule as §9).
- Credentials and destination load from `.env` or `.env.<env>` next to the
  compose file (`TELOS_BRAIN_ORG_API_KEY`, `TELOS_BRAIN_API_URL`; legacy
  `TELOS_ORG_API_KEY` / `TELOS_API_URL` still accepted). Use
  `brain deploy --env <local|dev|stage|prod>` to select a named file. Real
  environment variables override `.env`, so CI secrets always win.
- The **whole brain is parsed up front**, so any schema error fails the deploy
  before a single API call is made. Use `--dry-run` while authoring.
- Deploy order is fixed: **skills → connectors → tools → workflows → memory
  (blueprints)** (then entity / unit-of-work types). Connectors precede tools so
  tool definitions can eventually reference connector names. Workflows reference
  skills/tools by code/name, so those codes must be correct.

---

## 4. `brain-compose.yml` (the entry point)

```yaml
name: kappa                      # REQUIRED: the brain's name
# description: optional          # optional; used as the brain description on first deploy

# Optional brain-level settings (persisted on every `brain deploy`):
# embedding-model: voyage-3-lite # optional; defaults to voyage-3-lite when omitted
# learning-mode: off             # optional; off | low | medium | high (omit = off)
# checkpoint-strategy: Daily     # optional; see §4.2 (omit = Daily)
# allowed-callback-domains:      # optional; shared outbound host allowlist (see §4.3)
#   - harness.example.com

# Entities: top-level things the brain reasons about.
entities:
  - name: Application            # REQUIRED
    code: application            # REQUIRED (referenced by scopes elsewhere)
    # variables: optional per-entity variables (see §4.1)
    variables:
      - key: organisationId      # REQUIRED (the variable key)
        description: External CRM organisation ID.   # optional

# Units of work: scoped pieces of work. `scope` lists the entity/unit codes it
# operates across, in shorthand form.
unitsofwork:                     # also accepted as `unitsOfWork`
  - name: Ticket                 # REQUIRED
    code: ticket                 # REQUIRED
    scope: entity:application    # optional
    # variables: optional per-unit-of-work variables (see §4.1)
    variables:
      - key: jobId               # REQUIRED (the variable key)
        description: External job ID for this ticket.   # optional

# Each of the following is a LIST OF PATHS to self-contained definitions.
connectors:
  - connectors/example-oauth2.yml

tools:
  - tools/tickets/tools.yml

skills:
  - skills/eng/skillbook.yml
  - skills/ops/skillbook.yml

blueprints:
  - blueprints/product-brain/blueprint.yml
  - blueprints/application/blueprint.yml

workflows:
  - workflows/review-blueprint.md
```

Rules:

- `name` is the only required top-level field.
- `embedding-model` is optional (defaults to `voyage-3-lite` when omitted).
- `learning-mode` is optional (`off` | `low` | `medium` | `high`; omit or null is
  treated as `off`). Persisted onto the Brain on every deploy.
- `checkpoint-strategy` is optional (see §4.2).
- `allowed-callback-domains` is optional (see §4.3) — shared host allowlist for
  async run callbacks and declared-tool webhook URLs.
- `entities` / `unitsofwork` are optional lists; each item needs `name` + `code`.
- `connectors`, `tools`, `skills`, `blueprints`, `workflows` are optional lists of
  relative paths. Anything **not listed here is not deployed**, even if the file
  exists on disk.

### 4.1 Entity and unit-of-work variables (per-instance key/value pairs)

An entity type can declare **variables** — named slots that each *instance* of
that type can fill with a scalar value (e.g. an external `organisationId`, an
account code, a region). The **keys** are schema (declared here, in
`brain-compose.yml`); the **values** are runtime data set per entity instance
via the Execution API (see BRA402).

```yaml
entities:
  - name: Customer
    code: customer
    variables:
      - key: organisationId                       # REQUIRED (the variable key)
        description: The external CRM organisation ID for this customer.  # optional
      - key: accountCode
        description: The billing account code.
```

Rules:

- `variables` is an optional list under an entity; each item needs a `key`
  (`description` is optional). Keys are unique per entity type.
- Deploying is **upsert-always** (like the entity type itself): new keys are
  added, existing ones refresh their description, and keys removed from the list
  are retired on the next deploy.
- The point of declaring a variable is so a **tool parameter can bind to it** and
  have the current entity's value injected automatically at dispatch — see the
  `entity:` parameter field in §5.3.

A **unit-of-work type** declares variables in exactly the same way, for data that
belongs to a single piece of work rather than to the entity behind it (e.g. the
external `jobId` the harness created for this ticket):

```yaml
unitsofwork:
  - name: Ticket
    code: ticket
    scope: entity:customer
    variables:
      - key: jobId                                # REQUIRED (the variable key)
        description: The external job ID for this ticket.   # optional
      - key: batchCode
        description: The processing batch this ticket belongs to.
```

The same rules apply — keys are unique per unit-of-work type, deploying is
upsert-always, and values are set per unit-of-work instance via the Execution API
(BRA402). A tool parameter binds to one with the `unitofwork:` field (§5.3).

Choose by lifetime: put a value on the **entity** when it is stable across every
piece of work for that record; put it on the **unit of work** when it is specific
to this job. A tool can bind to both at once.

### 4.2 Checkpoint strategy

Checkpoints are lightweight point-in-time markers for a Brain's schema (skills,
workflows, tools, blueprint entries). Creation is cheap; schema components are
copied on write when they change after a checkpoint exists — each snapshot stores
the full serialised `.md`/`.yml` file content so diffs and reverts round-trip
tools, parameters, and all frontmatter. Runtime data (workflow runs, entities,
units of work, inbox) is out of scope.

Configure the schedule in `brain-compose.yml` (persisted on every `brain deploy`).
Strategy is **not** editable via the Management API or Settings UI. Retention
(`MaxCheckpoints`) is a system setting — not writable from the brain schema.

```yaml
# Scalar (single strategy):
checkpoint-strategy: Daily

# Comma-separated (multiple strategies):
checkpoint-strategy: Weekly,BeforeDeploy

# YAML list (equivalent):
checkpoint-strategy:
  - Weekly
  - BeforeDeploy
```

**`checkpoint-strategy`** — one or more of:

| Value | Kind | When a checkpoint is created |
| ----- | ---- | ---------------------------- |
| `BeforeDeploy` | Event | Immediately **before** schema resource phases on `brain deploy` (server-side). Copy-on-write then captures the pre-deploy schema as resources mutate. |
| `AfterDeploy` | Event | Immediately **after** all schema resource phases on `brain deploy` (server-side). Marks the post-deploy schema as the restore baseline. |
| `Daily` | Schedule | Once per day (Hangfire / `NextCheckpointUtc`). Equivalent aliases: `@daily`. |
| `Weekly` | Schedule | Once per week. Equivalent aliases: `@weekly`, `0 0 * * 0`. |
| *(cron)* | Schedule | Any other free-form cron expression (e.g. `0 30 9 * * 1-5`). |

Defaults: omit or blank → `Daily`. Multiple values are normalised to PascalCase
and stored comma-separated. **Every listed strategy is active** — they form a
union (a checkpoint fires when any strategy triggers). Event and schedule
strategies may be combined (e.g. `Weekly,BeforeDeploy` or
`Daily,Weekly,BeforeDeploy,AfterDeploy`).

Rules:

- Event strategies (`BeforeDeploy` / `AfterDeploy`) are handled **server-side**
  during deploy — the CLI does not create checkpoints. Both may be set together;
  each fires at its phase.
- Schedule strategies (`Daily` / `Weekly` / cron) are armed on deploy via
  `NextCheckpointUtc` (soonest next fire across **all** schedule entries). A
  Hangfire job sweeps due brains. Multiple schedules are a union, not
  first-wins.
- Event-only strategies clear `NextCheckpointUtc` so Hangfire does not invent a
  schedule that was not configured.
- Do not set `max-checkpoints` in `brain-compose.yml` — it is ignored on deploy.

### 4.3 Allowed callback / webhook domains (SSRF allowlist)

The Brain makes outbound HTTP calls to harness-owned URLs in two places:

1. **Async run callbacks** — optional `callbackUrl` on
   `POST /workflows/{code}/run/async` (see BRA403).
2. **Declared API tools** — `api.path` webhook URLs dispatched by the Tool Router
   during a run.

To prevent server-side request forgery (SSRF), both surfaces share one per-brain
host allowlist configured in `brain-compose.yml` and persisted on every
`brain deploy`. The field is **not** editable via Management API PATCH or the
Settings UI.

```yaml
# Scalar (single host):
allowed-callback-domains: harness.example.com

# Comma-separated:
allowed-callback-domains: harness.example.com, hooks.example.org

# YAML list (equivalent):
allowed-callback-domains:
  - harness.example.com
  - hooks.example.org
```

Rules:

- Hostnames are matched **exactly** (case-insensitive). No wildcards, no
  subdomain matching (`evil.harness.example.com` does not match
  `harness.example.com`).
- Ports are not part of the allowlist entry — match is on the DNS host only.
- Omit or blank → no host allowlist. Private / loopback / link-local / cloud
  metadata IP ranges are still blocked after DNS resolution in all environments.
- Schemes: `https` only in non-Development. In Development,
  `http://localhost` and `http://127.0.0.1` (any port) are also permitted for
  local harnesses.
- When the list is populated, a `callbackUrl` or tool `api.path` whose host is
  not listed is rejected (async run → `Failed`; tool call → error result, no
  outbound request). A `SYSTEM_CHANGE` inbox entry (status `PROCESSED`) is also
  created with the denied URL, reason, and the workflow/tool/run involved.
- Redirect following is disabled on outbound webhook HTTP clients so a redirect
  chain cannot reach an internal target after the initial URL passed validation.

Configure every harness hostname that will receive async completion callbacks or
declared-tool webhooks before deploying tools / using async runs against those
hosts.

---

## 5. Tools

Tools are organised into **groups**. The compose file points to a group manifest
(`tools.yml`); the manifest points to individual tool files.

### 5.1 Tool group manifest (`tools/<group>/tools.yml`)

```yaml
name: Tickets                          # REQUIRED
description: Tools for reading and updating tickets.   # REQUIRED
tools:
  - add-ticket-comment.yml           # paths relative to this manifest
```

### 5.2 Tool definition (one file per tool)

Every tool declares **exactly one** execution block: `api`, `mcp`, `system`,
`workflow`, or `native`. Declaring zero or more than one is a hard error.

Common fields for all tools:

```yaml
name: add_ticket_comment               # REQUIRED — the AI-facing tool name
version: 1.0                           # optional (see §9 versioning); defaults to 1
description: >-                         # REQUIRED
  Adds a comment to an existing ticket…
```

**API tool** (executed by calling an HTTP endpoint):

```yaml
api:
  method: POST                         # optional (maps to httpMethod)
  path: https://go.telosready.com/tool-api/add-ticket-comment   # REQUIRED (the webhook URL)
```

The `api.path` host must satisfy the shared outbound allowlist in §4.3
(`allowed-callback-domains`) when that list is configured. `https` is required
outside Development; private and metadata IP ranges are always blocked.

**MCP tool** (invoked via an MCP server tool — BRA200):

```yaml
mcp:
  tool: search                         # REQUIRED — MCP tool name on the server
  # Provide at least one outbound target (connector and/or direct URL / server):
  connector: my-mcp-connector          # optional — named connector (BRA209)
  server-url: https://mcp.example.com  # optional — direct MCP server URL
  server: my-mcp-server                # optional — legacy server label (still accepted)
```

A tool may use a connector, a direct `server-url` / `server`, or both. Runtime
MCP protocol dispatch is BRA201; connector auth/URL resolution for API tools is
BRA199.

**System tool** (executed by an in-brain system tool, not an external call):

```yaml
system:
  tool: find_available_skills          # REQUIRED (the system tool name)
```

The schema system tools — which let a running brain inspect and edit its own
configuration-as-code files — are documented in BRA203. The inbox system tools —
list / get / update entries and tasks by **reference** (never UUID) — are
documented in BRA405. Run grading (`set_run_grading`) is documented in BRA406;
learning-eval authoring that uses it is in BRA207.

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

A native tool is enabled directly on the model and executed by the provider. Unlike every other type it is **not** routed through the tool router, makes no outbound call, and takes **no `parameters`**. Add it to a workflow's `tools` list by `name`; at run time it is passed to the model as a built-in capability. Supported keys: `web_search`, `web_fetch`.

### 5.4 Worked example: `web_search` and `web_fetch`

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

### 5.3 Parameters

```yaml
parameters:
  - name: ticketReference              # REQUIRED — the AI-facing param name
    param: ticketReference             # optional — underlying key the call expects
                                       #   (defaults to `name`; legacy aliases:
                                       #    `api-param`, `targetKey`)
    description: >-                     # REQUIRED
      The ticket reference, e.g. "XXX037".
    type: string                       # outbound value type (see below)
    required: true                     # advisory flag — not enforced by the server

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
`secret`, `entity` or `header`**. Set `value` to pin a param and hide it. `name`
and `description` are required on every parameter.

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
remain strings regardless of `type`. `required` remains advisory only.

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
  - **with** `header:` → sent as that named HTTP header (target key = the header
    name);
  - **without** `header:` → sent in the request payload, i.e. the **query
    string** for a GET tool or the **JSON body** for a POST tool.

Only `api` tools inject secrets — `mcp`/`system`/`workflow`/`native` tools make
no authenticated outbound HTTP call, so these fields have no effect there.

See **BRA202** for how environment variables are uploaded/encrypted, the
well-known provider key names, resolution order, and full worked examples
(header, query and body).

#### Binding a parameter to an entity variable

A parameter can pull its value from the **current entity's** stored data rather
than a secret, a hardcoded value or the model. Declare an `entity:` field naming
a variable key that the entity's type declares in `brain-compose.yml` (§4.1):

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
  HTTP header; without it, it goes in the query string (GET) or JSON body (POST).
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
type declares in `brain-compose.yml` (§4.1):

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
  `header:` still chooses placement, and **if the current unit of work has no
  value for the key** (or the run has no unit of work in scope) the parameter is
  **omitted** from the request rather than sent blank.
- A single tool may mix `entity:`- and `unitofwork:`-bound parameters; each
  resolves against its own scope.

#### End-to-end example: an authenticated API tool that uses a variable

Putting §5.1–§5.3 together — a complete, authenticated API tool from scratch.

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

## 5A. Connectors

Connectors are named, brain-scoped integrations with an external service (REST
API root or MCP endpoint). Each connector is a single plain-YAML file; the compose
file lists the paths to deploy. Full authoring guide and worked examples:
**BRA209**. Canonical examples live under `connectors/` in this brain.

### 5A.1 Connector file (`connectors/{name}.yml`)

```yaml
name: my-connector                 # REQUIRED — unique per brain
url: https://api.example.com       # XOR with url-env — static HTTPS base URL
# url-env: ACME_API_URL            # XOR with url — brain env var for the base URL
auth-type: oauth2                  # REQUIRED — oauth2 | api-key | none
scope: brain                       # optional — defaults to brain (only value today)
parameters:                         # optional — omit the key entirely when empty
  - name: client-id
    description: OAuth 2 client ID
  - name: client-secret
    description: OAuth 2 client secret
```

Rules:

- Plain YAML — **no** markdown frontmatter delimiters (same style as tool files).
- `name` is unique per brain and is the stable path/code (`connectors/{name}.yml`).
- Exactly one of `url` (static HTTPS) or `url-env` (brain environment variable
  name whose value is the base URL — **BRA202** / **BRA209**). HTTP is allowed
  only for `localhost`, `127.0.0.1`, and `host.docker.internal` (**BRA106**).
- `parameters` declare credential **names** only. Secret **values** belong in
  brain environment variables (`.env` / BRA202) — never in the YAML.
- OAuth access/refresh tokens are runtime state (`ConnectorTokens`), not schema.
- Upsert-always on deploy (no `version` field): Name / Url / UrlEnv / AuthType /
  Scope / parameters are replaced on every deploy.
- Register paths under `connectors:` in `brain-compose.yml` (unlisted = not
  deployed).

---

## 6. Skills

Skills live in **skillbooks**. A skillbook manifest declares categories; each
category lists skill markdown files. Each skill's own metadata lives in its
markdown frontmatter.

### 6.1 Skillbook manifest (`skills/<book>/skillbook.yml`)

```yaml
name: Engineering Practices            # REQUIRED (uploaded as the book title)
code: ENG                              # REQUIRED (unique skillbook code)
prefix: EP                             # REQUIRED (skill code prefix, e.g. EP101)
version: 1.0                           # optional (see §9)
description: Core engineering standards and reusable technical practices.  # optional
categories:
  - name: Backend                      # REQUIRED
    description: Server-side design, data access and API practices.  # optional
    index: 100                         # optional ordering; defaults to array position
    skills:
      - backend/EP101-database-migrations.md   # paths relative to this manifest
      - backend/EP102-api-versioning.md
  - name: Frontend
    description: Client-side architecture and UI patterns.
    index: 200
    skills:
      - frontend/EP201-component-design.md
```

### 6.2 Skill file (markdown + frontmatter)

```markdown
---
name: Database Migrations              # REQUIRED (skill title)
code: EP101                            # REQUIRED (unique skill code; conventionally <prefix><n>)
description: How to author safe, reversible database migrations.  # optional
version: 1.0.0                         # optional (see §9)

# Optional: tool names this skill needs when loaded via get_skill (see §6.3).
# Values are tool `name`s from the Tools table (same identifiers workflows use).
tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
  - create_skill
  - create_schema_file
---

# Instructions

1. Keep every migration idempotent and forward-only where possible.
2. …
```

Rules:

- The markdown **body must not be empty** — the body is the skill content.
- `name` and `code` are required in frontmatter.
- Skill `code`s are what workflows reference in `injected-skills` /
  `available-skills`.
- `tools` is optional. Omit it entirely when the skill does not require tools.
  When present, list tool **names** (not paths). Deploy stores them as
  `Skills.ToolCodes`; extract writes the list back (or omits the key when empty).

### 6.3 Skill-declared tools and mid-run promotion

Skills may declare the tools they need. That declaration does **not** grant the
workflow those tools by itself — the workflow still owns the permission
envelope. Promotion only happens when all of the following are true:

1. The skill lists the tool under frontmatter `tools:`.
2. The workflow lists that same tool under `available-tools:` (not only under
   `tools:` — see §8).
3. During a run, the agent calls `get_skill` for that skill.

Matching names are then **promoted** for the remainder of that `WorkflowRun`
only (appended to `WorkflowRuns.PromotedTools`). They appear in the Claude tool
declarations on subsequent turns of the same run. They are never written back to
`WorkflowTools`, and they do not affect other runs.

| Outcome | Behaviour |
| ------- | --------- |
| Tool in skill `tools:` **and** workflow `available-tools:` | Promoted for this run |
| Tool in skill `tools:` but **not** in the workflow available pool | Silently skipped (workflow curation wins) |
| Tool already in workflow `tools:` (injected) | Already declared; promotion is a no-op / deduped |
| System tools | Always available when declared on the workflow; they are not stored in `WorkflowTools` and are not part of promotion |

Discovering available tools at runtime uses `find_available_tools` (semantic
search over the workflow's `available-tools` pool). See the live example in
`WF-SKILL-UPDATE` + skill `BRA203`.

---

## 7. Blueprints

A blueprint is a scoped collection of long-form knowledge. The `blueprint.yml`
manifest declares the **scope** and the **categories**; the blueprint's entries
are the **sibling markdown files** in the same folder, each tagged with a
category in its frontmatter.

### 7.1 Blueprint manifest (`blueprints/<name>/blueprint.yml`)

```yaml
name: Product Brain                    # REQUIRED (blueprint title)
version: 1                             # optional (see §9)
# description: optional

# Scope — either object form or shorthand string form (both accepted):
scope:
  type: brain                          # one of: brain | entity | unitofwork
  # code: application                  # REQUIRED when type is entity/unitofwork
categories:
  - name: Vision                       # REQUIRED
    description: The long-term vision and guiding principles.  # optional
  - name: Architecture
    description: High-level technical architecture and key decisions.
```

Scope shorthand (equivalent to the object form):

```yaml
scope: brain                           # brain-scoped
scope: entity:application              # entity-scoped, code = application
scope: unitofwork:ticket               # unit-of-work-scoped, code = ticket
```

Notes on scope:

- Accepted keywords: `brain`, `entity`, `unitofwork` (case-insensitive).
- `entity` and `unitofwork` **require a code** that matches an entity/unit code
  declared in `brain-compose.yml`.
- Omitting `scope` entirely defaults to `brain`.
- The blueprint's **code is derived from its folder name** (it has no `code`
  field of its own). So `blueprints/product-brain/` → code `product-brain`.

### 7.2 Blueprint entries (markdown + frontmatter)

**Every `.md` file in the blueprint folder is treated as an entry** (sorted
alphabetically). Each must declare a `category` that exists in the manifest.

```markdown
---
name: Vision Overview                  # REQUIRED (entry title)
category: Vision                       # REQUIRED (must match a manifest category, case-insensitive)
version: 1.0.0                         # optional; not currently uploaded per-entry
---

# Vision Overview

We are building an AI brain that captures and applies an organisation's
knowledge consistently…
```

Rules:

- The markdown **body must not be empty**.
- `category` must match one of the manifest's category names (case-insensitive);
  an unknown category is a hard error.
- Because every `.md` in the folder becomes an entry, don't drop unrelated
  markdown into a blueprint folder.

---

## 8. Workflows

A workflow is a **single, self-contained markdown file**. The frontmatter is the
header and the wiring (tools + skills); the markdown body is the instructions.

```markdown
---
name: Review Blueprint                 # REQUIRED (workflow title)
code: WF-REVIEW                        # REQUIRED (unique workflow code)
description: Reviews a blueprint submission and posts findings to the ticket.  # optional
version: 1.1                           # optional (see §9)
type: RUNNABLE                         # optional; one of TOOL | RUNNABLE | TRIGGERED | SYSTEM | SIMULATION | COMPACTION (default RUNNABLE)
# trigger: inbox:SKILL_UPDATE           # optional; TRIGGERED only — scalar or YAML list (BRA121)
# trigger: inbox:SKILL_UPDATE:low      # optional learning-mode qualifier (BRA122): low|medium|high
# trigger:
#   - inbox:SKILL_UPDATE
#   - inbox:WORKFLOW_UPDATE:medium
# model: anthropic/claude-sonnet-4-6   # optional; provider/model (see BRA210)
# model: openai/gpt-4o                 # OpenAI
# model: xai/grok-4.5                  # xAI / Grok

# Injected tools — included in the LLM tool declarations for every turn.
# Referenced by tool NAME (the tool's `name`).
tools:
  - find_available_skills
  - get_skill
  - find_available_tools
  - add_ticket_comment

# Available tools — in the workflow's searchable permission envelope only.
# Surfaced via find_available_tools, or promoted mid-run when get_skill loads a
# skill that lists them under its own tools: field (see §6.3). Not injected by
# default. Omit the key entirely when unused (existing workflows stay valid).
available-tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
  - create_skill
  - create_schema_file

# Pre-called tools (BRA216) — executed automatically at run startup before the
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
  omitted defaults to `RUNNABLE`. `TOOL` = callable by another workflow (e.g. exposed via a `workflow` tool), `RUNNABLE` = executed manually, `TRIGGERED` = fired when `trigger` matches, `SYSTEM` = never invoked directly; referenced by other workflows via `system-prompt-code` to supply the system prompt. `SIMULATION` = tool-response synthesis for simulation interception; the ToolRouter resolves the active workflow of this type (not by code) when intercepting Api/Mcp tools on a simulation run. `COMPACTION` = client-side context summariser for auto-compaction (Chat Completions providers) and the `compact_context` system tool on every provider (BRA263); at most one active per brain.
- Well-known `trigger` values include `inbox:<RoutingType>` / `inbox:*` (inbox
  learning loop), `unitofwork:complete` (unit-of-work learning eval), and
  `workflowrun:complete` (workflow-run learning eval, BRA091).
- **Inbox triggers** (two stages — full rules in **BRA404**):
  - **Entry create:** `inbox:<RoutingType>` or `inbox:*` (optional
    `:low|medium|high` learning-mode qualifier, BRA122) selects which
    `TRIGGERED` workflows get a `PENDING` inbox task when an entry is created.
    Qualifiers use `off < low < medium < high` (brain mode must meet or exceed
    the qualifier; unqualified inbox triggers always fire).
  - **Task auto-run:** once a task exists, the **workflow linked on that task**
    is authoritative. If that workflow has an inbox trigger whose learning-mode
    qualifier is satisfied, the task auto-runs (`PENDING → RUNNING`); otherwise
    it moves to `AWAITING_APPROVAL`. Entry routing is not re-checked at this
    stage. `trigger-mode` does **not** control inbox task approval.
- For `workflowrun:complete`, `trigger-mode: automatic` enqueues on Completed;
  `trigger-mode: manual` (or omitted) is kicked off from the admin **Run eval**
  button. Use `{{run.telemetry}}` (BRA204) for OTEL GenAI telemetry of the
  subject run. Full authoring guide (manual vs automatic, tools, prompts):
  **BRA207**. Canonical example: `workflows/WF-EVAL-RUN.md`.
- `tools` (injected) and `available-tools` (searchable / promotable) reference
  tools by their `name`. `injected-skills` / `available-skills` reference skills
  by their `code`. These lists accept a single string or a list. Make sure the
  referenced names/codes exist.
- A workflow with only `tools` and no `available-tools` is unchanged — every
  listed tool is treated as injected.
- Deploy writes `tools` → `WorkflowTools.AvailabilityType = injected` and
  `available-tools` → `available`. Extract splits the join rows back into the
  two lists.
- `input-tools` (optional, BRA216) declares tools to call automatically at run
  startup. See **§8.0a**.

### 8.0a Pre-called tools (`input-tools`, BRA216)

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
| `tool` | yes | Tool Router name (`Tools.Name` or system tool name) |
| `parameters` | no | String map of argument values; supports `{{…}}` tags |

**Runtime behaviour**

1. After BRA215 `variables` are available as `{{input.*}}`, each entry runs in
   declaration order.
2. Parameter values are rendered via the Template Service, then dispatched
   through `IToolRouter.RouteAsync`.
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

### 8.0b Choosing a model (`model`, see **BRA210**)

Set `model` to a `provider/model-name` string (e.g. `anthropic/claude-sonnet-4-6`,
`openai/gpt-4o`, `xai/grok-4.5`). Supported providers, example model codes, and
credential mapping are listed in **BRA210**. Bare model names (no prefix)
default to Anthropic. Omit `model` for the platform default
(`anthropic` / `claude-sonnet-4-5`).

### 8.1 LLM execution settings (optional)

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
reached (BRA263).

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

### 8.2 Chat session settings (optional)

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

---

## 9. Versioning rules (important)

The Management API versions every resource with a **single integer** and enforces
precedence: an incoming version must be **greater than or equal to** the stored
one, otherwise the upload is a **VersionConflict** and is skipped (not failed).
The same rule applies to CLI redeploy, the per-type upload endpoints, and
update-from-template (**BRA206**).

The file format is friendly about how you express versions; the CLI normalises
them all to the **leading integer** (the "major"):

| You write     | Deployed as |
| ------------- | ----------- |
| `1`           | `1`         |
| `1.0`         | `1`         |
| `1.2.3`       | `1`         |
| *(omitted)*   | `1`         |
| `2.5`         | `2`         |

Implication: **equal majors redeploy and overwrite**; only a lower incoming major
is skipped. Bump the leading integer (e.g. `1.x` → `2`) when you want a clear
newer release marker, or when destination already holds a higher version.

---

## 10. Field reference cheatsheet

Required fields, by file type (everything else is optional):

| File                     | Required fields                                                        |
| ------------------------ | --------------------------------------------------------------------- |
| `brain-compose.yml`      | `name`                                                                 |
| — entity                 | `name`, `code`; each `variables` item needs `key`                      |
| Connector YAML           | `name`, `url`, `auth-type` (`oauth2` \| `api-key` \| `none`); each parameter needs `name` + `description` |
| Tool group `tools.yml`   | `name`, `description`                                                  |
| Tool definition          | `name`, `description`, exactly one of `api`/`mcp`/`system`/`workflow`/`native` |
| — `api` block            | `path`                                                                 |
| — `mcp` block            | `server`, `tool`                                                       |
| — `system` block         | `tool`                                                                 |
| — `workflow` block       | `code`                                                                 |
| — `native` block         | `type` (`web_search` \| `web_fetch`)                                   |
| Tool parameter           | `name`, `description` (not used by `native` tools)                     |
| `skillbook.yml`          | `name`, `code`, `prefix`; each category needs `name`                   |
| Skill markdown           | frontmatter `name`, `code` + non-empty body; optional `tools` (tool names) |
| `blueprint.yml`          | `name`; each category needs `name`; `scope` code if entity/unitofwork |
| Blueprint entry markdown | frontmatter `name`, `category` + non-empty body                        |
| Workflow markdown        | frontmatter `name`, `code` + non-empty body; optional `tools` / `available-tools` / `input-tools` |

---

## 11. Authoring checklist

When creating or extending a brain:

1. Add/confirm entities and units of work in `brain-compose.yml`.
2. For each connector, create `connectors/{name}.yml` **and** list it under
   `connectors:` (see §5A / **BRA209**). Put secret values in `.env`, not YAML.
3. For each other capability, create its self-contained folder/file **and** add
   its path to the matching list in `brain-compose.yml` (unlisted files are
   ignored).
4. Keep relative paths correct — they resolve against the manifest's own folder.
5. Ensure cross-references resolve: workflow `tools` / `available-tools` → tool
   `name`s; skill `tools` → tool `name`s that the hosting workflow also lists
   under `available-tools` when you want mid-run promotion; workflow
   `injected-skills`/`available-skills` → skill `code`s; blueprint entry
   `category` → a manifest category; scope `code` → an entity/unit code.
6. Give every long-form markdown file a **non-empty body**.
7. Bump the **leading integer** of `version` on anything you change (connectors
   have no version — they upsert-always).
8. Use British English spelling in content.
9. Validate with `npm run deploy:dry` before deploying — it parses and validates
   the entire brain without touching the API.