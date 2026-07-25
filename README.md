# Starter Brain

Onboarding guide for a Telos Brain. Precise enough for Claude Code / Cursor to execute. Complete the steps in order.

## 1. Install and initialise

```bash
npm install -g @telos.ready/brain
brain init
```

## 2. Run the Getting Started skill interview

**Requires human input.** An AI agent must not skip or auto-answer this step.

Load and run the **Getting Started** skill (**BRA104**) from the Telos Brain skill book:

`skills/telos-brain/concepts/BRA104-getting-started.md`

The skill interviews you for entity settings, unit of work settings, blueprint categories, and skill categories, then produces a configuration summary to apply to the schema (e.g. `brain-compose.yml`, blueprint manifests, skillbook).

Complete this **before** deploy. Category quality directly determines learning quality — generic categories produce generic learnings.

## 3. Deploy

Copy `.env.example` to `.env` and set at least:

```
TELOS_ORG_API_KEY=your-org-api-key
ANTHROPIC_API_KEY=your-anthropic-api-key
VOYAGE_API_KEY=your-voyage-api-key
```

- `ANTHROPIC_API_KEY` — https://console.anthropic.com
- `VOYAGE_API_KEY` — https://dash.voyageai.com (required; this brain defaults to `voyage-3-lite`)

```bash
brain deploy --instance mybrain
```

**Capture the Brain API key from stdout immediately.** On first deploy the CLI prints a plaintext Brain API key **once only**. Store it securely (password manager / secrets manager). Do **not** commit it to source control.

Do not delete `brain.lock` after first deploy — subsequent deploys read the brain ID from it.

**Redeploy tip:** run `brain snapshot` before redeploying during iterative development to pull live version numbers to disk and avoid HTTP 409 conflicts.

## 4. Train the brain

**Option A — AI agent:** Ask Claude Code / Cursor to build out the schema from your business context (blueprint entries, skills, workflows).

**Option B — Inbox:** Upload documents, transcripts, or emails via the Brain admin UI or API inbox. Processing follows the brain's learning mode.

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
