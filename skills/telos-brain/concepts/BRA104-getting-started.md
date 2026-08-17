---
name: Getting Started
code: BRA104
version: 6
description: Interactive onboarding interview to configure a brain schema from
  scratch — entity type, unit of work type, blueprint categories, and skill
  categories. Use after brain init, before the first deploy. If the host
  application already exists, use BRA211 (auto-build) instead. For running the
  local Docker stack, use BRA106.
---

# Getting Started

This skill is an **AI-conducted interview**, not a document to read aloud.
Conduct it as a conversation: propose defaults, pause for the user's answer at
each decision, incorporate their choices, then move on. Do not dump all four
decisions as a static checklist.

**When to use:** after `brain init` has cloned the starter brain, and **before**
`brain deploy`. The goal is a configuration summary the agent (or user) can
apply to the schema files. To start the local Docker stack and point connectors
at a host app, load **BRA106**.

If you are building from an **existing application** rather than interviewing
the user, stop and load **BRA211** (auto-build) instead.

**Background (do not re-explain — load if needed):**

- **BRA106** — local Docker stack (`brain start` / deploy / host.docker.internal)
- **BRA201** — brain schema structure (`brain-compose.yml`, entities, units of
  work, blueprints, skillbooks)
- **BRA210** — LLM providers and example `model` codes for workflows
  (`anthropic/…`, `openai/…`, `xai/…`)
- **BRA208** — skill-book category design and numeric ranges
- **BRA102** — company brain model (brain-scoped vs entity-scoped memory)

After the brain is deployed and in use, point the user at the **Run** skills
(**BRA401** onwards) for the Execute API, entities/units of work at runtime,
inbox, run variables / `input-tools` (**BRA409**), file transcription
(**BRA410**), and related operations.

---

## Rules for the interviewer

1. Ask **one decision at a time**. Wait for a clear answer before continuing.
2. Always state the **default** and invite confirm / change.
3. Derive stable `code` values as lowercase, hyphen-free identifiers from the
   chosen names (e.g. `Clients` → `clients`, `Jobs` → `jobs`). Confirm the code
   with the user if the name is ambiguous.
4. For the unit of work, set `scope` to `entity:<entity-code>` using the entity
   code from decision 1.
5. For an entity-scoped blueprint, `scope.code` must equal the entity type code
   exactly.
6. **Do not deploy.** Do not invent schema files beyond producing the summary
   for the user/agent to apply. Deploy is the next onboarding step (README).
7. Do not delete `brain.lock` if it exists.

---

## Decision 1 — Entity settings

Pause and ask:

> What is the primary subject of work in this business?
>
> **Default:** Clients (`code: clients`) — a client of the business.
>
> Accept the default, or give a name (and optionally a short description). I will
> propose a `code` for `brain-compose.yml`.

Record: `name`, `code`, `description`.

---

## Decision 2 — Unit of work settings

Pause and ask:

> What is a discrete unit of work called in this business?
>
> **Default:** Jobs (`code: jobs`), scoped to the entity from step 1
> (`scope: entity:<entity-code>`).
>
> Accept the default, or give a name (and optionally a short description).

Record: `name`, `code`, `description`, `scope: entity:<entity-code>`.

---

## Decision 3 — Blueprint categories

**Before asking**, say clearly:

> Blueprint and skill categories are the highest-leverage schema decision.
> Learnings are routed into these categories — **generic categories produce
> generic learnings; specific, well-described categories produce useful
> memory.** Take a moment to tailor them to how this business actually thinks.

Then pause and propose the starter defaults (from the starter brain / BRA161):

**Company blueprint** (brain-scoped — shared across all entities):

| Category | Description |
|---|---|
| Business Concepts | Domain concepts and terminology specific to this business |
| Operations | How the business operates day to day, processes and workflows |
| Finance | Financial information, pricing, billing and financial processes |
| General | General knowledge about the business that does not fit other categories |

**Client / entity blueprint** (entity-scoped — isolated per entity instance).
Use the entity name from decision 1 in labels (default: Client):

| Category | Description |
|---|---|
| Client Context | Specific context, preferences and history for this client |
| Operations | Operational matters and processes specific to this client |
| Finance | Financial information, billing and commercial terms for this client |
| General | General notes about this client that do not fit other categories |

Ask the user to **confirm, add, rename, or remove** categories on each
blueprint. Keep overlapping names (`Operations`, `Finance`, `General`) spelled
identically across both blueprints unless the user deliberately renames them.
**Client Context** (or its renamed equivalent) belongs on the entity blueprint
only — not on the company blueprint.

Record the final category lists (name + description) per blueprint.

---

## Decision 4 — Skill categories

Pause and ask:

> How should the **business** skill book be organised?
>
> **Defaults** (see also BRA208 for designing categories and ranges):
>
> | Category | Index | Description |
> |---|---|---|
> | Sales | 100 | Sales processes, lead management and conversion |
> | Customer Management | 200 | Managing client relationships, communication and retention |
> | Operations | 300 | Day-to-day operational tasks and processes |
> | Finance | 400 | Financial processes, invoicing, billing and reporting |
> | General | 500 | General skills that span multiple areas |
>
> Confirm these, or customise names, descriptions, and indexes (use 100-step
> ranges: 100, 200, 300…). Leave `skills: []` empty for new categories.

Record the final skill category list (`name`, `description`, `index`).

---

## Output — configuration summary

After all four decisions, produce a single YAML summary the agent can apply to
the schema. Map answers into these blocks (substitute the user's choices; the
values below show the defaults):

```yaml
# === brain-compose.yml ===
entities:
  - code: clients
    name: Clients
    description: A client of the business

unitsofwork:
  - code: jobs
    name: Jobs
    scope: entity:clients

# === blueprints/company/blueprint.yml ===
# scope: { type: brain, code: null }
categories:
  - name: Business Concepts
    description: Domain concepts and terminology specific to this business
  - name: Operations
    description: How the business operates day to day, processes and workflows
  - name: Finance
    description: Financial information, pricing, billing and financial processes
  - name: General
    description: General knowledge about the business that does not fit other categories

# === blueprints/clients/blueprint.yml ===
# (folder name / scope.code must match entity code)
# scope: { type: entity, code: clients }
categories:
  - name: Client Context
    description: Specific context, preferences and history for this client
  - name: Operations
    description: Operational matters and processes specific to this client
  - name: Finance
    description: Financial information, billing and commercial terms for this client
  - name: General
    description: General notes about this client that do not fit other categories

# === skills/business/skillbook.yml (or the business skillbook in use) ===
categories:
  - name: Sales
    description: Sales processes, lead management and conversion
    index: 100
    skills: []
  - name: Customer Management
    description: Managing client relationships, communication and retention
    index: 200
    skills: []
  - name: Operations
    description: Day-to-day operational tasks and processes
    index: 300
    skills: []
  - name: Finance
    description: Financial processes, invoicing, billing and reporting
    index: 400
    skills: []
  - name: General
    description: General skills that span multiple areas
    index: 500
    skills: []
```

Then:

1. Offer to **apply** this summary to the schema files (edit
   `brain-compose.yml`, the company and entity `blueprint.yml` files, and the
   business `skillbook.yml`). If the entity code is not `clients`, rename or
   adjust the entity blueprint path/`scope.code` to match.
2. Remind the user the next onboarding step is **`brain deploy`** (do not run
   it unless they ask). For a local stack that is **`brain deploy --env local`**
   after **BRA106** (`brain start` + keys in `.env.local`).
3. For using the brain after deploy, point them to the **Run** category
   (**BRA401** — authentication conventions — is the usual starting point).
