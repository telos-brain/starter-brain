---
name: Designing Skill Books
code: BRA208
version: 3
description: How to design a Skill Book's category structure and skill number
  ranges so the AI can place knowledge correctly. Use when creating a new
  skillbook, proposing or reviewing categories, writing category descriptions,
  choosing category indexes (100, 200, 300…), or assigning skill codes within a
  range.
---

# Designing Skill Books

Categories are the information architecture of a Skill Book. They define what
kinds of skills belong and how they are grouped. Each category covers a numeric
range (100, 200, 300, up to 900) and holds up to 99 skills within that range.
Categories are the lens through which the AI analyses all future information —
every document, lesson, or brain dump will be read through these categories to
decide what knowledge to extract and where to place it. Getting categories right
is one of the most important steps in setting up a Skill Book.

This skill covers **how to design** that structure. For what a Skill Book is and
how agents work with skills, see **BRA103**. For the YAML file format and deploy
rules, see **BRA201** §6.

---

## 1. What you are designing

A Skill Book needs:

| Piece | Role |
|---|---|
| **Name + description** | Scope of the book — what subject it covers and what stays out |
| **Categories** | Distinct dimensions of the subject, each with a range index and a rich description |
| **Skill codes** | `<prefix><category><nn>` (e.g. `BRA201`) numbered inside each category's range |

Aim for around **9 categories** as a strong starting point — enough breadth
without fragmentation. There is no hard limit; add more only when the domain
genuinely demands it. Every category must earn its place as a distinct,
meaningful dimension. If two categories would frequently overlap or confuse the
AI about where to place a skill, merge them.

---

## 2. Classify the subject

Choose a decomposition lens that matches the domain. Different domains break
down differently:

### Professional discipline
(architecture, engineering, law, medicine, accounting)

Break down by facets of practice: core body of knowledge, tools and technology,
professional standards and regulation, design/methodology, project delivery,
client and stakeholder management, business of practice, and inspiration or
reference material.

*Example — Architecture:*
- 100: Design Principles and Theory
- 200: Building Technology and Materials
- 300: Tools and Software
- 400: Codes, Standards, and Regulations
- 500: Project Delivery and Documentation
- 600: Professional Practice and Business
- 700: Styles, Movements, and Precedents
- 800: Sustainability and Performance
- 900: Specialist and Emerging Topics

### Sport or officiation
(referee, coaching, athletics)

Break down by the rule framework, match-day procedures, fitness and preparation,
local regulations, personal development, and experience/case studies.

*Example — Football Referee:*
- 100: Laws of the Game
- 200: Match-Day Procedures
- 300: Fitness and Physical Preparation
- 400: Local Regulations and Competition Rules
- 500: Communication and People Management
- 600: Positioning and Movement
- 700: Video and Technology (VAR)
- 800: Career Development and Pathway
- 900: Match Experience and Case Studies

### Company or organisation

Think about the company as a value chain or operational stack. What are its
departments or functional areas? Consider the end-to-end flow: sales, onboarding,
planning, delivery, quality, support, business operations, finance, and
people/culture.

*Example — Software Consultancy:*
- 100: Sales and Business Development
- 200: Client Onboarding
- 300: Planning and Estimation
- 400: Delivery and Engineering
- 500: Quality Assurance and Standards
- 600: Operations and Tooling
- 700: People, Culture, and HR
- 800: Finance and Commercial
- 900: Strategy and Leadership

### Technical domain
(programming language, framework, platform)

Break down by fundamentals, core patterns, ecosystem and tooling, testing,
deployment, performance, security, and advanced or niche topics.

*Example — React Development:*
- 100: Language Fundamentals (JS/TS)
- 200: Component Patterns
- 300: State Management
- 400: Routing and Navigation
- 500: Data Fetching and APIs
- 600: Testing and Quality
- 700: Build, Deploy, and DevOps
- 800: Performance and Optimisation
- 900: Advanced Patterns and Architecture

### Personal knowledge or hobby
(cooking, photography, gardening)

Break down by foundational theory, core techniques, tools and equipment, styles
or genres, planning and workflow, and advanced or specialist areas.

### Regulatory or compliance domain
(tax, health and safety, data protection)

Break down by legislation and source rules, interpretation and guidance,
processes and procedures, reporting, audit, edge cases, and jurisdiction-specific
variations.

If the subject is ambiguous, clarify one thing before designing (e.g. "Is this a
product company or a consultancy?"). Do not invent categories outside the book's
stated scope.

---

## 3. Propose the category set

For each category, define:

1. **Name** — concise, 2–4 words
2. **Range** — a positive multiple of 100 (`100`, `200`, `300`…) stored as the
   category `index` in `skillbook.yml`
3. **Description** — rich enough for an AI to decide membership confidently
   (see §4)

Together the set should give comprehensive breadth: the owner should think "yes,
that covers everything I'd want to capture." Propose a full structure
proactively — do not leave category invention entirely to the user.

Record briefly which decomposition lens you applied and why it suits the
subject.

### Common refinements

| Signal | Action |
|---|---|
| Two categories feel too similar | Merge them |
| One category covers too much ground | Split it |
| A domain-specific facet is missing | Add a category |
| Names don't match the user's terminology | Rename |

---

## 4. Writing category descriptions

Category descriptions are the primary signal the AI uses to decide where a skill
belongs. A weak description means skills get misplaced or knowledge gets missed
entirely.

**A good description:**
- Defines boundaries clearly enough that an AI can decide "this belongs here" or
  "this doesn't"
- Captures the full breadth of topics — not just the obvious ones
- Distinguishes this category from adjacent ones where topics might overlap

**How to write them:**
- Lead with the core purpose, then list the key topic areas
- Be specific about what falls inside the boundary. "Skills related to project
  delivery" is too vague. "Planning, scheduling, milestones, progress tracking,
  stakeholder reporting, risk management, and delivery process improvements"
  tells the AI exactly what to look for
- If two categories could plausibly cover the same topic, draw the line
  explicitly in both descriptions

**Examples:**

Weak: "Skills about tools and software."

Strong: "Selection, configuration, and effective use of tools and software for
the discipline — including design tools, analysis software, documentation
platforms, collaboration tools, and emerging technology. Covers workflows,
integrations, tips, and tool-specific best practices."

Weak: "Career and professional development."

Strong: "Career progression, professional development pathways, mentoring,
certification and qualification requirements, industry networking, personal
effectiveness, and strategies for advancing within the profession."

Do not settle for vague one-line descriptions.

---

## 5. Range guidance

Category `index` values are multiples of 100 from 100 up to 900. Skills inside a
category use the remaining two digits (e.g. prefix `TEL` + category `100` →
`TEL101`). The category range also signals skill depth across the book:

| Range | Depth |
|---|---|
| 100s | Foundational |
| 200–400 | Intermediate |
| 500–700 | Advanced |
| 800–900 | Expert / specialist |

Number skills within a category sequentially as they are added. Leave gaps only
when you deliberately reserve space for a planned sequence; do not scatter
numbers randomly.

---

## 6. Encode in the schema

Once the structure is agreed, encode it in `skills/<book>/skillbook.yml`
(BRA201 §6.1):

```yaml
name: Engineering Practices
code: ENG
prefix: EP
version: 1
description: Core engineering standards and reusable technical practices.
categories:
  - name: Backend
    description: >-
      Server-side design, data access, API contracts, persistence, and
      operational concerns for backend services. Distinct from Frontend (UI and
      client state) and from Delivery (cross-cutting process and release).
    index: 100
    skills:
      - backend/EP101-database-migrations.md
  - name: Frontend
    description: >-
      Client-side architecture, component patterns, UI state, routing, and
      browser concerns. Distinct from Backend APIs and from Testing methodology.
    index: 200
    skills: []
```

Checklist when applying:

- [ ] Book `name`, `code`, `prefix`, and `description` match the intended scope
- [ ] Every category has a rich description (not a vague one-liner)
- [ ] Category `index` values are unique multiples of 100
- [ ] Adjacent categories have explicit boundary language where overlap is likely
- [ ] Skill codes use the book prefix and sit in the correct category range
- [ ] Category ranges match intended depth (100s foundational … 800–900 expert)
- [ ] Skillbook `version` is bumped when the manifest changes

---

## 7. Design rules

- Always propose categories from the subject — do not start from an empty list
- Aim for full breadth within the book's stated scope; nothing outside that scope
- Keep category names concise (2–4 words)
- Treat description quality as the most important property of a category
- Prefer merging overlapping categories over proliferating near-duplicates
- Place categories so range depth matches the material (100s foundational …
  800–900 expert)
