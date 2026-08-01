---
name: Telos Brain Core Concepts
code: BRA101
version: 4
description: The building blocks of a Telos Brain, how they relate, and the two
  authoring styles used to define them — the essential mental model before
  authoring or deploying a brain.
---

# Telos Brain Core Concepts

A Telos Brain is **configuration-as-code**: a set of YAML and markdown files that the CLI parses and uploads to the Management API. The server never sees raw files — the CLI converts them to JSON and POSTs them.

Everything is wired together from a single entry-point manifest (`brain-compose.yml`).

---

## The building blocks

| Concept | What it is | Defined by |
|---|---|---|
| **Entities** | Top-level things the brain reasons about (e.g. `Application`). | Inline in `brain-compose.yml`. |
| **Units of work** | A scoped piece of work operating across entities (e.g. `Ticket`). | Inline in `brain-compose.yml`. |
| **Connectors** | Named integrations with external services (REST/MCP) — URL, auth type, declared credentials. | Connector YAML files (`connectors/{name}.yml`). See **BRA209**. |
| **Tools** | Callable actions — HTTP API, MCP, workflow, native, or in-brain system tools. | Tool-group folders (`tools.yml`). |
| **Skills** | Reusable knowledge and practices, grouped into skillbooks. | Skillbook folders (`skillbook.yml`). |
| **Blueprints** | Long-form scoped knowledge (vision, architecture, concepts…). | Blueprint folders (`blueprint.yml`). |
| **Workflows** | Runnable instructions that wire together tools and skills. | A single markdown file per workflow. |

---

## Two authoring styles

These are used deliberately and are not interchangeable:

- **YAML** — for *structured wiring*: manifests, endpoints, parameters, categories, scopes.
- **Markdown with YAML frontmatter** — for *long-form content*: skills, blueprint entries, and workflow instructions. The frontmatter carries metadata; the markdown body is the content itself. Skills may optionally declare a `tools:` list of tool names they need when loaded via `get_skill` (see BRA201 §6.3).

---

## Key principles

- **The compose file is the source of truth.** Anything not listed in `brain-compose.yml` is not deployed, even if the file exists on disk.
- **All paths are relative** to the manifest that references them (no `./` prefix needed).
- **Deploy order is fixed:** skills → connectors → tools → workflows → memory (blueprints) → entity types → unit-of-work types. Cross-references (e.g. workflows referencing skill codes) must resolve correctly, so the referenced resource must already exist.
- **The whole brain is parsed up front.** Any schema error fails the deploy before a single API call is made. Use `--dry-run` while authoring to validate without touching the API.

---

## Scope

Blueprints and units of work are **scoped** — they operate at a specific level of the brain's context:

| Scope | Meaning |
|---|---|
| `brain` | Applies across the entire brain (default). |
| `entity:<code>` | Scoped to a specific entity (e.g. `entity:application`). |
| `unitofwork:<code>` | Scoped to a specific unit of work (e.g. `unitofwork:ticket`). |

Entity and unit-of-work codes must match declarations in `brain-compose.yml`.

---

See **BRA201** for the full schema authoring reference, including file formats, field requirements, and the versioning rules.