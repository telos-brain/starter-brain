---
name: Auto-Building a Brain
code: BRA211
version: 1
description: End-to-end process for auto-building a brain schema from an
  existing application — entity and unit of work, skill-book categories,
  workflows, tools, Execute API plumbing, and bidirectional secrets. Use
  instead of the BRA104 interview when the host app already exists. Load
  BRA105 before starting.
---

# Auto-Building a Brain

Build a working brain from the application in the current workspace. When you
finish, the app and the brain talk to each other — once the user has filled in
the missing secrets.

This is a **hands-off agentic build**. Plan the large steps (especially tools),
then execute. Resume from whatever is already on disk; do not start from scratch
if a schema already exists.

**Not this skill:** configuring a greenfield brain by interviewing the user —
that is **BRA104**.

---

## Before you start

1. Load **BRA105** (Brain Principles) and keep it in force for the whole build.
   Do not restate those principles here.
2. Load depth on demand, by code — **BRA101** (building blocks), **BRA201**
   (file formats), **BRA208** (skill-book categories), **BRA202** (secrets),
   **BRA401** / **BRA402** / **BRA403** (Execute API). Do not inline those
   skills.
3. Find any existing brain schema. It may be `./brain`, or another folder that
   contains `brain-compose.yml` (or `brain.yml`). Check what is already done
   and continue from there.
4. If there is no schema yet, run `brain init` (or init into a named folder)
   so you start from the starter brain.

### What to keep from the starter

The starter already contains learning and maintenance machinery, plus the
**Telos Brain** skill book (this book). Keep all of that.

- Keep the Telos Brain skill book. Add **new** skill books beside it.
- Keep the starter workflows (learning, eval, compaction, skill update, and
  other brain-making workflows). Add **new** application workflows beside them.
- Do not strip the starter down to an empty schema.

---

## 1. Entity and unit of work

Research the application: tenancy, the primary record the work hangs off, and
the jobs that start and finish.

| Concept | What you are looking for | Examples |
|---|---|---|
| **Entity** | Where siloed memory should live — a tenant or long-lived subject | Client, department, organisation, account |
| **Unit of work** | A job that starts and finishes, with context that lasts for that job | Proposal, campaign, job, ticket |

Multi-tenancy usually maps to the entity. Scoped memory and tool variables
follow from that (see **BRA102**, **BRA201** §4.1).

Write the entity type and unit-of-work type into `brain-compose.yml`, with
`scope: entity:<entity-code>` on the unit of work. Declare entity / unit-of-work
**variables** for tenant or external IDs the harness will inject — never for
the LLM to type in.

---

## 2. Blueprint and skill-book categories

Define these early. They are the lens for everything the brain later learns
(**BRA105**, **BRA208**).

Do **not** ask the user to invent categories from a blank page. Research the
application and ask: **what skills would an AI need in order to run this
application well?**

That is broader than “how to use the product”:

- If the app runs marketing campaigns, you need a place for *how to do good
  marketing*, not only how to click Campaigns.
- Look for craft that lives **outside** the database — professional practice,
  judgement, writing, process — that the schema of the app still hints at.

Create skill-book **categories** (and blueprint categories) that name that
information. Leave the categories **empty**. Do not try to write all the
skills. The output of this step is the information architecture, not the
content.

Add new skill books for the application domain. Do not pour application craft
into the Telos Brain book.

After deploy, tell the user how to train those empty categories (step 8).

---

## 3. Extract workflows

Search the codebase for clues:

- Existing prompts, system messages, or any place an LLM is already called.
- Triggers: in-app chat, scheduled heartbeats, buttons or actions that should
  run AI.

If the app has no chat, assume it will want one — that is a common reason to
add a brain. Also assume a heartbeat or similar scheduled pass may be useful.
Look for one **asynchronous** action that is contextual and useful (a button,
“job created”, “run heartbeat”).

For each workflow you add:

- Write default instructions that are specific to this application.
- Set LLM boundaries on the workflow (**BRA105**, **BRA201** §8.1): `max-turns`,
  `output-tokens`, thinking budget, `max-runs-per-hour`. Give the model a
  budget; do not leave it on free rein.
- Keep starter workflows. New workflows sit beside them.

You will wire tools onto these workflows in step 5, after tools exist.

---

## 4. Extract tools

Principle: **anything a user can do, the AI can do.** Tools are a thin
interface over existing behaviour — do not duplicate business logic in the
AI API layer.

### If the app already has AI tools

Put an HTTP API in front of them, with an API key. The brain calls that API;
the harness authenticates. Tenant and security IDs are **not** LLM-facing
parameters — they come from entity / unit-of-work variables or secrets
(**BRA201** §5.3, **BRA202**).

### If it does not

Walk the product: what can a user set up, create, update, list, complete?
Those become tools.

### References, never UUIDs

The LLM must not see database UUIDs. Every table the AI will list or act on
needs a **reference**:

- Two-letter prefix + integer, no underscore or dash — e.g. `JO1`, `US12`.
- Prefix from the object type (job → `JO`, user → `US`).
- Token-efficient, unique, and easy to allocate next-in-sequence.
- Store it on the row. If the schema has no suitable reference, add one.

List tools return references, not ids: `US1,Jane Chen,jane@acme.com,Director`.

### Token-efficient responses

These APIs are for an LLM, not a SPA.

- **CSV** for lists.
- **Markdown** for a single detailed object.
- Do not default to JSON payloads “because it is an API”.

Use tool `description`, parameter descriptions, and `response-markdown` /
`error-markdown` as the mini-skill (**BRA105**, **BRA201** §5). Point errors
at skill codes to load, not at a wall of explanation.

### How to take the tool work (plan, then back-fill)

This step is large. Use the schema as the plan:

1. Identify **tool groups** (product areas).
2. List the **tools** in each group.
3. Write the tool YAML (name, description, parameters, response templates).
4. Implement the matching API endpoints **one by one**, as a thin layer over
   existing functionality.
5. Add a unit test per endpoint, hooked into the app’s existing test
   framework.

Register groups in `brain-compose.yml`. Inject API keys with `secret:` and
tenant context with `entity:` / `unitofwork:` — the model never passes those.

---

## 5. Wire workflows to tools

Go back through each application workflow and attach the tools it actually
needs (`tools:` / `available-tools:`). Discovery tools (`find_available_skills`,
`get_skill`) belong on conversational workflows. Do not give every workflow
every tool.

---

## 6. Execute API plumbing in the application

The brain is not finished until the app can start runs. Load **BRA401**,
**BRA402**, **BRA403** for the HTTP surface.

### Chat (synchronous)

- Call `POST /workflows/{code}/run/sync`.
- Treat the returned run as a **session**. Reuse that session id on later
  turns.
- First version: keep session state in the client. Do not add a resumable
  server-side chat store unless the app already has one.
- UI: reuse an existing chat if there is one. Otherwise add either a main-menu
  item or a bottom-right chat icon that opens a small modal — then close it
  when done.

If a chat UI already exists, keep it. Route each AI turn through the Execute
API and store the run id where that UI already stores conversation state.

### One asynchronous action

Pick a useful in-app action (button, job created, heartbeat). That action
starts `POST /workflows/{code}/run/async`, scoped to an entity and/or unit of
work.

Before firing the run:

1. Ensure the entity exists in the brain (**BRA402**). Persist the brain
   entity id on the application row (e.g. `BrainEntityId` on Client).
2. Ensure the unit of work exists if the run is for a job. Persist
   `BrainUnitOfWorkId` the same way.
3. Create those records if they are missing, then pass `entityId` and/or
   `unitOfWorkId` on the run request.

Set `allowed-callback-domains` in compose if you use async `callbackUrl`
(**BRA201** §4.3).

---

## 7. Environment variables (both directions)

Traffic goes both ways, so keys must exist on both sides. Load **BRA202**.

| Direction | Secret | Where it lives |
|---|---|---|
| Brain → app (tools calling the app) | App API key the user mints | App `.env` **and** brain `.env` (same value). Tool parameters use `secret:`. |
| App → brain (Execute API) | Brain API key (issued on first `brain deploy`) | App `.env` after deploy. |

- Put **names** in both `.env.example` files (app and brain). Do not commit
  real values.
- You may write placeholders into local `.env` files; the user still has to
  mint matching keys.
- After first deploy, the CLI prints the brain API key **once**. The user
  (or you, if they paste it) puts that value in the app env.

Finish this step with a short **handoff list**: which keys to create, where
each one goes, and that they must match.

---

## 8. Deploy, train, snapshot

1. `brain deploy --dry-run`, then deploy when the schema is valid.
2. Do **not** fill the empty skill categories yourself. Tell the user:
   log into the Telos Brain portal, open the **inbox**, and upload documents,
   transcripts, or anything they want the brain trained on.
3. Training after deploy makes the live brain diverge from source. After a
   stretch of training, run `brain snapshot` so the schema on disk catches up.

---

## Progress checklist

Skip any step that is already true of the repo:

- [ ] Starter schema present; Telos Brain book and starter workflows kept
- [ ] Entity and unit of work in `brain-compose.yml`, with variables for tenancy
- [ ] Blueprint categories and application skill-book categories defined (empty)
- [ ] Application workflows added, with LLM budgets set
- [ ] Tool groups and tool YAML written
- [ ] Thin AI API layer + tests; references (not UUIDs) on listed records
- [ ] Workflows linked to the tools they need
- [ ] Chat (sync session) and one async action wired, with brain ids stored
- [ ] `.env.example` names for both directions; handoff list for the user
- [ ] Dry-run / deploy; user told how to train; snapshot noted for later

When the checklist is done, the build is complete except for the secrets the
user must paste and the knowledge they must train.
