---
name: Schema System Tools
code: BRA203
version: 4
description: The in-brain system tools that let a running brain inspect, edit,
  and create its own configuration-as-code schema — list_schema_files,
  search_schema_files, get_schema_file, update_schema_file, create_skill and
  create_schema_file. Covers what each does, how to wire them into a workflow,
  the schema file model they expose, and the safety guarantees (brain scoping,
  exact-one-match edits, automatic versioning, create paths).
# Tools this skill needs when loaded via get_skill. The hosting workflow must
# list them under available-tools (see WF-SKILL-UPDATE) for mid-run promotion.
tools:
  - list_schema_files
  - search_schema_files
  - get_schema_file
  - update_schema_file
  - create_skill
  - create_schema_file
---

# Schema System Tools

BRA201 covers authoring the schema as files and deploying them with the CLI. This
skill covers the complementary runtime capability: a **running brain editing its
own schema**. Six `system` tools expose the brain's configuration-as-code as a
virtual filesystem — list, search, read, edit and create workflow, skill, tool,
blueprint and connector files — without any outbound HTTP call. This is how a
learning-review workflow applies an approved learning back into the brain.

These are ordinary `system` tools (see BRA201 §5.2): they are declared with a
`system:` block and added to a workflow's tool lists like any other tool. They
are executed in-process by the tool router — no webhook URL, MCP server or API
key is involved.

This skill declares those tools in its own frontmatter `tools:` list
(BRA201 §6.3). When a workflow keeps them under `available-tools` and the agent
calls `get_skill` for `BRA203`, matching tools are promoted into the run's Claude
declarations for the rest of that run — see `WF-SKILL-UPDATE` for the wiring.

---

## The six tools

| Tool | Purpose | Parameters | Returns |
|---|---|---|---|
| **`list_schema_files`** | List every schema file for the brain. | none | CSV: `path,type,code` |
| **`search_schema_files`** | Filter files whose **code** or **title** contains a query (case-insensitive substring). | `query` | CSV: `path,type,code,title` |
| **`get_schema_file`** | Read the full canonical content (YAML or markdown) of one file. | `path` | file content |
| **`update_schema_file`** | Apply a targeted string-replace edit to one file and persist it. | `path`, `str_replace_old`, `str_replace_new` | success / error |
| **`create_skill`** | Create a skill in a SkillBook category with an auto-assigned code. | `skillbook_code`, `category_title`, `title`, `description`, `content` | new skill code / error |
| **`create_schema_file`** | Create any supported schema file via the deploy parse/persist path. | `path`, `content` | resource code / error |

The intended flow for edits is **list/search → get → update**: discover a path,
read the file to copy the exact text, then edit it. For creates, prefer
**`create_skill`** (skills) or **`create_schema_file`** (other types).

---

## The schema file model

Every file is addressed by a stable **path** and carries a **type** token and a
stable **code**. The path prefix matches the resource family, e.g.
`workflows/wf-review.md`, `skills/eng/EP101.md`, `tools/core/load-customer.yml`,
`connectors/my-connector.yml`.

The `type` column uses these tokens (leaf resources use the resource token to
match the path prefix; container manifests use a distinct token so a listing
never conflates a manifest with a resource of the same family):

| Token | File |
|---|---|
| `manifest` | the generated `brain-compose.yml` |
| `workflow` | a workflow markdown file |
| `skill` | a skill markdown file |
| `skillbook` | a skillbook manifest |
| `tool` | a tool definition |
| `toolgroup` | a tool-group manifest |
| `blueprint` | a blueprint entry |
| `blueprint-manifest` | a blueprint manifest |
| `connector` | a connector definition (`connectors/{name}.yml` — see **BRA209**) |

---

## `list_schema_files`

Takes no parameters. Returns a flat CSV (`path,type,code`) of the root manifest
plus one row per workflow, skill, tool, blueprint entry and connector. Use
`search_schema_files` when you also need titles.

## `search_schema_files`

Requires `query`. Matches the query as a case-insensitive substring against each
file's **code** and **title**, returning CSV with an extra `title` column. A
query with no matches returns just the header row — this is **not** an error.

## `get_schema_file`

Requires `path` (obtained from `list_schema_files` or `search_schema_files`).
Returns the full canonical YAML or markdown. An unknown path returns a
descriptive error naming `list_schema_files` as the way to find valid paths,
rather than empty content.

## `update_schema_file`

Requires `path`, `str_replace_old` and `str_replace_new`.

- `str_replace_old` must be **copied verbatim from `get_schema_file` output** and
  must match **exactly once**. Include enough surrounding context to be unique.
  The match is made after whitespace / zero-width normalisation, but the
  replacement is applied to the original content so formatting is preserved.
- `str_replace_new` is the replacement text. Supply an **empty string to delete**
  the matched text (the key must still be present).
- On success the file is persisted through the brain's normal deploy path, which
  **bumps the file's version automatically** — no manual version bump is needed.
- The generated `brain-compose.yml` manifest **cannot** be edited this way.

Zero-match, multi-match, unknown-path and parse errors are returned verbatim so
the agent can correct and retry.

## `create_skill`

Requires `skillbook_code`, `category_title`, `title`, `description`, and
`content`. Resolves the SkillBook by code and a category by title
(case-insensitive), auto-assigns the next skill code in that category range
(soft-deleted codes stay reserved), defaults version to 1, and returns the new
code (e.g. `BRA209`). Prefer this over `create_schema_file` for skills.

On failure, load **BRA208** (skill-book design) and **BRA201** §6 (skill format)
before retrying. Example tool definition: `tools/brain-schema/create-skill.yml`.

## `create_schema_file`

Requires `path` and `content` (full file). Supported paths:

| Path | Type |
|---|---|
| `workflows/{code}.md` | Workflow |
| `tools/{group}/{name}.yml` | Tool |
| `skills/{book}/{code}-{title}.md` | Skill |
| `blueprints/{code}/{title}.md` | Blueprint entry |

Rejected: `brain-compose.yml` and generated group manifests
(`tools/*/tools.yml`, `skills/*/skillbook.yml`, `blueprints/*/blueprint.yml`).
`connectors/{name}.yml` is **not** creatable via this tool yet — add the file
and compose entry, then `brain deploy` (or edit an existing connector with
`update_schema_file`). See **BRA209**.

Omit `version` to default to 1. Version conflicts return a clear error when the
incoming version is less than the stored version.

On failure, the tool's `error-markdown` (see
`tools/brain-schema/create-schema-file.yml`) names the skills that define each
file type's format — typically **BRA201**, plus **BRA208** for skills and
**BRA204** when template tags are involved. Load those with `get_skill`, fix the
content, and retry.

---

## Wiring them into a workflow

Declare each as a tool group of `system` tools, then add them to the workflow.
Two patterns are supported:

1. **Injected** — list them under the workflow's `tools:` so they are available
   every turn.
2. **Available + skill promotion** — list them under `available-tools:`, declare
   the same names on this skill's frontmatter `tools:`, and ensure the workflow
   can call `get_skill` / `find_available_tools`. Loading `BRA203` promotes the
   matching tools for the rest of that run (`WF-SKILL-UPDATE` uses this pattern).

A minimal tool definition:

```yaml
# tools/brain-schema/create-schema-file.yml
name: create_schema_file
version: 1
description: >-
  Creates a schema file at the given path with full content…
system:
  tool: create_schema_file
parameters:
  - name: path
    param: path
    description: The schema file path, e.g. "workflows/wf-example.md".
    type: string
    required: true
  - name: content
    param: content
    description: Full file content (YAML or markdown with frontmatter).
    type: string
    required: true
error-markdown: |
  Could not create schema file: {{result.result}}
  Load BRA201 (file formats) and BRA203 (this skill) via get_skill, then retry.
```

`create_skill` and `create_schema_file` example definitions (with full
`error-markdown` skill pointers) live under `tools/brain-schema/`.

---

## Safety and scope

- **Brain-scoped, always.** The brain the run belongs to is injected by the
  harness; it is never a tool parameter. Files from other brains are never
  listed, read, edited or created.
- **Exact-one-match edits.** `update_schema_file` refuses zero-match and
  multi-match edits, so an edit can never silently hit the wrong text.
- **Manifest is read-only.** `brain-compose.yml` is generated and rejected by
  both `update_schema_file` and `create_schema_file`.
- **Create via dedicated tools.** Prefer `create_skill` / `create_schema_file`.
  The older empty-`str_replace_old` workaround on `update_schema_file` is a
  fallback only for types those tools do not cover.
