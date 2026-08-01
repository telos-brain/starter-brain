---
name: Skill Discovery System Tools
code: BRA411
version: 1
description: The in-brain system tools for discovering and loading skills —
  list_skills, find_available_skills and get_skill. list_skills returns the
  full SkillBook catalogue as CSV (code, name, description);
  find_available_skills ranks by semantic relevance; get_skill loads full
  instructions (and may promote tools).
tools:
  - list_skills
  - find_available_skills
  - get_skill
---

# Skill Discovery System Tools

Agents discover procedures by loading skills. These three system tools cover
**catalogue**, **semantic search**, and **detail** without outbound HTTP.

These are ordinary `system` tools (BRA201 §5.2). Declarations live under
`tools/system-tools/` in the brain schema. This skill lists them in frontmatter
`tools:` so a workflow that keeps them under `available-tools` can promote them
via `get_skill` (same pattern as BRA405 / BRA408).

**Scope:** always the current Brain (`BrainId` harness-injected). Never pass a
brain id.

---

## The three tools

| Tool | Purpose | Key parameters | Returns |
|---|---|---|---|
| **`list_skills`** | Full SkillBook catalogue | `skillbook_code` | CSV `code,name,description` |
| **`find_available_skills`** | Semantic top matches | `query`; optional `skillbooks` | Markdown in `<available_skills>` |
| **`get_skill`** | Full skill body (+ tool promotion) | `code` | Markdown skill content |

Intended flow:

1. Prefer **`list_skills`** when you already know the SkillBook code and need
   every skill (audit, enumeration, pick by exact title/code).
2. Prefer **`find_available_skills`** when you have a natural-language need and
   want the most relevant handful.
3. Call **`get_skill`** with a skill code to load full instructions before
   following them.

---

## `list_skills`

| | |
| --- | --- |
| **Purpose** | Return every active skill in one SkillBook |
| **Parameters** | `skillbook_code` (required) — e.g. `BRA`, `ENG` |
| **YAML** | `tools/system-tools/list-skills.yml` |

### Returns

CSV with header row:

```
code,name,description
BRA101,Telos Brain Core Concepts,Foundational theory of Telos Brain…
BRA201,Brain Schema,Designing and structuring brain schemas…
```

| Column | Source |
| --- | --- |
| `code` | `Skills.Code` |
| `name` | `Skills.Title` |
| `description` | `Skills.Description` (empty string when unset) |

Only **active** skills (`IsDeleted = 0`) are returned, ordered by `code`.
Embeddings are **not** required — skills without vectors still appear.

### Errors (plain English)

- Missing `skillbook_code`
- SkillBook not found in this Brain
- Invalid JSON arguments

### When to use

- Enumerate a book before creating a new skill (`create_skill` / BRA203)
- Confirm a code exists without loading full content
- Browse a small/medium book end-to-end

Do **not** use `list_skills` as a substitute for semantic search on large books
when you only need a few relevant skills — use `find_available_skills`.

---

## `find_available_skills`

| | |
| --- | --- |
| **Purpose** | Rank skills by cosine similarity to a query |
| **Parameters** | `query` (required); `skillbooks` optional comma-separated book codes |
| **YAML** | `tools/system-tools/find-available-skills.yml` |

Returns up to **10** matches as markdown sections inside `<available_skills>`.
Skills without an embedding (or when embeddings are unconfigured) are skipped /
unavailable — then prefer `list_skills`.

---

## `get_skill`

| | |
| --- | --- |
| **Purpose** | Load one skill's full markdown instructions by code |
| **Parameters** | `code` (required) — e.g. `BRA411` |
| **YAML** | `tools/system-tools/get-skill.yml` |

May promote tools listed in the skill's frontmatter `tools:` into the run when
those tools are in the workflow's `available-tools` pool (BRA201 §6.3).

---

## Wiring them into a workflow

1. Include the `tools/system-tools/` group in `brain-compose.yml`.
2. Either inject under workflow `tools:`, or list under `available-tools:` and
   load this skill (`BRA411`) via `get_skill` to promote them.

---

## Safety and scope

- **Brain-scoped.** Never pass Brain id or SkillBook id — only `skillbook_code`
  / skill `code` strings.
- **Active rows only.** Soft-deleted skills and books are excluded.
- **Progressive disclosure.** List/search first; `get_skill` for full body.

---

## See also

- **BRA201** §5.2 / §6 — system tools and skill file format
- **BRA203** — `create_skill` and schema system tools
- **BRA208** — designing skill books and category ranges
- **BRA103** — what is a skill book
