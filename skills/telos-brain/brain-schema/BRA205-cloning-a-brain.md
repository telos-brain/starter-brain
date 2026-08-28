---
name: Cloning a Brain
code: BRA205
version: 7
description: How to deep-clone a brain instance via the Management API —
  POST /brains/{instance}/clone — including what is copied, what is excluded,
  environment-variable overrides, and the one-time API key response.
---

# Cloning a Brain

A brain clone creates a **new instance** in the same organisation that carries a
complete copy of the source brain's **configuration layer** (schema, skills,
tools, connectors, workflows, memory, type definitions, and encrypted
environment variables). Runtime data is not copied. The clone gets a **fresh API key**,
returned once in the create response — the source key is never reused.

Use cloning to spin up staging, development, or test instances from a known-good
brain without redeploying the schema from scratch, or to promote a configuration
while swapping credentials via environment-variable overrides.

This is a Management API operation (deploy plane), not an Execution API call.
See BRA401 for the Management vs Execution distinction.

---

## 1. Endpoint

```http
POST /brains/{instance}/clone
Authorization: Bearer <clerk-org-jwt>
# or
X-Telos-Api-Key: <organisation-api-key>
```

| Part | Meaning |
|---|---|
| `{instance}` | Source brain's **instance name** (organisation-scoped DNS slug), not a GUID |
| Auth | Clerk org session or organisation API key; **Admin** role required |
| Tenant scope | Clone stays in the caller's organisation; cross-org clone is not supported |

### Request body

```json
{
  "newName": "salesmate-staging",
  "environmentVariables": {
    "ANTHROPIC_API_KEY": "sk-ant-...",
    "VOYAGE_API_KEY": "pa-..."
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `newName` | Yes | Target **instance name** for the clone. Same rules as create: 3–63 characters, lowercase letters, digits and hyphens only, no leading/trailing hyphen. Must be unique within the organisation. |
| `environmentVariables` | No | Plaintext key/value map. Merged over the source brain's env vars (see §3). |

### Response `201 Created`

```json
{
  "id": "3f0c8a2e-....",
  "apiKey": "<plaintext-256-bit-key>"
}
```

| Field | Notes |
|---|---|
| `id` | GUID of the new brain |
| `apiKey` | Fresh Execution API credential. Returned **once only** — store it immediately. Subsequent reads mask it to the last four characters. |

The key is freshly generated. It is **not** a copy of the source brain's key.

### Status codes

| Status | Meaning |
|---|---|
| `201 Created` | Clone succeeded |
| `400 Bad Request` | Missing `newName`, or `newName` fails instance-name validation |
| `401 Unauthorized` | No authenticated organisation |
| `403 Forbidden` | Caller lacks Admin role |
| `404 Not Found` | No brain named `{instance}` in the caller's organisation |
| `409 Conflict` | An instance named `newName` already exists in the organisation |
| `500 Internal Server Error` | Clone failed unexpectedly (e.g. database constraint or schema mismatch). The `error` body includes the underlying failure detail so the caller can act on it. |

Errors use the standard Management API shape: `{ "error": "message" }`.

---

## 2. What is cloned

Everything below is deep-copied onto the new brain. Codes, names, versions, and
embedding bytes are preserved exactly so CLI deploy version-precedence rules and
semantic search keep working on the clone.

| Layer | Copied |
|---|---|
| **Brain header** | Display name, description, and embedding model (unchanged — do not reset; changing it would invalidate copied embeddings). New instance name, new API key, status `Active`. |
| **Connectors** | Connector definitions (name, URL / url-env, auth type, platform `type`, OAuth endpoints, api-key header, declared parameters including each parameter's `secret:` env-var name). Names are preserved so tools' `connector:` field still resolves. Connector-scoped env-var keys (`CONNECTOR_{id}_*`) are remapped to the new connector IDs. Named keys such as `ELEVENLABS_API_KEY` copy as-is. |
| **Tools** | Tool groups, tools (including embeddings), and parameters |
| **Workflows** | Workflows (all LLM settings, skill-code lists, versions, `deployment-type`) with injected/available tools re-linked and `input-tools` copied. `elevenlabs-agent-id` is **not** copied — the clone creates its own agent on first deploy. |
| **Skills** | Skill books, categories, and skills (including embeddings and tool-code lists) |
| **Memory** | Blueprints, categories, entries, chunks, and embeddings |
| **Schema types** | Entity types and their variable keys; unit-of-work types |
| **Secrets** | Encrypted environment variables (copied with the clone — see §3) |

### What is **not** cloned

Runtime / operational data stays on the source brain only:

- Workflow runs and messages
- Entities and their variables
- Units of work and their context / data
- Inbox entries and tasks
- OAuth access / refresh tokens (the clone must reconnect)
- ElevenLabs agent ids (`elevenlabs-agent-id`) — the clone must deploy its own agent

The clone is a clean configuration twin — not a copy of live jobs, chat history,
or inbox state.

### Transactionality

The whole clone is **all-or-nothing**. A failure leaves no partial clone.

---

## 3. Environment-variable merge

Environment variables are encrypted at rest — the same mechanism described in
**BRA202**. Source secrets can be copied directly onto the clone.

When the request includes `environmentVariables`:

1. Start from **all** live env vars on the source brain.
2. For each key in the request, **encrypt** the plaintext value and **override**
   that key on the clone.
3. Source keys not mentioned in the request are carried across unchanged,
   except connector-scoped keys (`CONNECTOR_{id}_*`), which are rewritten onto
   the clone's new connector IDs so api-key secrets still resolve.
4. Request keys that do not exist on the source are **added** on the clone.

This supports “promotion with credential swap”: clone production config into
staging and replace provider keys without re-specifying every variable.

Blank keys or blank values in the request map are skipped.

---

## 4. Instance name vs display name

Brains have two name fields:

| Field | Role |
|---|---|
| `InstanceName` | Organisation-scoped slug used in every Management API route (`/brains/{instance}/…`). Unique per organisation. |
| `Name` | Human-readable display name. Not unique. |

On clone, `newName` becomes the new **`InstanceName`**. Display `Name` and
`Description` are copied from the source. Uniqueness / `409` is enforced on
`InstanceName` only.

---

## 5. Typical flow

```bash
# Clone production config into a staging instance, swapping LLM credentials
curl -X POST "$TELOS_API_URL/brains/salesmate/clone" \
  -H "X-Telos-Api-Key: $TELOS_ORG_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "newName": "salesmate-staging",
    "environmentVariables": {
      "ANTHROPIC_API_KEY": "'"$STAGING_ANTHROPIC_KEY"'"
    }
  }'
```

Store the returned `apiKey` as the Execution API credential for the new
instance (e.g. in the staging brain's `.env` as the value consumers use with
`Authorization: Bearer …`). Deploy further schema changes to the clone with the
normal CLI path (`brain deploy . --instance salesmate-staging`) — version
numbers already match the source, so subsequent deploys must bump versions as
usual (see BRA201 versioning rules).

When the **source template** later gains newer configuration and you want to
pull those improvements into the clone without overwriting destination resources
that have moved ahead, use update-from-template — see **BRA206**.

---

## Related skills

- **BRA201** — schema authoring and deploy versioning
- **BRA202** — environment variables, encryption, and secret injection
- **BRA206** — updating a previously cloned brain from its template
- **BRA401** — Execution API auth (uses the API key returned by clone/create)
