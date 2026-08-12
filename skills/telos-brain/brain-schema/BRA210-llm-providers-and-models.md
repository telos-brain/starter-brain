---
name: LLM Providers and Models
code: BRA210
version: 2
description: Supported AI providers for workflow runs, the provider/model string
  format, example model codes, credential variable names, and which ConversantSettings
  apply per provider.
---

# LLM Providers and Models

Workflows choose an LLM with the optional frontmatter field `model`. The value
is a **`provider/model`** string. The platform resolves the provider prefix to a
conversant implementation and looks up the matching API key from brain
environment variables (see **BRA202**).

This skill is the authoring reference for which providers are supported and
which model codes are known to work. Pricing for cost calculation is managed
separately in organisation settings (`LlmPrices`); a missing price row leaves
`CostCents` null rather than failing the run.

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
| `claude-sonnet-4-6` (no prefix) | Treated as **anthropic** (default provider) |
| omitted / null | Default provider **anthropic**, default model `claude-sonnet-4-5` |

Both `/` and `\` are accepted as the separator. The provider prefix is
case-insensitive (`OpenAI/gpt-4o` → `openai`). The alias `claude/…` folds to
`anthropic`.

If the resolved provider has no matching `<PROVIDER>_API_KEY` on the brain, the
run cannot start.

---

## 2. Supported providers

| Provider prefix | Conversant | Credential (`.env`) | Notes |
| --------------- | ---------- | ------------------- | ----- |
| `anthropic` (alias `claude`) | Claude | `ANTHROPIC_API_KEY` | Default when `model` is omitted or unprefixed |
| `openai` | OpenAI Chat Completions | `OPENAI_API_KEY` | Also used for OpenAI embedding models when configured |
| `xai` | OpenAI-compatible Chat Completions at `api.x.ai` | `XAI_API_KEY` | Grok models; response `reasoning_content` mapped to Thinking |

Any other prefix is rejected at run time (`NotSupportedException`).

---

## 3. Example model codes

These are example workflow `model` values. Provider catalogues change over time —
use a model id your API key can call. Prefer an explicit `provider/` prefix.

### Anthropic / Claude

| `model` value | Typical use |
| ------------- | ----------- |
| `anthropic/claude-sonnet-4-6` | Default strong general / agentic workflows |
| `anthropic/claude-sonnet-4-5` | Platform default when `model` is omitted |
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

---

## 4. Workflow frontmatter examples

```yaml
# Claude (explicit)
model: anthropic/claude-sonnet-4-6

# OpenAI
model: openai/gpt-4o

# Grok
model: xai/grok-4.5
```

```yaml
# Bare model name → anthropic (default provider)
model: claude-haiku-4-5
```

---

## 5. ConversantSettings by provider

Optional LLM execution fields on the workflow (`max-turns`, `output-tokens`,
`caching`, `thinking`, …) are documented in **BRA201** §8.1.

| Setting | Anthropic | OpenAI | xAI |
| ------- | --------- | ------ | --- |
| `max-turns` | Applied | Applied | Applied |
| `output-tokens` (retry caps) | Applied (`max_tokens` / `max_tokens` stop) | Applied (`max_tokens` / `finish_reason=length`) | Applied (same as OpenAI) |
| `caching` | Applied | Ignored | Applied |
| `thinking` / `thinking-budget` / `thinking-effort` | Applied | Ignored (request) | Ignored (request) |
| `auto-compaction` | Applied (server-side) | Applied (client-side via COMPACTION workflow) | Applied (client-side via COMPACTION workflow) |

Unsupported fields are accepted on deploy and silently ignored at run time where
the table shows Ignored — they do not fail the run. Each provider applies
supported settings in its own native form (e.g. `caching: automatic` uses that
provider's automatic prompt-cache mechanism).

**Response-side reasoning (xAI / OpenAI-compatible):** Grok reasoning models often
return chain-of-thought in `message.reasoning_content` (especially on tool-call
turns where `content` is empty). The OpenAI-compatible conversant persists that
field as `WorkflowMessage.Thinking` so the run UI can show it alongside tool
cards. Workflow `thinking*` frontmatter still does not send Anthropic-style
thinking request parameters to OpenAI / xAI.

---

## 6. Native tools

Provider-native tools such as `web_search` and `web_fetch` are Anthropic-shaped
today. On OpenAI / xAI runs they are skipped rather than sent as unknown
capabilities. Declared and system tools still work on every provider.

---

## 7. Related skills

- **BRA201** §8 — workflow frontmatter, including LLM execution settings
- **BRA202** — `.env` upload and `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `XAI_API_KEY`
- **BRA403** — run telemetry (`gen_ai.request.model`, token fields, cost)
