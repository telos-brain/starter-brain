# Starter Brain

Onboarding guide for a Telos Brain. Precise enough for Claude Code / Cursor to execute. Complete the steps in order.

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

## 3. Deploy

Copy `.env.example` to `.env` and set at least:

```
TELOS_BRAIN_ORG_API_KEY=your-org-api-key
ANTHROPIC_API_KEY=your-anthropic-api-key
VOYAGE_API_KEY=your-voyage-api-key
```

- `TELOS_BRAIN_ORG_API_KEY` — https://go.telosbrain.com (sign up for free and create an API key)
- `ANTHROPIC_API_KEY` — https://console.anthropic.com (needed for the starter workflow `model:` pins)
- `VOYAGE_API_KEY` — https://dash.voyageai.com (required; this brain defaults to `voyage-3-lite`)

Starter workflows pin `anthropic/claude-sonnet-4-6` (compaction uses Haiku) so a brain with only `ANTHROPIC_API_KEY` still runs. To point **every** workflow at one provider/model without editing YAML, set **Default LLM model** in Settings, `DEFAULT_LLM_MODEL` in `.env`, or `llm-model` in `brain-compose.yml` (BRA210). A reachable brain default overrides the workflow pins. Leave those unset to keep each workflow's own `model:`.

Optional:

- `OPENAI_API_KEY` / `XAI_API_KEY` / `OPENROUTER_API_KEY` — for `openai/…`, `xai/…`, or `openrouter/…` models
- `LOCAL_LLM_1_BASE_URL` — Ollama / llama.cpp (`model: local_1/<id>`; BRA106 §8)
- `DEFAULT_LLM_MODEL` — e.g. `local_1/qwen3:8b`, `openrouter/anthropic/claude-sonnet-4.6`, or `anthropic/claude-sonnet-4-6`

```bash
brain deploy --env [local|dev|stage|prod]
```

Optional: `--instance <name>` to name the brain instance. Deploy reads `.env` for the variables the brain should use.

**Capture the Brain API key from stdout immediately.** On first deploy the CLI prints a plaintext Brain API key **once only**. Store it securely (password manager / secrets manager). Do **not** commit it to source control.

Do not delete `brain.lock` after first deploy — subsequent deploys read the brain ID from it.

**Changing models or local LLMs.** After you add or edit `DEFAULT_LLM_MODEL`, `LOCAL_LLM_*`, a provider API key, compose `llm-model`, or a workflow `model:` pin, redeploy so the brain stores the new values:

```bash
brain deploy --env [local|dev|stage|prod]
```

Settings **Default LLM model** applies immediately (no deploy). Persist the same value as `DEFAULT_LLM_MODEL` in `.env` (or `llm-model` in compose) so the next deploy does not clear it. Local Ollama from Brain-in-Docker must use `http://host.docker.internal:11434/v1`, not `localhost`. `ollama pull` (or loading a new llama.cpp weights file) on an already-stored `LOCAL_LLM_N_BASE_URL` does **not** need a redeploy — Settings lists models from the runner live. Full how-to: **BRA106** §8.

**Redeploy tip:** run `brain snapshot` before redeploying during iterative development to pull live version numbers to disk and avoid HTTP 409 conflicts.

## 4. Train the brain

After the schema exists, upload documents, transcripts, or emails via the Brain admin UI or API inbox. Processing follows the brain's learning mode.

**Learning mode:** `brain-compose.yml` defaults to `learning-mode: high`. Recommended: start at `high`, review daily checkpoints for the first 5 days on the Grading graph, then set `low` when learning quality is acceptable.

## 5. Use the brain via the Execute API

Smoke-test with the Brain API key from step 3:

```bash
curl -X POST https://go.telosbrain.com/workflows/WF-CHAT/run/sync \
  -H "Authorization: Bearer YOUR_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"inputMessage": "Hello world"}'
```

Full Execution API docs: Telos Brain skill book **Run** category — start with **BRA401** (authentication conventions), then **BRA402**–**BRA407**.

## 6. Custom harness or DIY

- **Custom harness:** https://www.telosready.com
- **DIY:** follow the Execute API / Run skills (**BRA401** onwards)

The curl in step 5 is a smoke test only. Production use needs a harness wired to your business systems.

## Repository hygiene

Gitignore (do not commit):

- `.env`
- `brain.lock`
- `node_modules/`
- `dist/`

Commit `.env.example` with placeholder values only. Never store the Brain API key in the repo.

## Support

Copyright Telos IP Limited 2026
www.telosbrain.com
support@telosbrain.com
