---
name: LLM Providers and Models
code: BRA210
version: 15
description: Supported AI providers for workflow runs, the provider/model string
  format, example model codes, credential variable names, OpenRouter, Azure
  OpenAI, local OpenAI-compatible runners (Ollama / llama.cpp), and which LLM
  execution settings apply per provider.
---

# LLM Providers and Models

Workflows choose an LLM with the optional frontmatter field `model`. The value
is a **`provider/model`** string. The platform resolves the provider prefix and looks up credentials from brain
environment variables (see **BRA202**).

This skill is the authoring reference for which providers are supported and
which model codes are known to work. Pricing for cost calculation is managed
in organisation settings as a fallback. OpenRouter runs prefer
the billed `usage.cost` on each API response when every token-bearing message
has one; otherwise a missing organisation price leaves cost unset rather
than failing the run. OpenRouter catalogue ids use dots
(`anthropic/claude-sonnet-4.6`); native Anthropic ids use hyphens
(`claude-sonnet-4-6`).

---

## 1. Model string format

```yaml
model: provider/model-name
```

| Form | Behaviour |
| ---- | --------- |
| `anthropic/claude-sonnet-4-6` | Uses the Anthropic conversant and `ANTHROPIC_API_KEY` |
| `openai/gpt-4o` | Uses the OpenAI conversant and `OPENAI_API_KEY` |
| `xai/grok-4.5` | Uses the xAI (Grok) conversant and `XAI_API_KEY` |
| `openrouter/anthropic/claude-sonnet-4.6` | Uses OpenRouter (`OPENROUTER_API_KEY`). Remainder is the OpenRouter model id |
| `openrouter/auto` | OpenRouter Auto Router — OpenRouter picks a model per turn |
| `azure/gpt-4o-prod` | Uses Azure OpenAI (`AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_ENDPOINT`). Remainder is the **deployment name** |
| `local_1/qwen3:8b` | Uses runner 1 (`LOCAL_LLM_1_BASE_URL`). Remainder is the runner's model id |
| `local_2/gemma4:e4b` | Uses runner 2 (`LOCAL_LLM_2_BASE_URL`) |
| `claude-sonnet-4-6` (no prefix) | Treated as **anthropic** (default provider for unprefixed names) |
| omitted / null | Brain default (`llm-model` / `DEFAULT_LLM_MODEL` / Settings) when set and reachable; otherwise the run **fails**. Leftover cloud keys are not used as a silent default. |

Both `/` and `\` are accepted as the separator. The provider prefix is
case-insensitive (`OpenAI/gpt-4o` → `openai`). The alias `claude/…` folds to
`anthropic`.

Named cloud providers require `<PROVIDER>_API_KEY` on the brain. Azure OpenAI
is the exception: it needs `AZURE_OPENAI_API_KEY` **and**
`AZURE_OPENAI_ENDPOINT` (not `AZURE_API_KEY`). Local runners require
`LOCAL_LLM_N_BASE_URL` instead (not `LOCAL_1_API_KEY`). If the matching
variable is missing, the resolver tries the next candidate in the chain below.

### Resolution order

1. Simulation `modelOverride` (this run only), if set. Missing credential
   fails the run — no silent fallback. Older clients may send
   `settingsOverride.model` instead; the dedicated field wins when both are
   present.
2. Brain default: Settings, compose `llm-model`, or `DEFAULT_LLM_MODEL`, if set
   and the matching env var exists. Compose `llm-model` wins over the env key
   at deploy time. A missing credential falls through to the workflow model.
3. Workflow frontmatter `model:`.
4. **Fail** — there is no silent Anthropic/OpenAI/OpenRouter/Azure platform default. A leftover
   `ANTHROPIC_API_KEY` (or OpenAI / xAI / OpenRouter / Azure key) is not used just because it is
   present; overnight / heartbeat runs must not spend cloud tokens by accident.

If none of those candidates have a credential, the run does not start and the
error lists what was tried (or explains that no model is configured). Set a
brain default when you want every workflow to use a local runner without editing
each YAML file. Deploy warns (does not 409) when executable workflows have no
`model:` and no default is set. SYSTEM workflows are skipped for that warning.

---

## 2. Supported providers

| Provider prefix | Conversant | Credential (`.env`) | Notes |
| --------------- | ---------- | ------------------- | ----- |
| `anthropic` (alias `claude`) | Claude | `ANTHROPIC_API_KEY` | Default provider for **unprefixed** model names |
| `openai` | OpenAI Chat Completions | `OPENAI_API_KEY` | Also used for OpenAI embedding models when configured |
| `xai` | OpenAI-compatible Chat Completions at `api.x.ai` | `XAI_API_KEY` | Grok models; response `reasoning_content` mapped to Thinking |
| `openrouter` | OpenAI-compatible Chat Completions at `openrouter.ai/api` | `OPENROUTER_API_KEY` | Aggregator. Remainder after the first `/` is the OpenRouter model id (`anthropic/claude-sonnet-4.6`, `openai/gpt-4o`, …). Settings lists a capped catalogue from `/v1/models` when the key is present. |
| `azure` | Azure OpenAI Chat Completions | `AZURE_OPENAI_API_KEY` and `AZURE_OPENAI_ENDPOINT` (optional `AZURE_OPENAI_API_VERSION`) | The remainder after the first `/` is the Azure **deployment name**, not the underlying model name. V1 is API key auth only (Managed Identity deferred). Do not seed `LlmPrices` — Azure bills the customer's Azure subscription; `CostCents` is null (UI `—`). Platform credits still apply via `RunSeconds`. See §3a. |
| `local_N` (e.g. `local_1`) | Local OpenAI-compatible | `LOCAL_LLM_N_BASE_URL` (required), `LOCAL_LLM_N_API_KEY` (optional) | Ollama, llama.cpp, or any OpenAI-compatible local server. See §3. |

Any other prefix is rejected at run time (`NotSupportedException`).

---

## 3. Local runners (Ollama / llama.cpp)

Numbered env vars register one or more OpenAI-compatible local endpoints.
`local_1/qwen3:8b` uses `LOCAL_LLM_1_BASE_URL`; `local_2/…` uses runner 2, and
so on. The remainder after the first `/` is the runner's model id, passed
verbatim.

```env
# Brain in Docker talking to Ollama on the host:
LOCAL_LLM_1_BASE_URL=http://host.docker.internal:11434/v1
# Native Brain (not Docker) talking to Ollama on the same machine:
# LOCAL_LLM_1_BASE_URL=http://localhost:11434/v1
# llama.cpp server:
# LOCAL_LLM_1_BASE_URL=http://localhost:8080/v1
# LOCAL_LLM_1_API_KEY=  # optional; omit for unsecured Ollama
```

How to wire this on a local Docker stack (including `host.docker.internal`):
**BRA106** §8. After changing `.env` runner URLs, `DEFAULT_LLM_MODEL`, compose
`llm-model`, or workflow `model:`, run `brain deploy` so the brain picks them
up. `ollama pull` of a new model on an already-stored runner URL does not
need a redeploy — Settings lists models from the runner live.

Do not add organisation prices for local runners. Cost stays unset (UI `—`).
Local runners do not replace embeddings. Semantic search still needs
`VOYAGE_API_KEY` or `OPENAI_API_KEY` when you want it; deploy does not require
either.

---

## 3a. Azure OpenAI

Azure OpenAI cannot be reached by pointing the OpenAI Chat Completions path at
a custom base URL. The Azure REST API uses:

```
POST {AZURE_OPENAI_ENDPOINT}/openai/deployments/{deployment}/chat/completions?api-version=2024-10-21
api-key: {AZURE_OPENAI_API_KEY}
```

The `model` field in workflow YAML is the customer's **deployment name**
(e.g. `azure/gpt-4o-prod`). That name is chosen in Azure and may not match the
underlying model (`gpt-4o`). Telos Brain treats it opaquely.

```env
AZURE_OPENAI_API_KEY=your_azure_openai_api_key
AZURE_OPENAI_ENDPOINT=https://YOUR-RESOURCE.openai.azure.com
# Optional. Defaults to 2024-10-21 when omitted.
# AZURE_OPENAI_API_VERSION=2024-10-21
```

V1 supports API key auth only. Managed Identity / Microsoft Entra ID is not
wired yet.

Do not seed `LlmPrices` rows for `azure`. Azure bills the customer's Azure
subscription directly. `CostCents` is null (UI `—`). Platform credits are still
charged from `RunSeconds × CentsPerMinute` — Azure runs are not free.

---

## 4. Example model codes

These are example workflow `model` values. Provider catalogues change over time —
use a model id your API key can call. Prefer an explicit `provider/` prefix.

### Anthropic / Claude

| `model` value | Typical use |
| ------------- | ----------- |
| `anthropic/claude-sonnet-4-6` | Strong general / agentic workflows |
| `anthropic/claude-haiku-4-5` | Faster / cheaper turns |
| `anthropic/claude-opus-4-5` | Highest capability Claude |

### OpenAI / ChatGPT

| `model` value | Typical use |
| ------------- | ----------- |
| `openai/gpt-4o` | Flagship multimodal chat / tools |
| `openai/gpt-4o-mini` | Cost-efficient general work |
| `openai/gpt-4.1` | Strong code / instruction following |
| `openai/o3` | Extended reasoning |

### xAI / Grok

| `model` value | Typical use |
| ------------- | ----------- |
| `xai/grok-4.5` | Flagship Grok coding / agentic |
| `xai/grok-4.3` | Lower-cost long-context Grok |
| `xai/grok-build-0.1` | Coding-focused early access |
| `xai/grok-4.20-0309-reasoning` | Reasoning-oriented Grok 4.20 |
| `xai/grok-4.20-0309-non-reasoning` | Non-reasoning Grok 4.20 |
| `xai/grok-4.20-multi-agent-0309` | Multi-agent long-context |

### OpenRouter

OpenRouter model ids already include a provider path. Prefix with `openrouter/`.
The remainder after the first `/` is the OpenRouter catalogue id — those ids
use dots (`anthropic/claude-sonnet-4.6`), not the hyphen Anthropic uses on its
own API.

| `model` value | Typical use |
| ------------- | ----------- |
| `openrouter/anthropic/claude-sonnet-4.6` | Claude via OpenRouter |
| `openrouter/openai/gpt-4o` | GPT-4o via OpenRouter |
| `openrouter/google/gemini-2.5-pro` | Gemini via OpenRouter |
| `openrouter/auto` | Auto Router — OpenRouter picks a model from market spend |
| `openrouter/auto-beta` | Auto Router early-access track |

Use a model id your OpenRouter key can call. Settings lists a capped catalogue from `/v1/models` when `OPENROUTER_API_KEY` is set, and always offers `openrouter/auto`.

A bare `openrouter` (no `/`) is treated as an Anthropic model name and does
**not** auto-route. Prefer the alias `openrouter/auto`. The long form
`openrouter/openrouter/auto` also works (remainder is the catalogue id).

**Auto Router.** Chat Completions sends `model: openrouter/auto`. Every
OpenRouter call (Auto and pinned models) also sends `session_id` = the
workflow run id so later turns can sticky-route to the same provider, and Auto
can prefer the same model when the task type is unchanged. `cost_tier` /
`allowed_models` plugins are not schema fields yet — OpenRouter's default
band applies (roughly `low`). After each turn the run stores the model that
actually answered (`openrouter/{response.model}`).

**Cost.** Each OpenRouter response includes billed `usage.cost` (USD). When
every token-bearing message has a billed amount, run cost is the sum —
that is what spend limits see. Otherwise cost uses the organisation price
list keyed `provider=openrouter` and `model=<catalogue id>`. Do not reuse native
Anthropic / OpenAI prices. Auto Router has no fixed list price; do not
price `openrouter/auto` on the organisation list. Pin a model when you need a
predictable fallback rate.

### Azure OpenAI

| `model` value | Typical use |
| ------------- | ----------- |
| `azure/gpt-4o-prod` | Customer's Azure deployment named `gpt-4o-prod` |
| `azure/my-deployment-name` | Any deployment name from the Azure resource |

Use the deployment name from the Azure portal / AI Foundry, not the underlying
model id.

### Local runners

| `model` value | Typical use |
| ------------- | ----------- |
| `local_1/qwen3:8b` | Ollama (or llama.cpp) model on runner 1 |
| `local_2/gemma4:e4b` | Second parallel local runner |

Use the model id your runner actually serves (`ollama list`, llama.cpp `--model`).

---

## 5. Workflow frontmatter examples

```yaml
# Claude (explicit)
model: anthropic/claude-sonnet-4-6

# OpenAI
model: openai/gpt-4o

# Grok
model: xai/grok-4.5

# OpenRouter (remainder is the OpenRouter model id)
model: openrouter/anthropic/claude-sonnet-4.6

# OpenRouter Auto Router
model: openrouter/auto

# Azure OpenAI (remainder is the deployment name)
model: azure/gpt-4o-prod

# Local Ollama / llama.cpp (runner 1)
model: local_1/qwen3:8b
```

```yaml
# Bare model name → anthropic (default provider)
model: claude-haiku-4-5
```

---

## 6. LLM settings by provider

Optional LLM execution fields on the workflow (`max-turns`, `output-tokens`,
`caching`, `thinking`, …) are documented in **BRA201** §8.1.

| Setting | Anthropic | OpenAI | xAI | OpenRouter | Azure | Local |
| ------- | --------- | ------ | --- | ---------- | ----- | ----- |
| `max-turns` | Applied | Applied | Applied | Applied | Applied | Applied |
| `output-tokens` (retry caps) | Applied (`max_tokens` / `max_tokens` stop) | Applied (`max_tokens` / `finish_reason=length`) | Applied (same as OpenAI) | Applied (same as OpenAI) | Applied (same as OpenAI) | Applied (same as OpenAI) |
| `caching` | Applied | Ignored | Applied | Ignored | Ignored | Ignored |
| `thinking` / `thinking-budget` / `thinking-effort` | Applied | Ignored (request) | Ignored (request) | Ignored (request) | Ignored (request) | Ignored (request) |
| `auto-compaction` | Applied (server-side) | Applied (client-side via COMPACTION workflow) | Applied (client-side via COMPACTION workflow) | Applied (client-side via COMPACTION workflow) | Applied (client-side via COMPACTION workflow) | Applied (client-side via COMPACTION workflow) |

Unsupported fields are accepted on deploy and silently ignored at run time where
the table shows Ignored — they do not fail the run. Each provider applies
supported settings in its own native form (e.g. `caching: automatic` uses that
provider's automatic prompt-cache mechanism).

**Response-side reasoning (xAI / OpenAI-compatible):** Grok reasoning models often
return chain-of-thought in `message.reasoning_content` (especially on tool-call
turns where `content` is empty). That text is stored on the run so the UI can
show it alongside tool cards. Workflow `thinking*` frontmatter still does not
send Anthropic-style thinking request parameters to OpenAI / xAI / OpenRouter.

---

## 7. Native tools

Provider-native tools such as `web_search` and `web_fetch` are Anthropic-shaped
today. On OpenAI / xAI / OpenRouter / Azure / local-runner runs they are skipped rather than sent as
unknown capabilities. Declared and system tools still work on every provider.

---

## 8. Related skills

- **BRA201** §8 — workflow frontmatter, including LLM execution settings
- **BRA202** — `.env` upload, cloud LLM keys, `DEFAULT_LLM_MODEL`, and `LOCAL_LLM_N_BASE_URL`
- **BRA106** §8 — local Docker stack: Ollama env vars and `host.docker.internal`
- **BRA212** — managing LLM costs (caching, cheaper models, budgets, spend limits)
- **BRA403** — run telemetry (`gen_ai.request.model`, token fields, cost)
