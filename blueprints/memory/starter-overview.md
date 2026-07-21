---
name: Starter Brain Overview
category: Overview
version: 1
---

# Starter Brain Overview

This is a clean starter Telos Brain. It ships with:

- The **Telos Brain** skillbook (`BRA`) covering core concepts, schema authoring
  and the Execution API.
- **Execution tools** for skills, memory (blueprint) access and focused Q&A via
  `ask_question`.
- **Management tools** for inspecting and editing the brain's own
  configuration-as-code schema.

Knowledge that should be retrieved at answer time belongs in this memory
blueprint under **Insights** (or another category you add). Prefer short,
self-contained entries with clear titles so `search_blueprint_entries` can find
them reliably.
