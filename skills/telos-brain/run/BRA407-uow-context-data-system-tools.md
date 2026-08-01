---
name: Unit of Work Context and Data System Tools
code: BRA407
version: 2
description: The in-brain system tools for recording and reading Unit of Work
  telemetry — add_uow_context, add_uow_data, update_uow_data, list_uow_context,
  list_uow_data, get_uow_context and get_uow_data. All identity is by 8-character
  reference (never UUID). Tools operate only on the harness-injected active
  Unit of Work. For the Execution API HTTP surface, see BRA402.
tools:
  - add_uow_context
  - add_uow_data
  - update_uow_data
  - list_uow_context
  - list_uow_data
  - get_uow_context
  - get_uow_data
---

# Unit of Work Context and Data System Tools

BRA402 covers the **Execution API** HTTP surface for units of work (paths still
use Guids). This skill covers the complementary **AI-facing system tools**: a
running brain adding, listing, reading and updating context and data entries on
its **currently active** Unit of Work **in-process** — no webhook URL, API key,
or unit-of-work reference parameter.

**Identity rule:** every tool parameter and every list/detail field that
identifies a context or data entry uses an **8-character lowercase alphanumeric
reference** (`[a-z0-9]{8}`). Do **not** pass UUIDs.

**Active UoW only:** `UnitOfWorkId` is harness-injected from the run. If no unit
of work is in scope, every tool returns a clear plain-English error.

These are ordinary `system` tools (BRA201 §5.2). Declarations live under
`tools/uow/` in the brain schema. This skill lists them in frontmatter `tools:`
so a workflow that keeps them under `available-tools` can promote them via
`get_skill`.

---

## The seven tools

| Tool | Purpose | Key parameters | Returns |
|---|---|---|---|
| **`add_uow_context`** | Append narrative context | `title`, `source`; optional `message`, `date`, `tags` | Confirmation with new **reference** |
| **`add_uow_data`** | Append structured data | `source`, `type`, `body`; optional `date`, `tags`, `effort` (seconds) | Confirmation with new **reference** |
| **`update_uow_data`** | str_replace on Body | `uow_data_reference`, `str_replace_old`, `str_replace_new` | Confirmation |
| **`list_uow_context`** | List context (summary) | *(none)* | CSV keyed by `Reference` |
| **`list_uow_data`** | List data (summary) | *(none)* | CSV keyed by `Reference` |
| **`get_uow_context`** | Full context detail | `uow_context_reference` | Markdown |
| **`get_uow_data`** | Full data detail | `uow_data_reference` | Markdown |

Intended flow mid-run:

1. `add_uow_context` / `add_uow_data` as the agent records decisions or traces
2. Later turn: `list_uow_context` / `list_uow_data` to discover references
3. `get_uow_context` / `get_uow_data` for full content
4. `update_uow_data` when a data body needs a targeted edit

Context entries are **append-only** — there is no `update_uow_context`.

---

## Worked example

Mid-run, the agent records a decision and a tool trace:

```
add_uow_context
  title: Chose plan B
  message: Customer preferred the cheaper option.
  source: agent
  tags: decision,customer
→ Unit of work context created with reference ab12cd34.

add_uow_data
  source: tool
  type: quote_snapshot
  body: {"plan":"B","monthly":49}
  tags: quote
  effort: 120
→ Unit of work data created with reference ef56gh78.
```

Later in the same run:

```
list_uow_data
→ Reference,Date,Source,Type,Body,Tags,Effort
  ef56gh78,...,tool,quote_snapshot,"{""plan"":""B""...",quote,120

get_uow_data
  uow_data_reference: ef56gh78
→ full markdown including Tags, Effort and Body

update_uow_data
  uow_data_reference: ef56gh78
  str_replace_old: "monthly":49
  str_replace_new: "monthly":39
→ Unit of work data ef56gh78 body updated.
```

---

## `update_uow_data` str_replace rules

Matching is **exact-first**, then whitespace-tolerant fallback (TS624):

1. Exact literal match — if exactly one match, replace it
2. If exact finds zero matches, retry after normalising whitespace / zero-width
   characters
3. Multiple exact matches → hard error (no fallback)
4. Zero matches after both passes → hard error

Errors are plain English, e.g. `str_replace_old matched 0 times — no replacement made`.

---

## Wiring them into a workflow

1. Include the `tools/uow/` group in `brain-compose.yml`.
2. Either inject under workflow `tools:`, or list under `available-tools:` and
   load this skill (`BRA407`) via `get_skill` to promote them.

---

## Safety and scope

- **Brain-scoped, always.** The harness injects the brain; it is never a
  parameter.
- **Active Unit of Work only.** Cross-UoW references read as not found.
- **References only.** Agents must copy `Reference` values from list/get /
  create output — never invent Guids.
- Prefer these system tools over outbound HTTP to `POST /units-of-work/{id}/…`
  when already inside a ConversantFactory run.

---

## See also

- **BRA402** — Execution API entities and units of work (HTTP; Guid paths)
- **BRA201** §5.2 — system tools
- **BRA405** — inbox system tools (same reference / progressive-disclosure pattern)
- **WF-UNIT-OF-WORK-CONTEXT** — canonical unit-of-work workflow pattern
