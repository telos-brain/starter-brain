---
name: Brain Schema
code: BRA201
version: 52
description: Overview of the brain-schema format — mental model, directory
  layout, deploy, versioning, and which skill to load for each file type. Do
  not load this for field-level authoring; use BRA213–BRA217, BRA209, or
  BRA202 instead.
---

# Authoring a Telos Brain Schema

A Telos Brain is **configuration-as-code**: YAML and markdown files that
`brain deploy` parses and uploads to the Management API. The server never sees
the raw files — the CLI parses them into JSON and POSTs them.

This skill is the **map**. Load a focused skill for the file you are editing
instead of treating this page as the field reference.

| When you are authoring… | Load |
| --- | --- |
| `brain-compose.yml` (entities, units of work, variables, checkpoints, callback domains) | **BRA213** |
| Tool groups and tool YAML (`api` / `mcp` / `system` / `workflow` / `native`, parameters) | **BRA214** |
| Connectors | **BRA209** |
| Environment variables and secrets | **BRA202** |
| `skillbook.yml` and skill markdown | **BRA215** (design/ranges: **BRA208**) |
| `blueprint.yml` and blueprint entries | **BRA216** |
| Workflow markdown (frontmatter, triggers, tools, input-tools, LLM settings) | **BRA217** |
| Models and provider codes | **BRA210** |
| Template tags (`{{…}}`) | **BRA204** |
| Schema system tools (`create_skill`, `create_schema_file`, …) | **BRA203** |
| Learning-eval workflows | **BRA207** |

---

## 1. Mental model

A brain is composed from a single entry-point manifest (`brain-compose.yml`)
that **points to** self-contained definitions:

| Concept | What it is | Defined by |
| --- | --- | --- |
| **Entities** | Top-level things the brain reasons about (e.g. `Application`). | Inline in `brain-compose.yml` (**BRA213**). |
| **Units of work** | A scoped piece of work operating across entities (e.g. `Ticket`). | Inline in `brain-compose.yml` (**BRA213**). |
| **Connectors** | Named external-service integrations (URL, auth, declared params). | Connector YAML (`connectors/{name}.yml`). **BRA209**. |
| **Tools** | Callable actions (HTTP API, MCP, or in-brain system tools). | Tool-group folders (`tools.yml`). **BRA214**. |
| **Skills** | Reusable knowledge/practices, grouped into skillbooks. | Skillbook folders (`skillbook.yml`). **BRA215**. |
| **Blueprints** | Long-form scoped knowledge (vision, architecture, concepts…). | Blueprint folders (`blueprint.yml`). **BRA216**. |
| **Workflows** | Runnable instructions that wire together tools + skills. | A single markdown file per workflow. **BRA217**. |

Two authoring styles are used, deliberately:

- **YAML** for *structured wiring* — manifests, endpoints, parameters,
  categories, scopes.
- **Markdown with YAML frontmatter** for *long-form content* — skills, blueprint
  entries, and workflow instructions. The frontmatter carries metadata; the
  markdown body is the content itself.

All referenced paths inside a manifest are **relative to that manifest's own
folder** (no `./` prefix needed). Anything **not listed** in `brain-compose.yml`
is not deployed, even if the file exists on disk.

---

## 2. Directory layout

Names are conventional, not required — the compose file is the source of truth:

```
brain-schema/
  brain-compose.yml              # entry point — everything is referenced from here
  package.json                   # provides `npm run deploy`
  .env.example                   # template for deploy credentials (copy to .env)
  .gitignore                     # ignores .env, node_modules, brain.lock

  connectors/
    example-oauth2.yml           # one connector definition per file (BRA209)

  tools/
    tickets/
      tools.yml                  # tool group manifest (BRA214)
      add-ticket-comment.yml     # one tool definition per file

  skills/
    eng/
      skillbook.yml              # skillbook manifest (BRA215)
      backend/
        EP101-database-migrations.md

  blueprints/
    product-brain/
      blueprint.yml              # blueprint manifest (BRA216)
      vision-overview.md

  workflows/
    review-blueprint.md          # one workflow per markdown file (BRA217)
```

**Do not commit** `.env` (real credentials), `node_modules/`, or `brain.lock`
(local deploy state, akin to `terraform.tfstate`).

---

## 3. Deploy workflow

From the schema folder:

```bash
npm run deploy         # brain deploy .
npm run deploy:dry     # brain deploy . --dry-run  (parse + validate only, no API calls)
```

Key facts:

- The CLI resolves the compose file by looking for, in order:
  `brain-compose.yml`, `brain-compose.yaml`, `brain.yml`, `brain.yaml` — or you
  pass an explicit `.yml` path.
- On **first deploy** an instance name is required: `brain deploy . --instance <name>`.
  Thereafter it's remembered in `brain.lock`.
- **Instance name** must be a DNS-style slug: 3–63 chars, lowercase letters,
  digits and internal hyphens only, no leading/trailing hyphen.
- To duplicate an existing instance's configuration into a new slug (e.g.
  production → staging), use the Management API clone endpoint — see **BRA205**.
- To pull newer template configuration into a previously cloned instance without
  overwriting destination resources that are already ahead, use update-from —
  see **BRA206** (same version-precedence rule as §4).
- Credentials and destination load from `.env` or `.env.<env>` next to the
  compose file (`TELOS_BRAIN_ORG_API_KEY`, `TELOS_BRAIN_API_URL`; legacy
  `TELOS_ORG_API_KEY` / `TELOS_API_URL` still accepted). Use
  `brain deploy --env <local|dev|stage|prod>` to select a named file. Real
  environment variables override `.env`, so CI secrets always win. Full secret
  handling: **BRA202**.
- The **whole brain is parsed up front**, so any schema error fails the deploy
  before a single API call is made. Use `--dry-run` while authoring.
- Deploy order is fixed: **skills → connectors → tools → workflows → memory
  (blueprints)** (then entity / unit-of-work types). Connectors precede tools so
  tool definitions can reference connector names. Workflows reference
  skills/tools by code/name, so those codes must be correct.

Local Docker / host-app wiring: **BRA106**.

---

## 4. Versioning rules

The Management API versions every resource with a **single integer** and enforces
precedence: an incoming version must be **greater than or equal to** the stored
one, otherwise the upload is a **VersionConflict** and is skipped (not failed).
The same rule applies to CLI redeploy, the per-type upload endpoints, and
update-from-template (**BRA206**).

The file format is friendly about how you express versions; the CLI normalises
them all to the **leading integer** (the "major"):

| You write | Deployed as |
| --- | --- |
| `1` | `1` |
| `1.0` | `1` |
| `1.2.3` | `1` |
| *(omitted)* | `1` |
| `2.5` | `2` |

Implication: **equal majors redeploy and overwrite**; only a lower incoming major
is skipped. Bump the leading integer (e.g. `1.x` → `2`) when you want a clear
newer release marker, or when destination already holds a higher version.
Connectors have no version — they upsert-always (**BRA209**).

---

## 5. Field reference cheatsheet

Required fields, by file type (everything else is optional). Full rules live
on the skill in the last column.

| File | Required fields | Detail |
| --- | --- | --- |
| `brain-compose.yml` | `name` | **BRA213** |
| — entity | `name`, `code`; each `variables` item needs `key` | **BRA213** |
| Connector YAML | `name`, `url` or `url-env`, `auth-type`; each parameter needs `name` + `description` | **BRA209** |
| Tool group `tools.yml` | `name`, `description` | **BRA214** |
| Tool definition | `name`, `description`, exactly one of `api`/`mcp`/`system`/`workflow`/`native` | **BRA214** |
| Tool parameter | `name`, `description` (not used by `native` tools) | **BRA214** |
| `skillbook.yml` | `name`, `code`, `prefix`; each category needs `name` | **BRA215** |
| Skill markdown | frontmatter `name`, `code` + non-empty body | **BRA215** |
| `blueprint.yml` | `name`; each category needs `name`; `scope` code if entity/unitofwork | **BRA216** |
| Blueprint entry markdown | frontmatter `name`, `category` + non-empty body | **BRA216** |
| Workflow markdown | frontmatter `name`, `code` + non-empty body | **BRA217** |

---

## 6. Authoring checklist

When creating or extending a brain:

1. Add/confirm entities and units of work in `brain-compose.yml` (**BRA213**).
2. For each connector, create `connectors/{name}.yml` **and** list it under
   `connectors:` (**BRA209**). Put secret values in `.env`, not YAML (**BRA202**).
3. For each other capability, create its self-contained folder/file **and** add
   its path to the matching list in `brain-compose.yml` (unlisted files are
   ignored).
4. Keep relative paths correct — they resolve against the manifest's own folder.
5. Ensure cross-references resolve: workflow `tools` / `available-tools` → tool
   `name`s; skill `tools` → tool `name`s that the hosting workflow also lists
   under `available-tools` when you want mid-run promotion; workflow
   `injected-skills`/`available-skills` → skill `code`s; blueprint entry
   `category` → a manifest category; scope `code` → an entity/unit code.
6. Give every long-form markdown file a **non-empty body**.
7. Bump the **leading integer** of `version` on anything you change (connectors
   have no version — they upsert-always).
8. Use British English spelling in content.
9. Validate with `npm run deploy:dry` before deploying — it parses and validates
   the entire brain without touching the API.
