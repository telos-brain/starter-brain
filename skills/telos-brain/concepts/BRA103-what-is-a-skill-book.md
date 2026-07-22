---
name: What Is a Skill Book
code: BRA103
version: 4
description: What a Skill Book is — a self-contained folder of categorised
  skills managed by skillbook.yml, portable between brains, with codes that let
  skills reference each other for progressive disclosure. Use when explaining
  skill books, placing a skill in a book, or deciding what belongs in skill
  content.
---

# What Is a Skill Book

A Skill Book is a named collection of skills organised into categories.
Categories are the information architecture of a Skill Book — they define what
kinds of skills belong in the book and how they are grouped. Each category
covers a numeric range (100, 200, 300, up to 900), which also signals skill
depth: 100s are foundational, 200–400 intermediate, 500–700 advanced, and
800–900 expert or specialist. Well-defined categories help the AI agent
understand the scope and intent of the Skill Book, so it can place skills
correctly and avoid overlap.

Each skill book has a prefix (e.g. TEL) and each skill has a unique code that
uses the prefix and the category range (e.g. TEL101). Skills also have a name, a
short description, and full markdown content.

---

## Codes and progressive disclosure

Skill codes are stable identifiers. Skills use them to refer to other skills
(e.g. "see **BRA201** §6") without embedding that material. That is progressive
disclosure: an agent discovers a skill, loads only what it needs, then follows
codes to related skills when deeper detail is required. Codes make the book
traversable — searchable, loadable by id, and linkable — rather than a flat dump
of everything at once.

---

## Packaging and portability

A Skill Book is a **self-contained folder**. The folder holds the skill markdown
files and a `skillbook.yml` manifest that declares the book's name, code,
prefix, categories, and which skills belong in each category. Because everything
the book needs lives in that folder, a Skill Book can be copied between brains
(or referenced from `brain-compose.yml`) without rewriting its contents — only
the compose path that points at `skillbook.yml` needs to change.

---

## What a skill is

Skills are reusable instruction patterns that encode best practices, standards,
processes and domain knowledge. Think of them as the agentic equivalent of NPM
packages — composable, versioned, and purpose-built.

---

## Skill content rules

- Skills are transferable across multiple AI platforms, so they must never
  include personal details or specifics. Always extract the best practices and
  learnings and if necessary, describe a scenario without any personally
  identifiable information.
- Keep skill descriptions concise — one sentence that says what the skill does
  and when to use it.
- Write skill content in clear, actionable markdown.

---

## Related skills

- **BRA208** — designing a book's category set and ranges
- **BRA201** §6 — encoding skillbooks in a brain schema (`skillbook.yml` and
  skill markdown)
