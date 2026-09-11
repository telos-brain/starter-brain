---
name: Brain Compose Manifest
code: BRA213
version: 1
description: How to author brain-compose.yml — name, models, entities, units of
  work, variables, checkpoint strategy, and the outbound callback-domain
  allowlist. Load this when creating or editing the compose file.
---

# Brain Compose Manifest

The compose file is the deploy entry point. Anything not listed under
`connectors`, `tools`, `skills`, `blueprints`, or `workflows` is not deployed,
even if the file exists on disk. Overview, directory layout, deploy, and
versioning: **BRA201**. Connectors: **BRA209**. Environment variables: **BRA202**.
Tool parameter bindings that read entity / unit-of-work variables: **BRA214**.

```yaml
name: kappa                      # REQUIRED: the brain's name
# description: optional          # optional; used as the brain description on first deploy

# Optional brain-level settings (persisted on every `brain deploy`):
# embedding-model: voyage-3-lite # optional; defaults to voyage-3-lite when omitted
# learning-mode: off             # optional; off | low | medium | high (omit = off)
# llm-model: local_1/qwen3:8b    # optional; default LLM (omit = workflow model)
# transcription-model: anthropic/claude-sonnet-4-6  # optional; vision model for image transcription (BRA410)
# checkpoint-strategy: Daily     # optional; see checkpoint strategy below (omit = Daily)
# allowed-callback-domains:      # optional; shared outbound host allowlist (see below)
#   - harness.example.com

# Entities: top-level things the brain reasons about.
entities:
  - name: Application            # REQUIRED
    code: application            # REQUIRED (referenced by scopes elsewhere)
    # variables: optional per-entity variables (see below)
    variables:
      - key: organisationId      # REQUIRED (the variable key)
        description: External CRM organisation ID.   # optional

# Units of work: scoped pieces of work. `scope` lists the entity/unit codes it
# operates across, in shorthand form.
unitsofwork:                     # also accepted as `unitsOfWork`
  - name: Ticket                 # REQUIRED
    code: ticket                 # REQUIRED
    scope: entity:application    # optional
    # variables: optional per-unit-of-work variables (see below)
    variables:
      - key: jobId               # REQUIRED (the variable key)
        description: External job ID for this ticket.   # optional

# Each of the following is a LIST OF PATHS to self-contained definitions.
connectors:
  - connectors/example-oauth2.yml

tools:
  - tools/tickets/tools.yml

skills:
  - skills/eng/skillbook.yml
  - skills/ops/skillbook.yml

blueprints:
  - blueprints/product-brain/blueprint.yml
  - blueprints/application/blueprint.yml

workflows:
  - workflows/review-blueprint.md
```

Rules:

- `name` is the only required top-level field.
- `embedding-model` is optional (defaults to `voyage-3-lite` when omitted).
- `learning-mode` is optional (`off` | `low` | `medium` | `high`; omit or null is
  treated as `off`). Persisted onto the Brain on every deploy.
- `llm-model` is optional (a `provider/model` string such as
  `local_1/qwen3:8b` or `anthropic/claude-sonnet-4-6`). When set and the matching
  credential exists, every live run uses this model instead of the workflow
  frontmatter. Omit or blank → each workflow uses its own `model:`. If that is
  also omitted, the run fails (no silent Anthropic/OpenAI default). A missing
  credential for this value falls back to the workflow model. The same default
  can be set with `DEFAULT_LLM_MODEL` in `.env` (compose `llm-model` wins when
  both are present). Simulation `settingsOverride.model` still wins per run.
  Deploy warns (does not fail) when executable workflows have no `model:` and
  no default is set. See **BRA210**.
- `transcription-model` is optional (a vision `provider/model` used for image
  transcription — **BRA410** / **BRA412**). Omit → auto-detect from
  `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`.
- `checkpoint-strategy` is optional (see below).
- `allowed-callback-domains` is optional (see below) — shared host allowlist for
  async run callbacks and declared-tool webhook URLs.
- `entities` / `unitsofwork` are optional lists; each item needs `name` + `code`.
- `connectors`, `tools`, `skills`, `blueprints`, `workflows` are optional lists of
  relative paths. Anything **not listed here is not deployed**, even if the file
  exists on disk.

## 1. Entity and unit-of-work variables (per-instance key/value pairs)

An entity type can declare **variables** — named slots that each *instance* of
that type can fill with a scalar value (e.g. an external `organisationId`, an
account code, a region). The **keys** are schema (declared here, in
`brain-compose.yml`); the **values** are runtime data set per entity instance
via the Execution API (see BRA402).

```yaml
entities:
  - name: Customer
    code: customer
    variables:
      - key: organisationId                       # REQUIRED (the variable key)
        description: The external CRM organisation ID for this customer.  # optional
      - key: accountCode
        description: The billing account code.
```

Rules:

- `variables` is an optional list under an entity; each item needs a `key`
  (`description` is optional). Keys are unique per entity type.
- Deploying is **upsert-always** (like the entity type itself): new keys are
  added, existing ones refresh their description, and keys removed from the list
  are retired on the next deploy.
- The point of declaring a variable is so a **tool parameter can bind to it** and
  have the current entity's value injected automatically at dispatch — see the
  `entity:` parameter field in **BRA214**.

A **unit-of-work type** declares variables in exactly the same way, for data that
belongs to a single piece of work rather than to the entity behind it (e.g. the
external `jobId` the harness created for this ticket):

```yaml
unitsofwork:
  - name: Ticket
    code: ticket
    scope: entity:customer
    variables:
      - key: jobId                                # REQUIRED (the variable key)
        description: The external job ID for this ticket.   # optional
      - key: batchCode
        description: The processing batch this ticket belongs to.
```

The same rules apply — keys are unique per unit-of-work type, deploying is
upsert-always, and values are set per unit-of-work instance via the Execution API
(BRA402). A tool parameter binds to one with the `unitofwork:` field (**BRA214**).

Choose by lifetime: put a value on the **entity** when it is stable across every
piece of work for that record; put it on the **unit of work** when it is specific
to this job. A tool can bind to both at once.

## 2. Checkpoint strategy

Checkpoints are lightweight point-in-time markers for a Brain's schema (skills,
workflows, tools, blueprint entries). Creation is cheap; schema components are
copied on write when they change after a checkpoint exists — each snapshot stores
the full serialised `.md`/`.yml` file content so diffs and reverts round-trip
tools, parameters, and all frontmatter. Runtime data (workflow runs, entities,
units of work, inbox) is out of scope.

Configure the schedule in `brain-compose.yml` (persisted on every `brain deploy`).
Strategy is **not** editable via the Management API or Settings UI. Retention
(`MaxCheckpoints`) is a system setting — not writable from the brain schema.

```yaml
# Scalar (single strategy):
checkpoint-strategy: Daily

# Comma-separated (multiple strategies):
checkpoint-strategy: Weekly,BeforeDeploy

# YAML list (equivalent):
checkpoint-strategy:
  - Weekly
  - BeforeDeploy
```

**`checkpoint-strategy`** — one or more of:

| Value | Kind | When a checkpoint is created |
| ----- | ---- | ---------------------------- |
| `BeforeDeploy` | Event | Immediately **before** schema resource phases on `brain deploy` (server-side). Copy-on-write then captures the pre-deploy schema as resources mutate. |
| `AfterDeploy` | Event | Immediately **after** all schema resource phases on `brain deploy` (server-side). Marks the post-deploy schema as the restore baseline. |
| `Daily` | Schedule | Once per day. Equivalent aliases: `@daily`. |
| `Weekly` | Schedule | Once per week. Equivalent aliases: `@weekly`, `0 0 * * 0`. |
| *(cron)* | Schedule | Any other free-form cron expression (e.g. `0 30 9 * * 1-5`). |

Defaults: omit or blank → `Daily`. Multiple values are normalised to PascalCase
and stored comma-separated. **Every listed strategy is active** — they form a
union (a checkpoint fires when any strategy triggers). Event and schedule
strategies may be combined (e.g. `Weekly,BeforeDeploy` or
`Daily,Weekly,BeforeDeploy,AfterDeploy`).

Rules:

- Event strategies (`BeforeDeploy` / `AfterDeploy`) are handled **server-side**
  during deploy — the CLI does not create checkpoints. Both may be set together;
  each fires at its phase.
- Schedule strategies (`Daily` / `Weekly` / cron) are armed on deploy. The
  soonest next fire across **all** schedule entries is used. Multiple
  schedules are a union, not first-wins.
- Event-only strategies do not create a recurring schedule.
- Do not set `max-checkpoints` in `brain-compose.yml` — it is ignored on deploy.

## 3. Allowed callback / webhook domains (SSRF allowlist)

The Brain makes outbound HTTP calls to harness-owned URLs in two places:

1. **Async run callbacks** — optional `callbackUrl` on
   `POST /workflows/{code}/run/async` (see BRA403).
2. **Declared API tools** — `api.path` webhook URLs dispatched by the Tool Router
   during a run.

To prevent server-side request forgery (SSRF), both surfaces share one per-brain
host allowlist configured in `brain-compose.yml` and persisted on every
`brain deploy`. The field is **not** editable via Management API PATCH or the
Settings UI.

```yaml
# Scalar (single host):
allowed-callback-domains: harness.example.com

# Comma-separated:
allowed-callback-domains: harness.example.com, hooks.example.org

# YAML list (equivalent):
allowed-callback-domains:
  - harness.example.com
  - hooks.example.org
```

Rules:

- Hostnames are matched **exactly** (case-insensitive). No wildcards, no
  subdomain matching (`evil.harness.example.com` does not match
  `harness.example.com`).
- Ports are not part of the allowlist entry — match is on the DNS host only.
- Omit or blank → no host allowlist. Private / loopback / link-local / cloud
  metadata IP ranges are still blocked after DNS resolution in all environments.
- Schemes: `https` only in non-Development. In Development,
  `http://localhost` and `http://127.0.0.1` (any port) are also permitted for
  local harnesses.
- When the list is populated, a `callbackUrl` or tool `api.path` whose host is
  not listed is rejected (async run → `Failed`; tool call → error result, no
  outbound request). A `SYSTEM_CHANGE` inbox entry (status `PROCESSED`) is also
  created with the denied URL, reason, and the workflow/tool/run involved.
- Redirect following is disabled on outbound webhook HTTP clients so a redirect
  chain cannot reach an internal target after the initial URL passed validation.

Configure every harness hostname that will receive async completion callbacks or
declared-tool webhooks before deploying tools / using async runs against those
hosts.

