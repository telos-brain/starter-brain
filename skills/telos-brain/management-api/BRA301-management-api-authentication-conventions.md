---
name: "Management API: Authentication, Conventions & Signup"
code: BRA301
version: 1
description: How to authenticate with the Telos Brain Management API, the
  common conventions that apply across deploy-plane endpoints, and the public
  organisation signup endpoint (POST /organisations/signup) used for
  agent-initiated onboarding.
---

# Management API: Authentication, Conventions & Signup

The Management API is the deploy and admin plane of Telos Brain. It is entirely
separate from the Execution API (the runtime surface used by harnesses) —
different middleware, different credentials, different routes. See **BRA401**
for the Execution API contract.

| | Management API | Execution API |
|---|---|---|
| Purpose | Provision organisations and configure brains | Run workflows against entities/units of work |
| Auth | Clerk org JWT, or `tbk_` organisation API key | Per-brain API key |
| Tenant scope | Explicit `BrainId` / `{instance}` per route | Implicit — resolved from the brain API key |
| Routes | `/organisations/...`, `/brains/...` | `/brain`, `/entities`, `/units-of-work`, `/workflows`, `/runs`, `/inbox`, `/skills`, `/transcription` |

Specific Management API operations live in other skills where they belong to
schema work: cloning (**BRA205**), update-from-template (**BRA206**). This skill
covers authentication, shared conventions, and public signup.

---

## Authentication

Authenticated Management API calls accept either:

```http
Authorization: Bearer <clerk-org-jwt>
```

or an organisation API key (`tbk_…` prefix):

```http
X-Telos-Api-Key: <organisation-api-key>
```

`Authorization: Bearer tbk_…` is also accepted as a convenience. Organisation
API keys are minted once (signup, or `POST /organisations/current/api-keys`)
and the plaintext is returned **exactly once**. Subsequent reads mask the key.

Most write endpoints require the **Admin** role. Organisation API keys created
by signup are Admin keys.

### Auth behaviour

| Situation | Response |
|---|---|
| Missing or malformed credentials | `401 Unauthorized` |
| Key is invalid, expired, or revoked | `401 Unauthorized` |
| Caller lacks the required role | `403 Forbidden` |
| Resource belongs to a different organisation | `404 Not Found` (never `403`) |

The exception is **public signup** (`POST /organisations/signup`), which is
unauthenticated — see below.

---

## Conventions

- **Base URL** — paths are relative to the Management API host (for example
  `https://api.telosbrain.com`).
- **Content type** — request and response bodies are JSON (`application/json`).
- **Tenancy** — organisation is resolved from the Clerk session or organisation
  API key. Brain routes use the organisation-scoped `{instance}` slug, not a
  GUID.
- **Errors** — failures return `{ "error": "message" }` with an appropriate
  status code.

### Common status codes

| Status | Meaning |
|---|---|
| `200 OK` | Successful read or idempotent action |
| `201 Created` | Resource created |
| `204 No Content` | Successful delete / revoke |
| `400 Bad Request` | Missing or invalid field |
| `401 Unauthorized` | Missing or invalid credentials |
| `403 Forbidden` | Authenticated but role insufficient |
| `404 Not Found` | Resource not found in the caller's organisation |
| `409 Conflict` | Version conflict or unique-name clash |

---

## Public organisation signup

### `POST /organisations/signup`

Public, unauthenticated endpoint for agent-initiated onboarding. No Clerk JWT
or API key is required. Use this when an application or AI agent needs to
register a new organisation without an existing account.

```http
POST /organisations/signup
Content-Type: application/json
```

### Request body

```json
{
  "accountName": "Acme Corp",
  "personName": "Ada Lovelace",
  "email": "ada@example.com",
  "termsAndConditions": true
}
```

| Field | Required | Notes |
|---|---|---|
| `accountName` | Yes | Organisation display name. Trimmed; blank is rejected. |
| `personName` | Yes | Full name of the person who will receive the Clerk invite. |
| `email` | Yes | Well-formed email address for the Clerk invite. |
| `termsAndConditions` | Yes | Must be **explicitly `true`**. `false` or a missing field returns `400`. |

### Response `201 Created`

```json
{
  "organisationId": "3f0c8a2e-....",
  "apiKey": "tbk_..."
}
```

| Field | Notes |
|---|---|
| `organisationId` | GUID of the new organisation |
| `apiKey` | 1-year Admin organisation API key (`tbk_` prefix), named `Signup key`. Returned **once only** — store it immediately. It cannot be recovered. |

### What happens on success

1. A new organisation is created with `Status = Pending` and a unique `Code`
   slug derived from the account name.
2. A 1-year Admin organisation API key is minted and returned in plaintext.
3. A Clerk invitation is sent to `email` / `personName`. If Clerk is
   unconfigured or the invite fails, signup still returns `201` — the
   organisation and API key are already created.

The same email may sign up more than once; each call creates a **separate**
organisation and key.

### Pending organisation behaviour

A Pending organisation is suspended: it has **zero effective credit** and does
**not** receive the $10 welcome credit at creation time. The API key is live,
but workflow runs are rejected by the credit gate.

When the invited user accepts the Clerk invite and makes their first
authenticated Management API request (Clerk session — not the organisation API
key), the organisation transitions **Pending → Active** and is granted **$10.00**
welcome credit (`1000` cents, description `Welcome credit`).

### Status codes

| Status | Meaning |
|---|---|
| `201 Created` | Organisation created; API key returned once |
| `400 Bad Request` | `termsAndConditions` is not `true`, `accountName` / `personName` missing, or email is invalid |

Typical `400` messages:

- `termsAndConditions must be explicitly true`
- `accountName is required`
- `personName is required`
- `A valid email address is required`
- `Failed to create organisation` (unexpected create failure)

---

## Related organisation endpoints

These require an authenticated organisation (Clerk or the `tbk_` key from
signup) and are **not** public:

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/organisations/me` | Admin | Current organisation profile |
| `PATCH` | `/organisations/me` | Admin | Rename the organisation |
| `GET` | `/organisations/me/balance` | Admin / Member | Credit balance in cents |
| `GET` | `/organisations/me/billing/statement` | Admin | Month-scoped billing statement |
| `POST` | `/organisations/current/api-keys` | Admin | Mint an additional organisation API key |
| `GET` | `/organisations/current/api-keys` | Admin | List keys (plaintext never returned) |
| `DELETE` | `/organisations/current/api-keys/{id}` | Admin | Revoke a key |

Use the signup-issued key as `X-Telos-Api-Key` (or `Authorization: Bearer tbk_…`)
for CLI deploy (`brain login` / `brain deploy`) after the organisation is
Active and has credit.
