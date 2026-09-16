# Starter Brain

Onboarding guide for a Telos Brain. Precise enough for Claude Code / Cursor to execute. Complete the steps in order.

An AI agent can take this repository from zero to a running cloud brain with **only a Telos Brain organisation API key**: sign up via the Management API, deploy, then smoke-test `WF-CHAT` on the Execution API. The human must accept the Clerk invite (and sign in) to activate the organisation and receive **$10** welcome credit.

## 1. Install and initialise

```bash
npm install -g @telos.ready/brain
brain init
```

## 2. Building the schema

The starter includes the Telos Brain skill book and learning/maintenance workflows. There are two ways to turn that into *your* brain. Complete this **before** deploy. Category quality directly determines learning quality — generic categories produce generic learnings.

1. **Auto-build from an existing application** — in Cursor or Claude Code, load skill **BRA211** (`skills/telos-brain/brain-schema/BRA211-auto-building-a-brain.md`) and follow it. That skill is fully contained (researches the app, writes the schema, and wires the Execute API). Do not copy that process into this README.
2. **Guided interview** — load skill **BRA104** (`skills/telos-brain/concepts/BRA104-getting-started.md`). **Requires human input** — an AI agent must not skip or auto-answer. It asks one decision at a time (entity, unit of work, blueprint categories, skill categories) and produces a configuration summary to apply.

Use BRA211 when the host application already exists. Use BRA104 for a greenfield brain.

You may skip this step and deploy the starter schema as-is for a smoke test.

## 3. Sign up (Telos Cloud) — agent-executable

Signup contract: skill **BRA301** (`skills/telos-brain/management-api/BRA301-management-api-authentication-conventions.md`). Skip this section if the user already has a `tbk_…` organisation API key.

### Collect these fields from the human (do not invent them)

| Field | Required | Notes |
| ----- | -------- | ----- |
| `accountName` | Yes | Organisation display name (propose the brain name if they have none). |
| `personName` | Yes | Full name of the person who will receive the Clerk invite. |
| `email` | Yes | Their real email. The invite is sent here. |
| `termsAndConditions` | Yes | Must be **explicitly `true`**. Ask them to accept Telos Brain terms. Never set this without confirmation. |

If they already have a `tbk_…` key, write it to `.env` and go to step 4.

### Call the public signup API

Unauthenticated. Do **not** send a Clerk JWT or API key.

```bash
curl -X POST https://go.telosbrain.com/organisations/signup \
  -H "Content-Type: application/json" \
  -d '{
    "accountName": "Acme Corp",
    "personName": "Ada Lovelace",
    "email": "ada@example.com",
    "termsAndConditions": true
  }'
```

`201 Created` returns:

```json
{
  "organisationId": "3f0c8a2e-....",
  "apiKey": "tbk_..."
}
```

**Store `apiKey` immediately.** It is a 1-year Admin organisation key, returned **once only**, and cannot be recovered. Write it to `.env` as `TELOS_BRAIN_ORG_API_KEY`. Do not commit it.

Typical `400` messages: `termsAndConditions must be explicitly true`, `accountName is required`, `personName is required`, `A valid email address is required`.

### Human: accept the invite for $10 credit

Signup creates the organisation as **Pending** with **zero credit**. The org API key works for deploy, but workflow runs are rejected by the credit gate until the organisation is Active.

Tell the user: **check email, accept the Clerk invite, and sign in at https://go.telosbrain.com**. That first Clerk-session visit activates the organisation and grants **$10.00** welcome credit. The organisation API key does **not** trigger activation.

The same email may sign up more than once; each call creates a separate organisation and key.

## 4. Deploy

Copy `.env.example` to `.env` (or create `.env` if signup already wrote the org key).

### Telos Cloud — org API key only

`brain-compose.yml` sets `llm-model: telosbrain/xai/grok-4.6`. On Telos Cloud that model uses the platform Grok key, so **no LLM provider key is required** to deploy or run.

```
TELOS_BRAIN_ORG_API_KEY=tbk_...
TELOS_BRAIN_API_URL=https://go.telosbrain.com
```

**LLM credits.** Telos Brain LLM (`telosbrain/…`) is billed to organisation brain credit at **double** the official xAI grok-4.6 API rate. We recommend adding your own LLM API key and changing `llm-model` in `brain-compose.yml` to that provider (for example `xai/grok-4.6` + `XAI_API_KEY`, or `anthropic/claude-sonnet-4-6` + `ANTHROPIC_API_KEY`) so you pay list price to the vendor. Compose `llm-model` wins over `DEFAULT_LLM_MODEL` at deploy — changing only the env var is not enough.

Optional:

- `VOYAGE_API_KEY` — https://dash.voyageai.com (semantic search; this brain defaults to `voyage-3-lite`). Deploy succeeds without it; embeddings are skipped.
- `OPENAI_API_KEY` / `XAI_API_KEY` / `OPENROUTER_API_KEY` — for `openai/…`, `xai/…`, or `openrouter/…` after you change `llm-model`
- `AZURE_OPENAI_API_KEY` / `AZURE_OPENAI_ENDPOINT` — for `azure/…` models (remainder is the Azure deployment name; BRA210)
- `LOCAL_LLM_1_BASE_URL` — Ollama / llama.cpp (`model: local_1/<id>`; BRA106 §8)
- `DEFAULT_LLM_MODEL` — only used when compose `llm-model` is commented out

```bash
brain deploy --env [dev|stage|prod]
```

Optional: `--instance <name>` to name the brain instance. Deploy reads `.env` for the variables the brain should use.

**Capture the Brain API key from stdout immediately.** On first deploy the CLI prints a plaintext Brain API key **once only**. Store it securely (password manager / secrets manager). Do **not** commit it to source control. That key is what the Execution API uses (not the `tbk_…` org key).

Do not delete `brain.lock` after first deploy — subsequent deploys read the brain ID from it.

### Local Docker — LLM keys required

`telosbrain/xai/grok-4.6` is **unavailable** on a local stack (no platform Grok key). You must supply an LLM and change the default model before workflows will run.

```bash
brain start
```

Then in `.env.local` (created by `brain start`) set a provider key **and** change `llm-model` in `brain-compose.yml` to match, for example:

- `ANTHROPIC_API_KEY` and `llm-model: anthropic/claude-sonnet-4-6`
- `XAI_API_KEY` and `llm-model: xai/grok-4.6`
- `LOCAL_LLM_1_BASE_URL=http://host.docker.internal:11434/v1` and `llm-model: local_1/qwen3:8b`

```bash
brain deploy --env local --instance local-brain
```

Full local stack: **BRA106** (`skills/telos-brain/concepts/BRA106-local-development.md`).

**Changing models or local LLMs.** After you add or edit `DEFAULT_LLM_MODEL`, `LOCAL_LLM_*`, a provider API key, compose `llm-model`, or a workflow `model:` pin, redeploy so the brain stores the new values:

```bash
brain deploy --env [local|dev|stage|prod]
```

Settings **Default LLM model** applies immediately (no deploy). Persist the same value as `llm-model` in compose (or `DEFAULT_LLM_MODEL` in `.env` if compose is commented out) so the next deploy does not clear it. Local Ollama from Brain-in-Docker must use `http://host.docker.internal:11434/v1`, not `localhost`. `ollama pull` (or loading a new llama.cpp weights file) on an already-stored `LOCAL_LLM_N_BASE_URL` does **not** need a redeploy — Settings lists models from the runner live. Full how-to: **BRA106** §8.

**Redeploy tip:** run `brain snapshot` before redeploying during iterative development to pull live version numbers to disk and avoid HTTP 409 conflicts.

## 5. Train the brain

After the schema exists, upload documents, transcripts, or emails via the Brain admin UI or API inbox. Processing follows the brain's learning mode.

**Learning mode:** `brain-compose.yml` defaults to `learning-mode: high`. Recommended: start at `high`, review daily checkpoints for the first 5 days on the Grading graph, then set `low` when learning quality is acceptable.

## 6. Use the brain via the Execute API

Smoke-test with the **Brain API key** from step 4 (not the `tbk_…` org key). On Telos Cloud this fails until the user has accepted the invite and the organisation has credit (step 3).

```bash
curl -X POST https://go.telosbrain.com/workflows/WF-CHAT/run/sync \
  -H "Authorization: Bearer YOUR_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"inputMessage": "Hello world"}'
```

Local stack: use `http://127.0.0.1:60061` instead of `https://go.telosbrain.com`.

Full Execution API docs: Telos Brain skill book **Run** category — start with **BRA401** (authentication conventions), then **BRA402**–**BRA407**.

## 7. Custom harness or DIY

- **Custom harness:** https://www.telosready.com
- **DIY:** follow the Execute API / Run skills (**BRA401** onwards)

The curl in step 6 is a smoke test only. Production use needs a harness wired to your business systems.

## Repository hygiene

Gitignore (do not commit):

- `.env`
- `.env.local`
- `brain.lock`
- `node_modules/`
- `dist/`

Commit `.env.example` with placeholder values only. Never store the organisation API key or the Brain API key in the repo.

## Support

Copyright Telos IP Limited 2026
www.telosbrain.com
support@telosbrain.com
