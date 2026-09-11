---
name: Blueprints
code: BRA216
version: 1
description: How to author blueprint.yml and blueprint entry markdown — scope,
  categories, and per-entry frontmatter. Load this when creating or editing a
  blueprint or one of its entries.
---

# Blueprints

A blueprint is a scoped collection of long-form knowledge. The `blueprint.yml`
manifest declares the **scope** and the **categories**; the blueprint's entries
are the **sibling markdown files** in the same folder, each tagged with a
category in its frontmatter. Overview and versioning: **BRA201**. Entity and
unit-of-work codes used in scope: **BRA213**.

## 1. Blueprint manifest (`blueprints/<name>/blueprint.yml`)

```yaml
name: Product Brain                    # REQUIRED (blueprint title)
version: 1                             # optional (see **BRA201** versioning)
# description: optional

# Scope — either object form or shorthand string form (both accepted):
scope:
  type: brain                          # one of: brain | entity | unitofwork
  # code: application                  # REQUIRED when type is entity/unitofwork
categories:
  - name: Vision                       # REQUIRED
    description: The long-term vision and guiding principles.  # optional
  - name: Architecture
    description: High-level technical architecture and key decisions.
```

Scope shorthand (equivalent to the object form):

```yaml
scope: brain                           # brain-scoped
scope: entity:application              # entity-scoped, code = application
scope: unitofwork:ticket               # unit-of-work-scoped, code = ticket
```

Notes on scope:

- Accepted keywords: `brain`, `entity`, `unitofwork` (case-insensitive).
- `entity` and `unitofwork` **require a code** that matches an entity/unit code
  declared in `brain-compose.yml`.
- Omitting `scope` entirely defaults to `brain`.
- The blueprint's **code is derived from its folder name** (it has no `code`
  field of its own). So `blueprints/product-brain/` → code `product-brain`.

## 2. Blueprint entries (markdown + frontmatter)

**Every `.md` file in the blueprint folder is treated as an entry** (sorted
alphabetically). Each must declare a `category` that exists in the manifest.

```markdown
---
name: Vision Overview                  # REQUIRED (entry title)
category: Vision                       # REQUIRED (must match a manifest category, case-insensitive)
version: 1.0.0                         # optional; not currently uploaded per-entry
---

# Vision Overview

We are building an AI brain that captures and applies an organisation's
knowledge consistently…
```

Rules:

- The markdown **body must not be empty**.
- `category` must match one of the manifest's category names (case-insensitive);
  an unknown category is a hard error.
- Because every `.md` in the folder becomes an entry, don't drop unrelated
  markdown into a blueprint folder.

---
