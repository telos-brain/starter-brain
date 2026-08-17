---
name: Connectors
code: BRA209
version: 4
description: How to author connector YAML files for external services (OAuth 2,
  API key, or none). Covers file layout, brain-compose registration, parameter
  declarations vs secret storage, url vs url-env for environment-specific base
  URLs, deploy behaviour, and worked examples.
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
**BRA201** §5A. Secret storage and `.env` upload are in **BRA202**. Runtime
schema edit tools are in **BRA203**.

---

## 1. What a connector is (and is not)

| Piece | Where it lives | Role |
|---|---|---|
| **Connector YAML** | `connectors/{name}.yml` | Name, URL, auth type, scope, **declared** auth parameters |
| **Parameter values** (client id, API key, …) | Brain environment variables (from `.env` / Management API) | Encrypted secrets — **never** in the YAML |
| **OAuth access / refresh tokens** | `ConnectorTokens` table (runtime) | Acquired via OAuth flow; not authored in the schema |

Connectors are **configuration**. Tokens are **runtime state**. Mixing them
(e.g. stuffing tokens into `BrainEnvironmentVariables` as free-form keys) is
rejected by design — see BRA196 / OAuth lifecycle docs.

---

## 2. Directory layout and compose registration

```
brain-schema/
  brain-compose.yml
  connectors/
    example-oauth2.yml
    example-api-key.yml
    example-none.yml
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
```

Paths are relative to the compose file (no `./` prefix needed).

Deploy order places **connectors before tools**, so tools can reference
connector names via a top-level `connector:` field (BRA199). Connectors are
optional — omit the key when unused.

---

## 3. File format

Plain YAML (no markdown frontmatter). One connector per file. Path convention:
`connectors/{name}.yml` where `{name}` matches the `name` field.

```yaml
name: my-connector                 # REQUIRED — unique per brain; used as the deploy key
url: https://api.example.com       # XOR with url-env — static HTTPS base URL
# url-env: ACME_API_URL            # XOR with url — brain env var name for the base URL
auth-type: oauth2                  # REQUIRED — oauth2 | api-key | none
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
| `name` | yes | Unique per brain (`UQ_Connectors_BrainId_Name`). Stable identifier for schema paths and APIs. |
| `url` | one of | Static HTTPS base URL (REST root or MCP endpoint). HTTP allowed only for localhost / `127.0.0.1` / `host.docker.internal` (**BRA106**). **Exactly one** of `url` or `url-env`. |
| `url-env` | one of | Name of a brain environment variable (from `.env`) whose value is the base URL (HTTPS, or HTTP for those local hosts). Resolved at tool/OAuth dispatch — same schema, different domains per brain. |
| `auth-type` | yes | Exactly one of: `oauth2`, `api-key`, `none`. |
| `scope` | no | Defaults to `brain`. Entity-scoped connectors are out of scope for now. |
| `api-key-header` | no | For `api-key` auth only (BRA199). Header name for the key. Omit or blank → `Authorization: Bearer {key}`. Example: `X-Api-Key`. |
| `parameters` | no | List of `{ name, description }`. Omit the key when there are none — do **not** emit `parameters: []`. |

### Auth types

| `auth-type` | When to use |
|---|---|
| `oauth2` | Interactive OAuth 2 — access/refresh tokens managed at runtime |
| `api-key` | Static API key (or similar) supplied as a secret |
| `none` | Public endpoint; no credentials |

### Parameters vs secrets

- **`parameters`** declare *what* credentials the connector needs (metadata for
  UI, deploy, and future OAuth wiring).
- **Values** are stored as brain environment variables (encrypted at rest) —
  upload them via `.env` on deploy (**BRA202**) or the Management API secrets
  endpoint (connector-scoped keys such as `CONNECTOR_{connectorId}_CLIENT_ID`).
- Never put client secrets, API keys, or tokens in the connector YAML.

---

## 4. Worked examples

Canonical examples ship in this brain under `connectors/`:

### 4.1 OAuth 2 — `connectors/example-oauth2.yml`

```yaml
name: example-oauth2
url: https://api.example.com
auth-type: oauth2
scope: brain
parameters:
  - name: client-id
    description: OAuth 2 client ID issued by the external provider.
  - name: client-secret
    description: OAuth 2 client secret. Store the value as a brain environment variable — never commit it here.
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
```

### 4.3 No auth — `connectors/example-none.yml`

```yaml
name: example-none
url: https://httpbin.org
auth-type: none
scope: brain
```

No `parameters` key — empty lists are omitted.

### 4.4 Environment-specific base URL — `url-env`

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

### 4.5 Referencing a connector from a tool (BRA199 / BRA200)

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

```yaml
name: search_documents
version: 1
description: Search documents via MCP.
mcp:
  tool: search
  connector: example-none    # optional — or use server-url / server instead
```

At API dispatch the Tool Router resolves the connector URL (+ optional path),
validates it with SSRF guards, and injects OAuth2 / API-key auth. Tools without
`connector:` keep their inline URL / server behaviour.

---

## 5. Deploy and upsert semantics

On `brain deploy`:

1. The CLI uploads schema files (including `connectors/*.yml` when listed).
2. The server parses the compose `connectors:` list and upserts each connector
   by `(BrainId, Name)`.
3. Declared parameters are **replaced wholesale** for that connector.
4. There is **no version field** — a redeploy always applies the incoming
   Name / Url / AuthType / Scope / parameters (upsert-always, like entity types).

Validate with `npm run deploy:dry` before deploying.

---

## 6. Editing connectors at runtime

Schema system tools (**BRA203**) treat connectors as first-class schema files:

| Operation | Support |
|---|---|
| `list_schema_files` / `search_schema_files` | Yes — type token `connector`, path `connectors/{name}.yml` |
| `get_schema_file` | Yes — returns canonical YAML |
| `update_schema_file` | Yes — exact-one-match string replace, then upsert via `IConnectorService` |
| `create_schema_file` | **Not yet** for `connectors/…` — create via deploy (add file + compose entry) |

`brain-compose.yml` remains read-only via `update_schema_file`.

---

## 7. Authoring checklist

1. Create `connectors/{name}.yml` with `name`, exactly one of `url` / `url-env`,
   and `auth-type`.
2. Add declared `parameters` (name + description only) when the connector needs
   credentials; omit the key when it does not.
3. List the path under `connectors:` in `brain-compose.yml`.
4. Put secret **values** and any `url-env` URL values in `.env` (or the secrets
   API) — never in the YAML.
5. Use British English in descriptions.
6. Dry-run deploy, then deploy.

---

## Related skills

| Skill | Topic |
|---|---|
| **BRA201** | Full brain schema authoring reference (connectors summarised in §5A) |
| **BRA202** | Environment variables, encryption, secret injection into tools |
| **BRA203** | Schema system tools (list / get / update connector files) |
