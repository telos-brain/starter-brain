---
name: Managing LLM Costs
code: BRA212
version: 4
description: How to keep LLM spend down in a Telos Brain — aim for 80% cache
  reads (or turn on automatic caching), convert JSON tool data to markdown or
  CSV, treat tool definitions as mini-skills to cut retries, compact older
  context, pick a cheaper model (Grok), enforce token budgets plus daily spend
  and run/turn caps, and use progressive disclosure so the agent discovers
  skills and tools as needed. Use when authoring or reviewing workflows, tools,
  or brain-compose.yml for cost, tokens, cache, compaction, or model choice.
tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
---

# Managing LLM Costs

Tokens are an allocation, not an entitlement (**BRA105**). The schema is how
you enforce that: workflow frontmatter, tool YAML, and `brain-compose.yml`.
Seven levers do most of the work. Apply them together — a cheap model with
uncached, JSON-heavy, retry-prone tools still burns money.

Measure after you change something. Run telemetry (**BRA403**) reports
`gen_ai.usage.input_tokens`, `output_tokens`, `cache_read_input_tokens`,
`cache_creation_input_tokens`, `telos.turns.used` / `telos.turns.max`, and
`CostCents`. For OpenRouter, `CostCents` is the sum of billed `usage.cost`
when every token-bearing message has one; otherwise it uses `LlmPrices` and
is null when the organisation has no row for that model. Spend limits only
accumulate when prices exist (or billed cost was persisted).

---

## 1. Aim for 80% cache reads (or at least turn on automatic caching)

Cache-read tokens are billed at a small fraction of uncached input. Cache
*writes* cost more than a normal input token. The first turn of a run creates
the cache; later turns should *read* it. **Aim for 80% of prompt tokens to be
cache reads** on multi-turn workflows:

```
cache_read / (input + cache_read + cache_creation) ≥ 0.80
```

Those fields are the BRA403 span attributes `gen_ai.usage.input_tokens`,
`cache_read_input_tokens`, and `cache_creation_input_tokens`.

### Schema

On every multi-turn / conversational workflow, set:

```yaml
caching: automatic
```

| `caching` value | Behaviour |
|---|---|
| `automatic` | Provider automatic prompt cache. **Use this.** |
| omitted | Historic hand-crafted per-block `cache_control` markers |
| `none` | Suppress all cache markers — only for short, one-shot jobs (e.g. `WF-COMPACT`) |

Applied on Anthropic and xAI; OpenAI ignores it (**BRA210** §5).

- **Claude:** top-level `cache_control` — the API places the breakpoint.
- **xAI / Grok:** `x-grok-conv-id` sticky-routing header keyed by
  `WorkflowRunId`, which is what makes Grok cache hits possible.

### How to actually hit 80%

The cache prefix must be **byte-stable** across turns of the same run:

- Keep `system-prompt-code`, `tools`, and `injected-skills` fixed. Changing
  any of them busts the prefix and forces a cache write.
- Prefer `available-skills` / `available-tools` over stuffing the prefix
  (see §7). Discovery tools load depth on demand; the prefix stays small
  and stable.
- Fetch structured context with `input-tools` (**BRA201** §8.0a) rather than
  rewriting the system prompt per run. Pre-called results land in a user
  block *after* the cacheable prefix.
- Reuse chat sessions (`POST /workflows/{code}/run/sync`, then continue the
  same run — **BRA403**). A new run starts a new cache.
- Do not set `caching: none` on chat or agentic loops.
- Put volatility last: the latest user message and fresh tool results. The
  conversant already orders messages for cache; do not fight that by
  injecting changing text into the system prompt.

If cache-read share stays low after `caching: automatic`, the prefix is
churning — look at injected skills, tool lists, and per-run system text
before blaming the provider.

---

## 2. Convert JSON data to markdown or csv

JSON is expensive in context: braces, quotes, and repeated keys are billed
on the turn they arrive *and* on every later turn they remain in history.
These APIs are for an LLM, not a SPA (**BRA211**).

### Host tool APIs (preferred)

Return the shape the model should see:

| Payload | Format |
|---|---|
| A list of records | **CSV** (header row + one row per item). References, not UUIDs. |
| A single detailed object | **Markdown** (headings, labelled fields, short prose). |
| An error | One or two sentences, plus the skill code to load. |

Do not default to JSON “because it is an API”.

```
# List — CSV
US1,Jane Chen,jane@acme.com,Director
US2,Sam Reid,sam@acme.com,Engineer

# Detail — markdown
## Ticket TK41
**Status:** Open
**Owner:** US1
```

### Tool `response-markdown` (when the API still returns JSON)

Declared tools can reshape the JSON body before the model sees it
(**BRA204** §3.5). Use YAML block scalars. The `result` scope is **flat**
(nested objects/arrays are stringified):

```yaml
name: list_widgets
description: >-
  Lists widgets for the current entity as a compact table. Use when you
  need names and references, not a full dump.
api:
  method: GET
  path: https://api.example.com/widgets
parameters:
  - name: organisation_id
    param: organisationId
    entity: organisationId
    description: Injected organisation id — hidden from the model.
response-markdown: |
  | Reference | Name | Status |
  |---|---|---|
  | {{result.reference}} | {{result.name}} | {{result.status}} |
error-markdown: |
  Could not list widgets: {{result.error}}

  Load **BRA201** §5 if the call shape looks wrong. Do not retry with a
  different parameter name.
```

Rules:

- Omit `response-markdown` / `error-markdown` only when the HTTP body is
  already markdown or CSV.
- Never echo the raw JSON blob “for completeness”.
- `input-tools` results also sit in context and are capped at 200,000
  characters — format those payloads the same way, or the first turn is
  already bloated.

---

## 3. Improve tool definitions to reduce failures/retries

**Tool definitions are mini-skills.** They sit in the system prompt (and
therefore in the cache prefix). Prefer fixing the tool YAML — description,
parameter names/descriptions, `response-markdown`, `error-markdown` — over
teaching the tool only in a skill (**BRA105**, **BRA201** §5).

Every failed call is a billed turn (full prompt + output) plus another billed
turn for the retry. A vague tool that the model mis-invokes twice can cost
more than the successful work.

### What to put on the tool

| Field | Cost job |
|---|---|
| `description` | When to call, when *not* to call, and one example value. Tight, but sufficient. |
| Parameter `description` | Format, example, and allowed values. The model only sees params that are not bound to `value` / `secret` / `entity` / `unitofwork` / `input`. |
| Parameter `type` | `int` / `decimal` / `date` / `datetime` so the router coerces before dispatch — parse failures become a clear tool error instead of a bad HTTP call. |
| Hidden bindings | Tenant ids, API keys, and run inputs are **not** LLM-facing. Bind them; do not ask the model to copy a UUID. |
| `response-markdown` | Compact success text. Tell the model what happened and the reference to use next. |
| `error-markdown` | Name the skill to load (`get_skill BRA201`), then stop. Do not invite a guessed retry. |

Canonical pattern — `create_skill` (**BRA203**): success returns the new
code; failure points at **BRA203** / **BRA208** / **BRA201** instead of
dumping a parser stack.

```yaml
error-markdown: |
  Could not create skill: {{result.result}}

  Before retrying, load with `get_skill`:

  - **BRA203** — `create_skill` parameters
  - **BRA208** — categories and ranges
  - **BRA201** — skill file format

  Do not pass `brain_id` — it is harness-injected.
```

### Authoring checks

- One clear `name` the model will actually emit (`list_widgets`, not
  `widgetSvc_listV2`).
- Hide everything the model cannot know or must not see.
- Keep the description short — it is billed on every cache write of the
  prefix.
- If evals show the same argument mistake, fix the parameter description
  or `error-markdown`, not the workflow instructions.

---

## 4. Use compaction to summarise older context and tool use

Long runs re-send the whole transcript, including bulky tool results.
Compaction summarises **older turns and tool use** and keeps recent turns
intact so the active prompt stays inside the window — and inside the bill.

### Automatic (preferred on chat / agentic loops)

```yaml
auto-compaction: 100000
```

| Field | Default | Meaning |
|---|---|---|
| `auto-compaction` | off | Positive integer = input-token trigger. Claude API enforces a **50,000** minimum; OpenAI / xAI use the same threshold client-side. |

- **Claude:** server-side `compact_20260112`.
- **OpenAI / xAI:** client-side — the brain's `COMPACTION` workflow runs
  when the prompt-token threshold is reached (**BRA210** §5).

### On demand

Expose the `compact_context` system tool on long conversational workflows.
It has no parameters; it summarises the current run's history. Failure is
fail-open (context unchanged). The tool definition lives at
`tools/system-tools/compact-context.yml`.

### The COMPACTION workflow

At most **one** active `type: COMPACTION` workflow per brain. Canonical
file: `workflows/wf-compact.md` (`WF-COMPACT`). Author it as a cheap,
one-shot summariser:

```yaml
name: Compact Context
code: WF-COMPACT
type: COMPACTION
model: anthropic/claude-haiku-4-5   # or xai/grok-4.3 — keep this cheap
output-tokens: 2048
max-turns: 1
caching: none
```

Instructions should preserve goals, decisions, identifiers, tool outcomes
that changed state, and open questions — and return **plain text**, not
fenced markdown. No tools.

Without this workflow, OpenAI / xAI auto-compaction and `compact_context`
have nothing to call.

---

## 5. Use a cheaper model (Grok is a great choice for cheaper without compromising quality)

Workflows choose a model with `model: provider/model-name` (**BRA210**).
Omit it and you get Anthropic `claude-sonnet-4-5` — a strong default, not
a cheap one.

**Grok is a great choice for cheaper without compromising quality.** Set
`XAI_API_KEY` in the brain `.env` (**BRA202**) and:

```yaml
model: xai/grok-4.5
```

| `model` | Typical use |
|---|---|
| `xai/grok-4.5` | Flagship Grok — agentic / coding, usually cheaper than Claude Sonnet at similar quality |
| `xai/grok-4.3` | Lower-cost long-context Grok — good for COMPACTION, classification, simple `TOOL` workflows |
| `anthropic/claude-haiku-4-5` | Fast, cheap Claude turns (compaction, short Q&A) |
| `anthropic/claude-sonnet-4-6` | Hard agentic work that still needs Claude |
| `anthropic/claude-opus-4-5` | Rare — highest Claude capability, highest spend |

Match the model to the workflow, not the brain:

- Chat / general agent: `xai/grok-4.5` unless you have a Claude-specific
  reason (native `web_search` / `web_fetch`, thinking modes).
- Compaction, ask-question, routing: Haiku or `xai/grok-4.3`.
- Do not put Opus on a heartbeat or eval loop.

Native tools (`web_search`, `web_fetch`) are Anthropic-shaped and are
skipped on OpenAI / xAI (**BRA210** §6). If a workflow needs them, keep
Claude for that workflow only.

Organisation `LlmPrices` must include the model you pick. A missing price
row leaves `CostCents` null — telemetry still shows tokens, but daily /
monthly spend limits cannot see the spend. For OpenRouter, add rows with
**provider** `openrouter` and **model** the catalogue id
(`anthropic/claude-sonnet-4.6`, not the native Anthropic hyphenated id).
OpenRouter runs that persist billed `usage.cost` do not need a matching row
for `CostCents` to populate. Local runners and Azure OpenAI are bring-your-
own-billing: do not seed `LlmPrices` for them; `CostCents` stays null.
Platform credits still apply via `RunSeconds`.

---

## 6. Use token budgets, daily cost limits and run+turn limits

Give every workflow a budget and refuse work that exceeds it. Raise a
limit only when the work needs more, never because the model asked
(**BRA105**).

### Per-workflow (frontmatter — **BRA201** §8.1)

```yaml
model: xai/grok-4.5
caching: automatic
auto-compaction: 100000
output-tokens: 2048, 4096          # start small; retry only if truncated
max-turns: 10                      # 10 is the engine default — do not raise to 50 without cause
thinking: adaptive
thinking-effort: low               # real thinking-cost lever on Claude (billed tokens, not the ceiling)
# thinking-budget: 1024            # only if thinking must not steal the reply budget
max-runs-per-hour: 50              # rolling hour; raise only for batch system workflows (e.g. WF-EVAL)
max-recursion-depth: 5
```

| Field | Default | Cost role |
|---|---|---|
| `output-tokens` | `4096` (one attempt) | Ceiling per attempt. A list (`2048, 4096, 16384`) is ordered retries when a turn stops at `max_tokens`. Failed truncated attempts are still billed. Prefer a short first cap. |
| `max-turns` | `10` | Tool-use loop cap. The run **Fails** when it is exhausted. Chat (`WF-CHAT`) may need more; a lookup workflow should stay at 3–8. |
| `thinking` / `thinking-effort` / `thinking-budget` | thinking off | Claude-only at request time (**BRA210** §5). `thinking-effort` is the spend lever — Anthropic bills tokens *generated*, not `output-tokens`. Prefer `adaptive` + `low` over `extended`. |
| `max-runs-per-hour` | `50` | Rolling-hour cap per workflow. Heartbeats and eval batches set this *up*; user-facing tools should stay low. |
| `max-recursion-depth` | `5` | Caps `run_workflow` / workflow-tool nesting before a child `WorkflowRun` is created. |
| `session-timeout` | `30` (minutes) | Closes idle chat sessions so they stop accruing and become eligible for eval (**BRA201** §8.2). |

`output-tokens` and `max-turns` apply on every provider. Do not copy
`WF-CHAT`'s `max-turns: 50` onto a triggered or tool workflow.

### Per-brain spend ceilings (`brain-compose.yml`)

Daily and monthly LLM spend limits are optional compose fields. Values are
**USD**; deploy stores them as cents. Omit or leave blank for no limit.
Omitted keys on a later deploy **clear** a previously set limit.

```yaml
name: kappa
daily-limit-usd: 5.00
monthly-limit-usd: 50.00
```

| Field | Effect |
|---|---|
| `daily-limit-usd` | Refuse new runs when today's UTC spend (`CostCents` sum) meets or exceeds the ceiling. Error `daily_spend_limit_reached`. |
| `monthly-limit-usd` | Same for the UTC calendar month. Error `monthly_spend_limit_reached`. Daily is checked first. |

Null ceilings are skipped (not treated as zero). Limits apply at run
start — a run already in flight is not killed mid-turn. They only work
when `LlmPrices` can price the models you use.

There is no per-minute run cap in the schema today; use `max-runs-per-hour`
for rate.

---

## 7. Use progressive disclosure to allow the agent to discover skills and tools as needed

Injecting every skill and every tool on every turn is the most expensive
default. Descriptions live in the system prompt (and the cache prefix). A
chat workflow with thirty tools and ten injected skills pays that tax on
**every** cache write — and the model still has to read it all.

Progressive disclosure (**BRA103**, **BRA105**) is the opposite: put a
short catalogue in reach, load the full body only when the work needs it.

### Skills

| Frontmatter | What the model sees | Cost |
|---|---|---|
| `injected-skills` | Full skill body inlined into the prompt on every turn | High — reserve for principles you always want (e.g. `BRA105`) |
| `available-skills` | Code is in the searchable pool; body loads only via `get_skill` | Low until loaded |

Discovery tools — inject these, they are small (**BRA411**):

| Tool | Use |
|---|---|
| `find_available_skills` | Semantic search: natural-language `query` → top matching codes |
| `list_skills` | Full SkillBook catalogue as CSV (`code,name,description`) |
| `get_skill` | Load the full markdown body for one `code` |

Skill content should itself disclose progressively: keep each skill short,
split when it covers too much, and point at related codes (`see **BRA201**
§6`) instead of inlining. That is how further depth is loaded on demand.

### Tools

Tool definitions sit in the system prompt. A workflow with only `tools:`
and no `available-tools` injects **every** listed tool on every turn
(**BRA201** §8).

| Frontmatter | What the model sees | Cost |
|---|---|---|
| `tools` | Full tool schema (name, description, parameters) every turn | High — inject only discovery tools plus tools used on most turns |
| `available-tools` | Permission envelope only — not in the prompt until surfaced | Low until discovered or promoted |

How an available tool becomes callable (**BRA201** §6.3):

1. **`find_available_tools`** — semantic search over the workflow's
   `available-tools` pool (`query` → names and descriptions). Inject this
   tool.
2. **`get_skill` promotion** — if the loaded skill lists the tool under
   its own `tools:` **and** the workflow lists it under `available-tools:`,
   it is promoted for the rest of **this run only**. The workflow still
   owns the permission envelope: a skill cannot grant a tool the workflow
   did not list.
3. A tool in the skill but missing from `available-tools` is silently
   skipped.

Canonical pattern: `WF-SKILL-UPDATE` injects `find_available_skills`,
`get_skill`, and `find_available_tools`; schema edit tools stay under
`available-tools` until `BRA203` is loaded.

```yaml
tools:
  - find_available_skills
  - get_skill
  - find_available_tools
available-tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
  - create_skill
injected-skills:
  - BRA105
available-skills:
  - BRA201
  - BRA203
  - BRA208
  - BRA212
```

### Authoring checks

- Inject discovery tools on conversational / edit workflows. Without them,
  `available-*` pools are unreachable.
- Inject a skill only when it must be in every turn (`BRA105` on schema
  edit). Everything else is `available-skills`.
- Inject a tool only when most turns will call it. Domain tools belong in
  `available-tools` and on the skill that needs them.
- Do not give every workflow every tool. Narrow `available-tools` is still
  a permission boundary, not a prompt dump.
- Skills that need tools must declare them in frontmatter `tools:` or
  promotion never happens.

---

## 8. Worked example — a cost-aware chat workflow

```yaml
---
name: Chat
code: WF-CHAT
type: RUNNABLE
model: xai/grok-4.5
caching: automatic
auto-compaction: 100000
output-tokens: 2048, 4096
max-turns: 20
max-runs-per-hour: 50
session-timeout: 15
system-prompt-code: WF-SYSTEM-PROMPT
tools:
  - find_available_skills
  - get_skill
  - find_available_tools
  - compact_context
available-tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
available-skills:
  - BRA212
---
```

And in `brain-compose.yml`:

```yaml
daily-limit-usd: 10.00
monthly-limit-usd: 100.00
```

Checklist when reviewing a brain for cost:

- [ ] Multi-turn workflows have `caching: automatic` (not `none`)
- [ ] Cache-read share on busy chats is trending toward 80% (**BRA403**)
- [ ] List tools return CSV; detail tools return markdown (or
      `response-markdown` reshapes JSON)
- [ ] Tool `error-markdown` names a skill to load — no blind retries
- [ ] Hidden bindings for secrets, entity, unit-of-work, and input
- [ ] One `type: COMPACTION` workflow exists; chat sets `auto-compaction`
- [ ] `model` is Grok / Haiku unless Claude is required
- [ ] `output-tokens` starts small; `max-turns` matches the job
- [ ] `daily-limit-usd` / `monthly-limit-usd` are set on the compose file
- [ ] Organisation `LlmPrices` includes every model the brain calls (OpenRouter: provider `openrouter`, catalogue id as the model)
- [ ] Conversational workflows inject discovery tools (`find_available_skills`,
      `get_skill`, `find_available_tools`) and keep domain skills/tools in
      `available-skills` / `available-tools`

---

## 9. Related skills

- **BRA103** — skill codes and progressive disclosure
- **BRA105** — budget principle and “keep each skill short” (always inject when editing the brain)
- **BRA201** §5 — tool YAML; §6.3 skill-declared tool promotion; §8 `tools` / `available-tools`; §8.0a `input-tools`; §8.1 LLM execution settings
- **BRA202** — `XAI_API_KEY` / `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`
- **BRA203** — schema tools (`update_schema_file` to apply these fields)
- **BRA204** §3.5 — `{{result.*}}` in `response-markdown` / `error-markdown`
- **BRA210** — provider / model strings and which settings each provider honours
- **BRA211** — token-efficient host APIs (CSV / markdown, references not UUIDs)
- **BRA403** — telemetry for tokens, cache, turns, and cost
- **BRA411** — `list_skills`, `find_available_skills`, `get_skill`
