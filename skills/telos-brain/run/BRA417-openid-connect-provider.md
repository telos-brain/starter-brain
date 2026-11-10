---
name: "OpenID Connect provider for Telos Brain MCP"
code: BRA417
version: 6
description: How to run an OpenID Connect provider that Telos Brain calls as
  a confidential client. Use when a brain's MCP users must sign in at your
  identity provider. Telos Brain is the client. You are the provider.
---

# OpenID Connect provider for Telos Brain MCP

Use this when you run the identity provider and a Telos Brain administrator
wants that brain's MCP users to sign in with you.

There are three parties. Keep them separate.

| Party | Role |
|---|---|
| **You** | The OpenID Connect **provider** (the issuer). You authenticate the person and issue an ID token to Telos Brain. |
| **Telos Brain** | The OpenID Connect **client**. It is a confidential client that you register. It calls your authorise, token, and key endpoints. |
| **MCP application** | Not your client. Cursor and other MCP applications talk only to Telos Brain. They never register with you, and they never present your tokens. |

Telos Brain remains the OAuth authorisation server for MCP. After you sign
the person in, Telos Brain issues its own MCP access token to the MCP
application. Your ID token and your access token are not MCP credentials.
Telos Brain does not forward them to tools.

The choice is per brain. Another brain in the same organisation can keep
Telos Brain sign-in. Organisation-scoped MCP does not use this option.

OpenID Connect requires a Pro or Enterprise plan. On any other plan the
brain's MCP page still shows the option, and it cannot be selected or saved.

## Register Telos Brain as a client

Create one confidential client for Telos Brain. One registration can serve
every brain that uses this provider. The redirect URI is the same for all of
them. You may also issue a separate client per brain. Either way, the brain
administrator saves that brain's issuer, client id, and client secret on the
brain's MCP page.

| Setting on your client | Value |
|---|---|
| Client type | Confidential. |
| Token endpoint authentication | `client_secret_post`. Telos Brain sends `client_id` and `client_secret` in the form body. It does not send HTTP Basic authentication. |
| Grant | Authorisation code only. |
| PKCE | S256. Required. Telos Brain always sends a code challenge, including for this confidential client. |
| Redirect URI | The exact value shown on the brain's MCP page. In production this is `https://mcp.telosbrain.com/oauth/oidc/callback`. |
| Scopes | `openid`, `profile`, and `email`. Add any further scopes the brain administrator will request. |

Give the brain administrator three values. They paste them into that brain.
Telos Brain stores the client secret encrypted and does not show it again.

| They save | What it is |
|---|---|
| Issuer URL | Your issuer. An absolute `https` URL with no username, password, query, or fragment. |
| Client ID | The `client_id` you issued to Telos Brain. The ID token `aud` must be this value. |
| Client secret | The secret for that client. |

Do not give them an MCP application's redirect URI. Telos Brain's callback is
the only redirect URI you register.

## What Telos Brain calls

Telos Brain fetches your discovery document before it will save the setting,
and again at each sign-in. Discovery is:

`GET {issuer}/.well-known/openid-configuration`

If the administrator pastes that discovery URL, Telos Brain strips
`/.well-known/openid-configuration` and uses the remainder as the issuer.
The document must return `200` and JSON. Telos Brain does not follow
redirects. A `301` or `302` is a failure.

The document's `issuer` must be the same URL (a trailing slash is ignored).
These fields are required:

- `authorization_endpoint`
- `token_endpoint`
- `jwks_uri`

`userinfo_endpoint` is optional. Telos Brain calls it only when the ID token
has no usable `email` claim.

Every URL must be public `https`. Telos Brain refuses private, loopback,
link-local, and metadata addresses. An issuer on an internal network cannot
be used.

### Authorise

Telos Brain sends the browser to your `authorization_endpoint`:

```text
GET {authorization_endpoint}
  ?response_type=code
  &client_id=<the client id you issued>
  &redirect_uri=https://mcp.telosbrain.com/oauth/oidc/callback
  &scope=openid profile email
  &state=<Telos Brain request id>
  &nonce=<Telos Brain nonce>
  &code_challenge=<S256>
  &code_challenge_method=S256
```

`scope` always starts with `openid profile email`. Further scopes are only
those the brain administrator saved.

Return the browser to the redirect URI with `code` and the same `state`.
On failure, return `error` and the same `state`. Do not redirect anywhere
else. Do not put an ID token in the redirect. Telos Brain does not use the
implicit or hybrid flows.

### Token

Telos Brain redeems the code itself. The browser does not call your token
endpoint.

```text
POST {token_endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
code=<code>
redirect_uri=https://mcp.telosbrain.com/oauth/oidc/callback
client_id=<the client id you issued>
client_secret=<the client secret you issued>
code_verifier=<the PKCE verifier>
```

The response must include `id_token`. Include `access_token` as well when
the email is not in the ID token and you publish `userinfo_endpoint`.
Telos Brain discards both tokens after it has read the identity. It does not
store them, refresh them, or send them to MCP.

Telos Brain does not call you again when the MCP application refreshes.
Refreshing an MCP session does not refresh a session with you.

## ID token

Sign the ID token with RS256, RS384, RS512, ES256, ES384, or ES512. Publish
the key at `jwks_uri`. `none` is rejected.

| Claim | Requirement |
|---|---|
| `iss` | Your issuer, matching the discovery document. |
| `aud` | The client id you issued to Telos Brain. |
| `exp` | Required. Telos Brain allows two minutes of clock skew. |
| `nonce` | The `nonce` from the authorise request, unchanged. |
| `sub` | Required. Stable for the person. At most 255 characters. |
| `email` | The person's email address. Used to find their brain user. |
| `entity_id` | Optional. The id of an entity that already belongs to that brain. When present, Telos Brain locks the MCP session to it. |

Telos Brain does not create a brain user from this token, and it does not
link the subject to a Telos Brain member. It matches `email`
case-insensitively to a brain user that already exists on that brain. A
match injects that user's variables into the MCP run. No match still
completes sign-in, without those variables. Put the address in the ID token
when you can. Otherwise return it from userinfo as `email`.

`entity_id` must be a GUID. Telos Brain does not create an entity from this
claim. A value that is blank, not a GUID, or not an entity on that brain
rejects the sign-in. Omit the claim when you are not choosing an entity. The
MCP application can still pass its own `entity_id` to Telos Brain. When both
are present, the ID token value is the one stored on the MCP session.

## What you do not receive

- The MCP application's client id, redirect URI, or PKCE challenge. Those
  belong to Telos Brain's own MCP authorisation server.
- The MCP application's `entity_id`. Telos Brain does not send that value to
  you. Set `entity_id` on the ID token when the session should run as one of
  that brain's entities.
- A Telos Brain access token (`tbm_`) or an organisation API key (`tbk_`).
  If you see one, it was not sent by this client.

## Troubleshooting

| What you see | What to check |
|---|---|
| The brain will not save the issuer | Discovery did not return `200` JSON, the document redirected, `issuer` does not match, or `authorization_endpoint`, `token_endpoint`, or `jwks_uri` is missing. The host must be public HTTPS. |
| Redirect URI mismatch | Register the MCP page's redirect URI exactly. It is Telos Brain's callback, not the MCP application's callback. |
| Token endpoint returns `invalid_client` | Accept `client_secret_post`. Telos Brain does not use HTTP Basic authentication. |
| Sign-in returns to the MCP application as `access_denied` | The ID token failed validation. Check `iss`, `aud` (the client id), `exp`, `nonce`, `sub`, and the signing algorithm. The key must be on `jwks_uri`. A present `entity_id` that is blank or not a GUID fails here too. |
| Sign-in returns to the MCP application as `invalid_request` after you issued a code | `entity_id` is not an entity on that brain. Telos Brain does not create one. |
| The person is signed in but their variables are missing | `email` was absent from the ID token and from userinfo, or it does not match a brain user on that brain. Telos Brain does not create the user. |
| You expected a call when the MCP session refreshed | Telos Brain does not call the provider on refresh. Identity from the original ID token is kept on the MCP token. |
