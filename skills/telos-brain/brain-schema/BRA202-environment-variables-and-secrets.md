---
name: Environment Variables, Secrets & API Keys
code: BRA202
version: 9
description: How a brain's .env variables are uploaded, encrypted and stored; the
  well-known "system" keys the platform recognises (LLM provider keys, the brain
  API key); how to inject a stored secret into an api tool's outbound request —
  as an HTTP header, a query parameter or a JSON body field; how connector
  credential values relate to the same store; and how connector url-env resolves
  a base URL from this store.
---

# Environment Variables, Secrets & API Keys

A brain often needs secrets to do its work: an LLM provider key to run its
workflows, and any number of third-party API keys for the tools it calls. Telos
Brain stores these as **brain environment variables** — encrypted at rest and
scoped to a single brain — and lets a tool inject one into its outbound request
without the secret ever living in the schema itself.

This document explains how variables flow from your `.env` to the brain, which
variable names the platform treats specially, and how to wire a stored secret
into an API tool.

---

## 1. How `.env` variables reach the brain

Every brain schema carries a `.env` file next to its `brain-compose.yml`. On
`brain deploy`, the CLI loads that file and, immediately after ensuring the brain
exists (and **before** the schema is deployed), uploads its variables to the
Management API, which **encrypts each value at rest** against the brain.

The rule is simple:

- **Every variable declared in `.env` is uploaded** as a brain environment
  variable — LLM provider keys, third-party API keys, anything.
- **Except CLI/deploy configuration**, which is everything prefixed `TELOS_`
  (e.g. `TELOS_BRAIN_ORG_API_KEY`, `TELOS_BRAIN_API_URL`). The `TELOS_` prefix is
  reserved for the CLI; those values are consumed locally to talk to the
  Management API and are **never** uploaded to the brain.
- A variable with a **blank value is skipped** (you can't store an empty secret).
- A real environment variable always **overrides** the checked-out `.env` value,
  so CI secrets win over a committed file. Only the variables **declared in the
  schema's own `.env`** are uploaded — never your whole shell environment.

Storage semantics: variables are keyed by name per brain and **upserted**
(replaced, not versioned). Re-deploying with a new value overwrites the old one.

```
.env  ──brain deploy──▶  POST /brains/{instance}/environment-variables
                         (one upsert per variable, value encrypted at rest)
```

When cloning a brain (`POST /brains/{instance}/clone`), source environment
variables are copied with the clone and optional plaintext overrides in the
request body are encrypted and merged over them — see **BRA205**.

Updating a brain from a template (`POST /brains/{target}/update-from/{source}`)
does **not** touch destination environment variables — secrets stay as they are
on the target. Manage them separately via deploy or the environment-variables
endpoint. See **BRA206**.

> Security: never commit a real `.env`. Keep it in `.gitignore` (alongside
> `node_modules/` and `brain.lock`) and provide `.env.example` as the template.

---

## 2. The "system" keys (well-known variable names)

Most variables are opaque to the platform — it stores them and hands them back
only when a tool asks for one by name (§3). A few names are **recognised by
convention** and used automatically:

| Variable                | Prefix scope | Meaning                                                                                       |
| ----------------------- | ------------ | --------------------------------------------------------------------------------------------- |
| `ANTHROPIC_API_KEY`     | uploaded     | LLM provider key for **Anthropic / Claude**. Resolved automatically for any run whose model is `anthropic/…` (or unprefixed — Anthropic is the default provider). |
| `OPENAI_API_KEY`        | uploaded     | LLM provider key for **OpenAI**. Resolved for runs whose model is `openai/…`.                 |
| `XAI_API_KEY`           | uploaded     | LLM provider key for **xAI / Grok**. Resolved for runs whose model is `xai/…` (BRA240).       |
| `TIMEZONE`              | uploaded     | Optional IANA timezone id (e.g. `Pacific/Auckland`) used by `{{now.local*}}` template tags. When unset or unrecognised, local time falls back to UTC. |
| `TELOS_BRAIN_ORG_API_KEY` | **local**  | Organisation deploy credential the CLI authenticates with. Never uploaded to the brain. Legacy: `TELOS_ORG_API_KEY`. |
| `TELOS_BRAIN_API_URL`   | **local**    | Management API base URL (deploy destination) for the CLI. Never uploaded to the brain. Legacy: `TELOS_API_URL`. |
| `TELOS_BRAIN_TOKEN`     | **local**    | Optional Clerk bearer token for the CLI. Never uploaded to the brain. Legacy: `TELOS_TOKEN`. |

### LLM provider keys use a standard variable name

The model resolver maps a workflow's `model` prefix onto a **standard** variable
name of the form `<PROVIDER>_API_KEY` (upper-case). So:

- `anthropic/claude-sonnet-4-5` → looks up **`ANTHROPIC_API_KEY`**
- `openai/gpt-…` → looks up **`OPENAI_API_KEY`**
- `xai/grok-4.5` → looks up **`XAI_API_KEY`**
- a workflow with **no** `model` (or a bare model name with no provider prefix)
  falls back to the default provider **anthropic**, i.e. still **`ANTHROPIC_API_KEY`**.

Always name the Claude key **`ANTHROPIC_API_KEY`** — that is the one and only
name the platform looks for for Anthropic. Use **`OPENAI_API_KEY`** for OpenAI
and **`XAI_API_KEY`** for Grok. If a workflow's model resolves to a provider
whose `<PROVIDER>_API_KEY` variable is not set for the brain, the run cannot
start.

For the full list of supported providers, example workflow `model` codes, and
which ConversantSettings each provider honours, see **BRA210**.

---

## 3. Using an API key in a tool (secret injection)

A declared **API tool** (`api:` block) can call any HTTP endpoint. When that
endpoint needs authentication, the tool references a **stored variable by name**;
at dispatch the platform decrypts the variable and injects it into the request —
as an HTTP header (the common case, §3.1), or as a query parameter / body field
(§3.3). The secret never appears in the tool definition, the schema, or the
model's view of the tool.

This is expressed with these parameter fields:

- `secret:` — the **name of a brain environment variable** (from `.env`) whose
  decrypted value is injected.
- `header:` — *(optional)* the **HTTP header name** to place it in (e.g.
  `Authorization`). A parameter with `header:` is sent as a request header;
  **without** `header:` the value goes into the request payload instead (§3.3).
- `value:` (optional) — a **template** in which the placeholder `{secret}` is
  replaced by the resolved secret, so you can format things like
  `"Bearer {secret}"`. With `secret:` but **no** `value:`, the raw secret is
  injected as-is.

A parameter that declares `secret:` (or `header:`, or a hardcoded `value:`) is
**hidden from the LLM** — the model neither sees nor supplies it; it is pure
transport configuration.

**Only `api:` tools inject secrets.** `system:`, `workflow:` and `native:` tools
run in-process or on the model and never make an authenticated outbound HTTP
call, so `secret:`/`header:` have no effect there.

**Resolution order.** Each parameter value is resolved in this priority: (1) the
decrypted variable named by `secret:` (substituted into the `{secret}` template
when `value:` is present); else (2) a hardcoded `value:`; else (3) for an exposed
parameter, the argument the model supplied under the parameter's `name`. A
parameter that resolves to nothing is omitted from the request.

### 3.1 Worked example — calling an authenticated API

Store the key in `.env` (it will be uploaded, encrypted):

```bash
# .env  (in the brain schema folder, next to brain-compose.yml)
ACME_API_KEY=sk_live_xxx
```

Reference it from the tool. Here the key is injected as a bearer token:

```yaml
# tools/acme/create-widget.yml
name: create_widget
version: 1
description: Creates a widget in Acme via its HTTP API.
api:
  method: POST
  path: https://api.acme.example.com/widgets
parameters:
  # Injected secret — hidden from the model, sent as an HTTP header.
  - name: authorization
    description: Acme API key, injected as the Authorization bearer token.
    header: Authorization           # send as the "Authorization" header
    secret: ACME_API_KEY            # value comes from the brain env variable
    value: "Bearer {secret}"        # template: {secret} → decrypted value

  # Ordinary, model-supplied body parameters work exactly as before.
  - name: name
    description: The display name of the widget to create.
```

At run time the tool call becomes:

```
POST https://api.acme.example.com/widgets
Authorization: Bearer sk_live_xxx
Content-Type: application/json

{ "name": "<value the model supplied>" }
```

If `ACME_API_KEY` is not set for the brain, the header is **omitted** (logged and
skipped) rather than sent as a placeholder — the call proceeds and typically
fails auth, which surfaces as a clear tool error.

### 3.2 Calling the brain's own Execution API

The same mechanism lets a workflow call **this brain's own** Execution API when
an HTTP round-trip is genuinely required. Prefer an in-process **system tool**
when one exists (e.g. `create_inbox_entry` — see BRA405 / BRA207) so no API key
or host URL is needed.

When you do need HTTP, the brain's API key is issued once at brain-creation
time; store it in `.env` as a normal variable (conventionally `BRAIN_API_KEY`)
and inject it:

```yaml
api:
  method: POST
  path: https://go.telosbrain.com/inbox    # Execution API host the Tool Router can reach
parameters:
  - name: authorization
    description: This brain's Execution API key.
    header: Authorization
    secret: BRAIN_API_KEY
    value: "Bearer {secret}"
```

> Header vs value/query: a parameter with a hardcoded `value:` **and no**
> `secret:`/`header:` is still a fixed body/query parameter hidden from the LLM
> (see BRA201 §5.3). Adding `header:` moves it to a request header; adding
> `secret:` sources its value from a stored variable instead of hardcoding it.

### 3.3 Where the secret goes: header, query or body

`header:` is only one placement. When a parameter has **no** `header:`, its
resolved value joins the request **payload**, and the HTTP method decides how:

- **GET** — payload parameters are appended to the URL as the **query string**
  (`?key=value`, url-encoded).
- **POST / other** — payload parameters are serialised into the **JSON body**.

So an API keyed by a query parameter or a body field (rather than a header)
still injects the secret with `secret:` — just omit `header:`:

```yaml
# API that authenticates with a `?api_key=…` query parameter (GET tool).
api:
  method: GET
  path: https://api.acme.example.com/search
parameters:
  # No `header:` → this goes into the query string as api_key=<decrypted value>.
  - name: api_key
    description: Acme API key, injected as the api_key query parameter.
    secret: ACME_API_KEY
  # Exposed, model-supplied query parameter.
  - name: q
    description: The search query.
```

The `param:` field renames the wire key when it must differ from the AI-facing
`name` (BRA201 §5.3) — e.g. `param: apiKey` sends `apiKey=…` while the parameter
is still authored as `api_key`. Either way the secret parameter stays hidden
from the model.

> If a secret's variable is not set for the brain, the parameter is **skipped**
> (logged), not sent blank — in every placement. The request proceeds and
> typically fails auth, surfacing as a clear tool error rather than a silent
> mis-send.

---

## 4. Connector credentials and base URLs

**Connectors** (BRA209) declare required auth inputs in YAML (`parameters` with
`name` / `description` only). The **values** for those inputs — OAuth client id /
secret, API keys, and similar — are stored as brain environment variables in this
same encrypted store. They are never written into the connector YAML.

- Declare the parameter names on the connector file (schema).
- Put the values in `.env` (or upsert via the Management API secrets endpoint,
  which uses connector-scoped keys such as `CONNECTOR_{connectorId}_CLIENT_ID`).
- OAuth **access / refresh tokens** are runtime state (`ConnectorTokens`), not
  environment variables — do not put bearer tokens in `.env` for that purpose.

A connector may also take its **base URL** from this store via YAML `url-env:`
(instead of a static `url:`). Put the URL in `.env` under that variable name;
at tool/OAuth dispatch the platform resolves it the same way as other brain
environment variables. Values must be HTTPS, except HTTP is allowed for
`localhost`, `127.0.0.1`, and `host.docker.internal` (Brain-in-Docker → host
API — **BRA106**). Use `url-env` when one schema is deployed to local, test,
and production brains with different hosts. See **BRA209** §4.4.

See **BRA209** for the connector file format and examples.

---

## 5. Checklist

1. Put every secret the brain needs in the schema's `.env` (never commit it).
2. Name the Claude key exactly `ANTHROPIC_API_KEY` (and OpenAI `OPENAI_API_KEY`).
3. Keep CLI/deploy config under the `TELOS_` prefix — it stays local.
4. To authenticate an **`api:` tool**, reference the variable with `secret:`;
   add `header:` (plus an optional `value: "Bearer {secret}"` template) to send
   it as a header, or omit `header:` to inject it into the query string (GET) or
   JSON body (POST). Never paste the key into the tool file.
5. For **connectors**, declare parameter names in `connectors/*.yml` and store
   values here — never in the connector file (BRA209). When using `url-env`, put
   the base URL in `.env` under that variable name (HTTPS, or HTTP for local
   harness hosts — **BRA106**).
6. Redeploy to rotate a secret — the value is upserted (replaced) against the
   brain and re-encrypted.
