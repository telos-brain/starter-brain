---
name: Skills File Format
code: BRA215
version: 1
description: How to author skillbook.yml and skill markdown files — categories,
  skill codes, frontmatter, and skill-declared tools for mid-run promotion.
  Load this when creating or editing a skill file. For category design and
  ranges see BRA208.
---

# Skills File Format

Skills live in **skillbooks**. A skillbook manifest declares categories; each
category lists skill markdown files. Each skill's own metadata lives in its
markdown frontmatter. Overview and versioning: **BRA201**. How to design
categories and ranges: **BRA208**. What a skill book is: **BRA103**. Workflow
`available-tools` promotion: **BRA217**.

## 1. Skillbook manifest (`skills/<book>/skillbook.yml`)

```yaml
name: Engineering Practices            # REQUIRED (uploaded as the book title)
code: ENG                              # REQUIRED (unique skillbook code)
prefix: EP                             # REQUIRED (skill code prefix, e.g. EP101)
version: 1.0                           # optional (see **BRA201** versioning)
description: Core engineering standards and reusable technical practices.  # optional
categories:
  - name: Backend                      # REQUIRED
    description: Server-side design, data access and API practices.  # optional
    index: 100                         # optional ordering; defaults to array position
    skills:
      - backend/EP101-database-migrations.md   # paths relative to this manifest
      - backend/EP102-api-versioning.md
  - name: Frontend
    description: Client-side architecture and UI patterns.
    index: 200
    skills:
      - frontend/EP201-component-design.md
```

## 2. Skill file (markdown + frontmatter)

```markdown
---
name: Database Migrations              # REQUIRED (skill title)
code: EP101                            # REQUIRED (unique skill code; conventionally <prefix><n>)
description: How to author safe, reversible database migrations.  # optional
version: 1.0.0                         # optional (see **BRA201** versioning)

# Optional: tool names this skill needs when loaded via get_skill (see below).
# Values are tool `name`s from the Tools table (same identifiers workflows use).
tools:
  - list_schema_files
  - get_schema_file
  - update_schema_file
  - create_skill
  - create_schema_file
---

# Instructions

1. Keep every migration idempotent and forward-only where possible.
2. …
```

Rules:

- The markdown **body must not be empty** — the body is the skill content.
- `name` and `code` are required in frontmatter.
- Skill `code`s are what workflows reference in `injected-skills` /
  `available-skills`.
- `tools` is optional. Omit it entirely when the skill does not require tools.
  When present, list tool **names** (not paths). Deploy stores them as
  `Skills.ToolCodes`; extract writes the list back (or omits the key when empty).

## 3. Skill-declared tools and mid-run promotion

Skills may declare the tools they need. That declaration does **not** grant the
workflow those tools by itself — the workflow still owns the permission
envelope. Promotion only happens when all of the following are true:

1. The skill lists the tool under frontmatter `tools:`.
2. The workflow lists that same tool under `available-tools:` (not only under
   `tools:` — see **BRA217**).
3. During a run, the agent calls `get_skill` for that skill.

Matching names are then **promoted** for the remainder of that run only. They
appear in the model's tool list on subsequent turns of the same run. They are
never written back to the workflow definition, and they do not affect other runs.

| Outcome | Behaviour |
| ------- | --------- |
| Tool in skill `tools:` **and** workflow `available-tools:` | Promoted for this run |
| Tool in skill `tools:` but **not** in the workflow available pool | Silently skipped (workflow curation wins) |
| Tool already in workflow `tools:` (injected) | Already declared; promotion is a no-op / deduped |
| System tools | Always available when declared on the workflow; they are not part of promotion |

Discovering available tools at runtime uses `find_available_tools` (semantic
search over the workflow's `available-tools` pool). See the live example in
`WF-SKILL-UPDATE` + skill `BRA203`.

---
