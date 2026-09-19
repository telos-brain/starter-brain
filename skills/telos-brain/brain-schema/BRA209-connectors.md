---
name: Connectors
code: BRA209
version: 21
description: "How to author connector YAML files for external services (OAuth 2,
  API key, none, or caller-jwt). Covers file layout, brain-compose registration, optional
  platform type (e.g. elevenlabs), parameter declarations vs secret storage,
  url vs url-env, parameter secret: bindings, request-defaults for shared
  headers and common params, parameter as:/in:/value: for any OAuth names,
  oauth-request for non-standard flows, oauth-captures, the production OAuth
  redirect URI https://go.telosbrain.com/oauth/callback, deploy behaviour, and
  worked examples."
tools:
  - list_schema_files
  - search_schema_files
  - get_schema_file
  - update_schema_file
---

# Connectors

A **connector** is a named, brain-scoped definition of an external service the
brain can authenticate to and call — a REST API base URL or an MCP server
endpoint. Connectors are configuration-as-code files under `connectors/`, listed
from `brain-compose.yml`, and deployed with `brain deploy`.

This skill is the authoring guide. The full schema reference also lives in
**BRA209**. Secret storage and `.env` upload are in **BRA202**. Runtime
schema edit tools are in **BRA203**.

---

## 1. What a connector is (and is not)

| Piece | Where it lives | Role |
|---|---|---|
| **Connector YAML** | `connectors/{name}.yml` | Name, URL, auth type, scope, **declared** auth parameters |
| **Parameter values** (client id, API key, …) | Brain environment variables (from `.env` / Management API) | Encrypted secrets — **never** in the YAML |
| **OAuth access / refresh tokens** | Runtime (OAuth flow) | Acquired via OAuth; not authored in the schema |

Connectors are **configuration**. Tokens are **runtime state**. Mixing them
(e.g. stuffing tokens into environment variables as free-form keys) is
rejected by design — keep tokens in the OAuth flow, not in schema files.

---

## 2. Directory layout and compose registration

```
brain-schema/
  brain-compose.yml
  connectors/
    example-oauth2.yml
    example-api-key.yml
    example-none.yml
    example-caller-jwt.yml
  tools/
  …
```

Register every connector you want deployed in `brain-compose.yml`. Unlisted
files are **not** deployed, even if they exist on disk (same rule as tools /
skills / workflows):

```yaml
connectors:
  - connectors/example-oauth2.yml
  - connectors/example-api-key.yml
  - connectors/example-none.yml
  - connectors/example-caller-jwt.yml
```

Paths are relative to the compose file (no `./` prefix needed).

Deploy order places **connectors before tools**, so tools can reference
connector names via a top-level `connector:` field. Connectors are
optional — omit the key when unused.

---

## 3. File format

Plain YAML (no markdown frontmatter). One connector per file. Path convention:
`connectors/{name}.yml` where `{name}` matches the `name` field.

```yaml
name: my-connector                 # REQUIRED — unique per brain; used as the deploy key
url: https://api.example.com       # XOR with url-env — static HTTPS base URL
# url-env: ACME_API_URL            # XOR with url — brain env var name for the base URL
auth-type: oauth2                  # REQUIRED — oauth2 | api-key | none | caller-jwt
# type: elevenlabs                 # optional — platform identity (see Type below)
scope: brain                       # optional — defaults to brain; only brain is valid today
parameters:                         # optional — omit the key entirely when empty
  - name: client-id                # REQUIRED per parameter
    description: OAuth 2 client ID # REQUIRED per parameter
  - name: client-secret
    description: OAuth 2 client secret
```

### Field rules

| Field | Required | Notes |
|---|---|---|
| `name` | yes | Unique per brain. Stable identifier for schema paths and APIs. |
| `url` | one of | Static HTTPS base URL (REST root or MCP endpoint). HTTP allowed only for localhost / `127.0.0.1` / `host.docker.internal` (**BRA106**). **Exactly one** of `url` or `url-env`. |
| `url-env` | one of | Name of a brain environment variable (from `.env`) whose value is the base URL (HTTPS, or HTTP for those local hosts). Resolved at tool/OAuth dispatch — same schema, different domains per brain. |
| `auth-type` | yes | Exactly one of: `oauth2`, `api-key`, `none`, `caller-jwt`. |
| `type` | no | Optional platform identity. Free text; omit when unused. First convention value: `elevenlabs`. Distinct from `auth-type`. |
| `scope` | no | Defaults to `brain`. Entity-scoped connectors are out of scope for now. |
| `api-key-header` | no | For `api-key` auth only. Header name for the key. Omit or blank → `Authorization: Bearer {key}`. Example: `X-Api-Key`. |
| `parameters` | no | List of `{ name, description, secret?, as?, in?, value? }`. You may declare **any** parameter names, not only `client-id` / `client-secret`. `secret:` names a brain environment variable. `as:` is the provider-facing OAuth name (`clientId`, `redirectUri`). `in:` is `authorize`, `token`, or `both`. `value:` is a static non-secret. For `api-key` auth `secret:` binds the API key (omit → `CONNECTOR_{connectorId}_CLIENT_SECRET`). For `oauth2` auth `secret:` on `client-id` / `client-secret` binds `.env` names (omit → `CONNECTOR_{connectorId}_CLIENT_ID` / `_CLIENT_SECRET`). Omit the `parameters` key when there are none — do **not** emit `parameters: []`. |
| `oauth-request` | no | Optional OAuth request customisation for non-RFC providers. See **OAuth request customisation** below. |
| `request-defaults` | no | Shared headers and query/body values applied to **every** tool call on this connector. Same placement as a tool parameter (`header:`, `secret:`, `value:`). A tool-level declaration of the same header or key wins. See **Request defaults** below. |

### Auth types

| `auth-type` | When to use |
|---|---|
| `oauth2` | Interactive OAuth 2 — access/refresh tokens managed at runtime |
| `api-key` | Static API key (or similar) supplied as a secret |
| `none` | Public endpoint; no credentials |
| `caller-jwt` | Forwards the Execute API `caller_jwt` as `Authorization: Bearer` on outbound tool calls. No stored credentials — omit `parameters`. Brain does not validate the JWT. |

### Type (platform identity)

`type` is an **optional** discriminator for platform-specific services. Omit
the key when the connector is a generic REST/MCP endpoint. The first convention
value is:

| `type` | Used for |
|---|---|
| `elevenlabs` | ElevenLabs Conversational AI. Pair with a workflow that sets `deployment-type: elevenlabs_conversational_ai` (**BRA217**). Bind the `xi-api-key` with `secret:` on the `api-key` parameter (or omit `secret:` to use the connector's default client-secret variable). |

A brain should declare **at most one** connector of each platform type. The
deployment handler picks the first by name and logs a warning if several match.

This is **not** the same field as a workflow's `type` (`TOOL` / `RUNNABLE` / …)
or a tool's execution block.

### Parameters vs secrets

- **`parameters`** declare *what* credentials the connector needs (metadata for
  UI, deploy, and future OAuth wiring).
- **Values** are stored as brain environment variables (encrypted at rest) —
  upload them via `.env` on deploy (**BRA202**) or the Management API secrets
  endpoint. For **api-key** auth, bind the key with `secret:` on the `api-key`
  parameter (same field as tools). When `secret:` is omitted the platform
  reads `CONNECTOR_{connectorId}_CLIENT_SECRET`.
- For **oauth2** auth, bind client credentials the same way: `secret:` on
  `client-id` and `client-secret` names the `.env` variables (e.g.
  `EXAMPLE_CLIENT_ID` / `EXAMPLE_CLIENT_SECRET`). Connect and token refresh read
  those names. When `secret:` is omitted they fall back to
  `CONNECTOR_{connectorId}_CLIENT_ID` / `_CLIENT_SECRET` (also used by
  dynamic client registration). OAuth **access / refresh tokens** remain
  runtime state — do not put bearer tokens in `.env`.
- Never put client secrets, API keys, or tokens in the connector YAML.

### OAuth 2 redirect URI (register this with the provider)

The Connect button in the admin UI starts a full-page redirect to the
provider. After consent the provider must send the browser back to the
Management API callback — **not** a SPA-only path.

The URI is `{ManagementApi:BaseUrl}/oauth/callback` (exact match; no trailing
slash). Production Telos Brain:

```
https://go.telosbrain.com/oauth/callback
```

Register that value as the **Web application** redirect URI in the provider
console before clicking Connect. Local development uses the
same path on the configured Management API origin (typically the Vite proxy,
e.g. `http://localhost:50406/oauth/callback`).

### OAuth captures (extra values after Connect)

Some providers return more than an access token — a tenant id, instance URL,
account id, and so on. Declare those on the connector as `oauth-captures`.
After the token exchange each capture is:

1. Read from the **token JSON**, or from an optional follow-up HTTP request
   (Bearer access token; `path` is relative to the connector `url`)
2. Extracted with `json-path` (`tenantId`, `$.foo.bar`, `$[0].tenantId`)
3. Stored **with the token** (same place as the access/refresh token)
4. Optionally copied to a **brain environment variable** (`store: brain:NAME`
   or `store: NAME`) or an **entity variable** (`store: entity:key`)
5. Optionally sent on later connector API calls as `header:`

Entity stores need an entity id on Connect (`POST …/oauth/initiate?entityId=`).
The admin Connectors page is brain-scoped, so use `brain:` there.

When the extra value is not on the token, add a follow-up request (path
relative to the connector `url`) and optionally send it as a header on later
API calls:

```yaml
oauth-captures:
  - name: tenant-id
    json-path: $[0].tenantId
    request:
      method: GET
      path: /connections
    store: brain:EXAMPLE_TENANT_ID
    header: x-tenant-id
```

A value that is already on the token JSON omits `request:`:

```yaml
oauth-captures:
  - name: instance-url
    json-path: instance_url
    store: brain:EXAMPLE_INSTANCE_URL
```

### Request defaults (shared headers and params)

Declare headers and common parameters **once on the connector** instead of
repeating them on every tool. Placement matches **BRA214**: `header:` sends
an HTTP header; without it the value is a GET query parameter or POST/PUT
JSON field. `secret:` reads a brain environment variable; `value:` is a
static string (`{secret}` works as a template when both are set).

```yaml
request-defaults:
  - name: accept
    header: Accept
    value: application/json
  - name: tenant-id
    header: X-Tenant-Id
    secret: EXAMPLE_TENANT_ID
  - name: summary-only
    value: "true"
```

- Applied to every `api:` tool (and as headers on `mcp:` tools) that uses
  this connector.
- A tool parameter with the same header name or payload key **overrides** the
  default.
- One source per header: if an `oauth-captures` entry also sets `header:`
  for the same name, the capture owns that header. The request-default is
  ignored (including a `.env` `secret:`). Prefer captures for values
  collected at Connect; prefer request-defaults for static or `.env` values.
- A header `secret:` that is not set fails the tool **before** the HTTP
  call, naming the header and variable. A capture `header:` with no stored
  value fails the same way and tells the caller to reconnect (follow-up
  captures are retried on the next tool call first).
- If Connect ran before captures were declared, the next tool call re-runs
  follow-up capture requests and stores the values on the token.

### OAuth parameter names (`as:`) — any params you want

Known roles (`client-id`, `client-secret`, `redirect-uri`, `grant-type`,
`refresh-token`, `code`) are recognised from the parameter `name` (kebab,
snake, or concatenated). The platform fills those values. **`as:`** is the
name sent on the wire — use it when the provider rejects RFC names
(`client_id` / `redirect_uri`) and wants camelCase (`clientId` /
`redirectUri`).

Any other parameter is an extra. Give it a `secret:` or a static `value:`,
and optionally `in:` (`authorize`, `token`, or `both`; extras default to
authorise). Extra params are appended to the authorise query and/or token
body using `as:` when set, otherwise the declared `name`.

```yaml
parameters:
  - name: client-id
    as: clientId
    secret: EXAMPLE_CLIENT_ID
  - name: client-secret
    secret: EXAMPLE_CLIENT_SECRET
  - name: redirect-uri
    as: redirectUri
  - name: grant-type
    as: grantType
  - name: audience
    as: audience
    in: authorize
    value: books
```

The Management API exposes the same fields as `asName`, `sendIn`, and
`value` on each connector parameter.

### OAuth request customisation (`oauth-request`)

Standard Connect still redirects the browser to `authorization-url` with
RFC query names and POSTs `application/x-www-form-urlencoded` to
`token-url`. Providers that do not speak that dialect declare:

```yaml
oauth-request:
  authorize-mode: json-redirect   # GET authorization-url, follow JSON URL
  authorize-redirect-path: redirectUri
  token-format: json              # POST JSON instead of form
  signing: hmac-sha256            # x-client-id, x-timestamp, x-signature
  hmac-mount: /v1                 # stripped from the path before signing
  refresh-url: https://api.example.com/oauth/refresh
```

| Field | Default | Meaning |
|---|---|---|
| `authorize-mode` | `redirect` | `json-redirect` GETs `authorization-url` (with remapped query params) and sends the browser to the URL in the JSON body (or a `3xx Location`). If the response is 2xx/3xx without a URL (for example an HTML consent page), the initiate URL itself is the browser destination. |
| `authorize-redirect-path` | `redirectUri` then `redirect_uri` / `url` / `consentUrl` | JSON path of the consent URL. |
| `token-format` | `form` | `json` posts a JSON object. Field names come from parameter `as:`. |
| `signing` | `none` | `hmac-sha256` signs token, refresh, capture, and tool calls. Base string is `METHOD:path:timestamp:bodyHash` (SHA-256 hex of the body for POST/PUT/PATCH). |
| `hmac-mount` | (none) | Prefix removed from the path before signing (`/v1/oauth/token` → `/oauth/token`). |
| `refresh-url` | `token-url` | Use when refresh is a different endpoint. |
| `token-access-path` | RFC `access_token` plus aliases (`accessToken`, `token`, `jwt`) | JSON path only when the token is not a recognised field. Leave unset for camelCase `accessToken` at the root or under `data` / `result` / `payload`. |
| `token-refresh-path` | RFC `refresh_token` plus aliases | JSON path of the refresh token. |
| `token-expires-path` | RFC `expires_in` plus aliases | JSON path of expiry (seconds, unix time, or timestamp). |

`json-redirect` omits `response_type`, PKCE, and `resource` — those extra
RFC params are what non-standard initiate endpoints reject. Token HMAC
sends only `code` / `refresh-token` / `grant-type` (plus extras with
`in: token`) — client credentials go in the HMAC headers, not the body.

Token JSON is read using RFC names first (`access_token`, `refresh_token`,
`expires_in`), then common aliases (`accessToken`, `token`, `jwt`, expiry
timestamps) with case-insensitive matching. Nested envelopes (`data`,
`result`, `payload`, `tokens`, …) are searched recursively. If a token
field is an object, the parser reads `token` / `jwt` / `value` inside it.
A string on `data` / `result` / `payload` itself is also accepted.
Form-encoded token bodies are accepted. If the provider uses a unique
layout, set `token-access-path` / `token-refresh-path` /
`token-expires-path`. A failed parse reports the provider `error` /
`message` or the response keys — never the token values.

---

## 4. Worked examples

Canonical examples ship in this brain under `connectors/`:

### 4.1 OAuth 2 — `connectors/example-oauth2.yml`

```yaml
name: example-oauth2
url: https://api.example.com
auth-type: oauth2
scope: brain
authorization-url: https://auth.example.com/oauth/authorize
token-url: https://auth.example.com/oauth/token
oauth-scope: read:example
parameters:
  - name: client-id
    description: OAuth 2 client ID issued by the external provider.
    secret: EXAMPLE_CLIENT_ID
  - name: client-secret
    description: OAuth 2 client secret. Store the value as a brain environment variable — never commit it here.
    secret: EXAMPLE_CLIENT_SECRET
```

```bash
# .env — uploaded on deploy (BRA202)
EXAMPLE_CLIENT_ID=…
EXAMPLE_CLIENT_SECRET=…
```

### 4.2 API key — `connectors/example-api-key.yml`

```yaml
name: example-api-key
url: https://api.example.com/v1
auth-type: api-key
scope: brain
api-key-header: X-Api-Key
parameters:
  - name: api-key
    description: API key for authenticating outbound calls to this service.
    secret: ACME_API_KEY
```

### 4.3 No auth — `connectors/example-none.yml`

```yaml
name: example-none
url: https://httpbin.org
auth-type: none
scope: brain
```

No `parameters` key — empty lists are omitted.

### 4.4 Caller JWT — `connectors/example-caller-jwt.yml`

```yaml
name: example-caller-jwt
url: https://api.example.com
auth-type: caller-jwt
scope: brain
```

No stored credentials. The harness passes `caller_jwt` on `POST /workflows/{code}/run/sync` or `/run/async`. Brain injects it as `Authorization: Bearer` at dispatch. If the JWT is missing, the tool call fails with a clear error.

**Async expiry caveat:** Clerk JWTs typically expire in ~60 seconds. On `/run/async`,
tools may dispatch after the JWT has expired; the downstream API will return 401,
which Brain surfaces as a tool error. Use `caller-jwt` on async runs only when the
token lifetime covers expected dispatch delay, or when the downstream service
accepts longer-lived tokens.

### 4.5 Environment-specific base URL — `url-env`

When the same schema must hit different hosts in test vs production, declare
`url-env` instead of a literal `url`, and put the URL value in each brain's
`.env` (uploaded on deploy — **BRA202**):

```yaml
# connectors/acme.yml
name: acme
url-env: ACME_API_URL
auth-type: api-key
scope: brain
api-key-header: X-Api-Key
parameters:
  - name: api-key
    description: API key for Acme.
    secret: ACME_API_KEY
```

```bash
# .env on the test brain
ACME_API_URL=https://api.test.example.com

# .env on the production brain
ACME_API_URL=https://api.example.com
```

At dispatch the platform resolves `ACME_API_URL`, validates it as an HTTPS URL
(HTTP is allowed only for `localhost`, `127.0.0.1`, and `host.docker.internal`
— see **BRA106**), then combines it with the tool's relative `api.path`.
Missing or invalid values fail the tool call with a clear error. Do **not**
set both `url` and `url-env`.

When the Brain runs in Docker and the API is on the developer machine, put
`http://host.docker.internal:<PORT>` in `.env.local`. `localhost` inside the
container is the Brain container, not the host. Full local-stack instructions
are in **BRA106**.

### 4.6 ElevenLabs platform connector

Use `type: elevenlabs` so workflow deployment can find this connector. Auth is
normally `api-key`. Name the `.env` variable with `secret:` on the `api-key`
parameter (same field as tools). When `secret:` is omitted the handler falls
back to `CONNECTOR_{connectorId}_CLIENT_SECRET`. Never put the key in the YAML.
The handler calls `https://api.elevenlabs.io` and sets `xi-api-key` on the
request.

```yaml
name: elevenlabs
url: https://api.elevenlabs.io
auth-type: api-key
type: elevenlabs
scope: brain
parameters:
  - name: api-key
    description: ElevenLabs xi-api-key. Store the value as a brain environment variable — never commit it here.
    secret: ELEVENLABS_API_KEY
```

```bash
# .env — uploaded on deploy (BRA202)
ELEVENLABS_API_KEY=xi-...
```

Workflows that should be projected as ElevenLabs agents also need
`deployment-type: elevenlabs_conversational_ai` — see **BRA217**.

### 4.7 Non-standard OAuth (custom names, JSON token, HMAC)

Use this when the provider is not RFC OAuth 2: camelCase query names, a
JSON token body, HMAC request signing, or an initiate URL that is itself
the browser destination. Register
`https://go.telosbrain.com/oauth/callback` (or
`{ManagementApi:BaseUrl}/oauth/callback` locally) with the provider.

```yaml
name: example-partner
url: https://api.example.com
auth-type: oauth2
scope: brain
authorization-url: https://api.example.com/oauth/initiate
token-url: https://api.example.com/oauth/token
oauth-request:
  authorize-mode: json-redirect
  authorize-redirect-path: redirectUri
  token-format: json
  signing: hmac-sha256
  hmac-mount: /v1
  refresh-url: https://api.example.com/oauth/refresh
parameters:
  - name: client-id
    description: OAuth client ID issued by the provider.
    secret: EXAMPLE_CLIENT_ID
    as: clientId
  - name: client-secret
    description: OAuth client secret. Store in .env — never commit the value here.
    secret: EXAMPLE_CLIENT_SECRET
  - name: redirect-uri
    description: Registered OAuth callback. Brain fills this with the platform redirect URI.
    as: redirectUri
  - name: grant-type
    description: Token grant type field name.
    as: grantType
  - name: refresh-token
    description: Refresh token field name on token refresh.
    as: refreshToken
oauth-captures:
  - name: workspace-id
    json-path: workspaceId
    store: brain:EXAMPLE_WORKSPACE_ID
```

```bash
# .env — uploaded on deploy (BRA202)
EXAMPLE_CLIENT_ID=…
EXAMPLE_CLIENT_SECRET=…
```

### 4.8 Referencing a connector from a tool

API and MCP tools may omit a connector (inline URL / server) or point at one:

```yaml
name: list_widgets
version: 1
description: Lists widgets from the Acme API.
api:
  method: GET
  path: /v1/widgets          # optional relative path on the connector base URL
  connector: example-api-key # optional — Connectors.Name for this brain
```

Write `{parameter-name}` in `path` to put a parameter in the URL
(`path: /v5/entities/{nzbn}`). See **BRA214**.

```yaml
name: search_documents
version: 1
description: Search documents via MCP.
mcp:
  tool: search
  connector: example-none    # optional — or use server-url / server instead
```

At API dispatch the Tool Router resolves the connector URL (+ optional path),
validates it with SSRF guards, and injects that connector's `auth-type`
(OAuth2 Bearer, API key, or caller JWT). Shared extra headers and params
come from `request-defaults` and from `oauth-captures` → `header:`. A tool
may still declare its own `header:` / `secret:` (**BRA214**); those override
a default with the same name. A brain `.env` value is unused until a tool
`secret:`, a `request-defaults` `secret:`, or a capture `store:` names it.

Tools without `connector:` keep their inline URL / server behaviour. Header,
query, body, and path placement for tool parameters is **BRA214**.

---

## 5. Deploy and upsert semantics

On `brain deploy`:

1. The CLI uploads schema files (including `connectors/*.yml` when listed).
2. The server parses the compose `connectors:` list and upserts each connector
   by `(BrainId, Name)`.
3. Declared parameters are **replaced wholesale** for that connector.
4. There is **no version field** — a redeploy always applies the incoming
   Name / Url / UrlEnv / AuthType / Type / Scope / ApiKeyHeader /
   parameters (including each parameter's `secret:`) (upsert-always, like entity types).

Validate with `npm run deploy:dry` before deploying.

---

## 6. Editing connectors at runtime

Schema system tools (**BRA203**) treat connectors as first-class schema files:

| Operation | Support |
|---|---|
| `list_schema_files` / `search_schema_files` | Yes — type token `connector`, path `connectors/{name}.yml` |
| `get_schema_file` | Yes — returns canonical YAML |
| `update_schema_file` | Yes — exact-one-match string replace, then the connector definition is saved |
| `create_schema_file` | **Not yet** for `connectors/…` — create via deploy (add file + compose entry) |

`brain-compose.yml` remains read-only via `update_schema_file`.

---

## 7. Authoring checklist

1. Create `connectors/{name}.yml` with `name`, exactly one of `url` / `url-env`,
   and `auth-type`. Set `type` when the connector backs a platform (e.g.
   `elevenlabs`).
2. Add declared `parameters` (`name` + `description`; optional `secret:`,
   `as:`, `in:`, `value:`) when the connector needs credentials or extra
   OAuth fields; omit the key when it does not. Use `oauth-request` when
   the provider is not standard OAuth 2.
3. List the path under `connectors:` in `brain-compose.yml`.
4. Put secret **values**, any `url-env` URL values, and any parameter `secret:`
   values in `.env` (or the secrets API) — never in the YAML.
5. Use British English in descriptions.
6. Dry-run deploy, then deploy.

---

## Related skills

| Skill | Topic |
|---|---|
| **BRA201** | Schema overview (which skill to load for each file type) |
| **BRA217** | Workflow `deployment-type` / `elevenlabs-agent-id` |
| **BRA202** | Environment variables, encryption, secret injection into tools |
| **BRA203** | Schema system tools (list / get / update connector files) |
