---
name: "Execution API: Users and user variables"
code: BRA416
version: 4
description: How to add, update, and remove brain users on the Execution API,
  and how their encrypted variables are written. Covers users.yml, which
  upserts names and emails and never contains variable values. There is no
  read endpoint for a variable.
---

# Execution API: Users and user variables

See **BRA401** for authentication and conventions. See **BRA209** for connector
`scope: user`, which stores that person's API key or OAuth tokens as user
variables. See **BRA202** for brain environment variables, which are a
different store.

A **brain user** is a person on one brain, keyed by email. A **user variable**
is an encrypted secret on that person. The harness writes variables through
this API. Nothing in the API, `users.yml`, or a log returns the value.

Tenancy is implicit (**BRA401**). Do not send a brain id.

---

## What is stored

| | Brain user | User variable |
|---|---|---|
| Identity | `name` and `email` | `key` on that user |
| Email | Trimmed and stored lower-case. Unique among live users in the brain | — |
| Value | — | Encrypted at rest. Write-only |
| Limits | Name and email, 255 characters | Key, 255 characters |
| Removed email | The row is hidden. Adding the same email again creates a **new** id | Removed with the user |

There is no list or get endpoint. A caller that needs the id keeps the `id`
returned by add or update.

---

## `POST /users`

Creates a user. `name` and `email` are required. `variables` is optional.

```http
POST /users
Authorization: Bearer <brain-api-key>
Content-Type: application/json

{
  "name": "Jane Smith",
  "email": "Jane@Example.com",
  "variables": [
    { "key": "crm-api-key", "value": "secret-value" }
  ]
}
```

`201 Created`:

```json
{ "id": "3f0c..." }
```

The stored email is `jane@example.com`. The response is the id only.

| Status | When |
|---|---|
| `201 Created` | User created. Body is `{ "id" }` |
| `400 Bad Request` | Missing or invalid name, email, or variable key. `{ "error" }` |
| `409 Conflict` | A live user already has that email |
| `500 Internal Server Error` | The user could not be saved. The message does not include a variable value |

A blank `value` is skipped and does not create that key. If the same key
appears twice in one request, the later value is stored.

---

## `PATCH /users`

Updates the live user with that email. `email` is required. Send `name`,
`variables`, or both. Sending neither is `400` with
`A name or variables list is required.`

```http
PATCH /users
Authorization: Bearer <brain-api-key>
Content-Type: application/json

{
  "email": "jane@example.com",
  "name": "Jane S.",
  "variables": [
    { "key": "crm-api-key", "value": "rotated-secret" }
  ]
}
```

`200 OK` returns `{ "id" }` for the same user. Keys not in `variables` are
left as they are. A blank `value` does not replace or clear the stored secret.
An unknown email is `404` with `No user with email '…' was found.`

---

## `DELETE /users?email=`

Soft-deletes the user and that user's variables.

```http
DELETE /users?email=jane@example.com
Authorization: Bearer <brain-api-key>
```

`204 No Content` on success. An unknown email is `404`. A missing or invalid
email is `400`.

Adding that email again creates a new user id. Old variable rows stay deleted.

---

## Variable rules

- The value is encrypted before it is stored, using the same encryption as
  brain environment variables (**BRA202**).
- The plaintext and the ciphertext are never returned, logged, or written
  into `users.yml`.
- Keys are free text up to 255 characters. Connector scope `user` uses the
  keys in the table below; other keys are up to the harness.
- A connector's client id and client secret stay brain environment variables
  for both scopes. They are not user variables. OAuth Connect, refresh, and
  HMAC signing read them from that store.
- A user-scoped API connector sends the person's key as `api-key-header`
  (or `Authorization: Bearer`) and can still send brain environment variables
  as other headers. Those come from the connector's `request-defaults`
  `secret:` and from a tool parameter `secret:` with `header:`.

| Connector `scope: user` | User-variable key |
|---|---|
| API key | `CONNECTOR_{connectorId}_CLIENT_SECRET` |
| OAuth access token | `CONNECTOR_{connectorId}_ACCESS_TOKEN` |
| OAuth refresh token | `CONNECTOR_{connectorId}_REFRESH_TOKEN` |
| OAuth expiry | `CONNECTOR_{connectorId}_EXPIRES_AT` |
| Account label | `CONNECTOR_{connectorId}_CONNECTED_AS` |

`{connectorId}` is the connector GUID. Connect on the admin Connectors page
writes these keys for the linked member. The brain user is linked to the
Clerk member when the emails match and that member can use this brain through
MCP: they are an Admin of the brain's organisation, or a Member with Full or
McpOnly access. The link is set when the brain user is added and when the
member later gains that access. Without it, Connect does not store a key.
Dispatch reads the keys for the run's acting user and fails closed when there
is no acting user or the secret is missing (**BRA209**).

A tool parameter `user:` names one of these variables (or `id` for the acting
user's id) and injects it on that call. `secret:` on a tool parameter remains
a brain environment variable. See **BRA214**. The Execute API `userId` is
optional; when present it must be a user of this brain. MCP uses the linked
brain user and still runs when there is no link.

---

## `users.yml`

The file sits at the brain root. It is not listed in `brain-compose.yml`.
Schema list, search, get, and apply always include it. An empty brain
serialises as:

```yaml
users: []
```

Otherwise, names and emails only, ordered by email:

```yaml
users:
  - name: Jane Smith
    email: jane@example.com
```

Deploy, apply, and create **upsert** each row by email: a known email updates
the name, a new email inserts a user. People absent from the file are left in
place. Changing an email in the file adds a second user and leaves the old
email. The file cannot set or clear variables.

`users.yml` is not copied by clone (**BRA205**) or update-from (**BRA206**).
User variables are not copied either.

---

## Related skills

| Skill | Topic |
|---|---|
| **BRA401** | Authentication, tenancy, error envelope |
| **BRA201** | Where `users.yml` sits in the schema |
| **BRA202** | Brain environment variables (a different secret store) |
| **BRA209** | Connector `scope: user` |
