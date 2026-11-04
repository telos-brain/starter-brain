---
name: "Management API: Cost series"
code: BRA302
version: 1
description: How to read daily cost rows for one Brain over a required date
  range — GET /brains/{instance}/cost-series — so an organisation API key
  caller can graph LLM cost, derived brain-minute cost, tokens, minutes, and
  embeddings. Not the admin costs page.
---

# Management API: Cost series

Organisation callers graph one Brain's usage with a single read. The response
is every daily bucket in an inclusive calendar range: LLM cost, derived brain
cost, tokens, minutes, and embeddings. Optional `group_by` splits each day.

This is a Management API read (deploy plane), not an Execution API call.
Authenticate with **BRA301**. Do not use the per-brain Execution API key.

`GET /brains/{instance}/costs` is a different contract. That route backs the
admin costs page (optional dates, status, paging, and `group_by=none` meaning
one row per run). This skill does not change it. Here `group_by=none` means
**one row per calendar day**.

---

## 1. Endpoint

```http
GET /brains/{instance}/cost-series?from=2026-11-01&to=2026-11-30
X-Telos-Api-Key: tbk_...
```

| Part | Meaning |
|---|---|
| `{instance}` | Brain **instance name** (organisation-scoped slug), not a GUID. Never pass `BrainId`. |
| Auth | Clerk organisation JWT or organisation API key (`X-Telos-Api-Key` or `Authorization: Bearer tbk_…`). No Admin role is required on this read. A Member still needs Full access to the brain; otherwise `403` with an empty body. |
| Tenant scope | The brain must belong to the authenticated organisation. A missing or cross-organisation instance is `404`. |

### Query parameters

Snake_case. There is no `page`, `page_size`, `status`, or `group_key`.

| Parameter | Required | Notes |
|---|---|---|
| `from` | Yes | Inclusive calendar date, `yyyy-MM-dd`, in the Brain timezone. |
| `to` | Yes | Inclusive calendar date, `yyyy-MM-dd`. Must be on or after `from`. |
| `group_by` | No | `none` (default, including omitted or blank; case-insensitive), or exactly `model`, `workflow`, `entity`, or `unitOfWork`. |

`from` and `to` are dates, not timestamps. `2026-11-04T00:00:00` is rejected.

### Status codes

| Status | Meaning |
|---|---|
| `200 OK` | JSON array of rows. Empty array when the range has no runs. |
| `400 Bad Request` | Missing, unparseable, or inverted dates, or an unknown `group_by` |
| `401 Unauthorized` | No authenticated organisation |
| `403 Forbidden` | Member without Full access. Body is empty. |
| `404 Not Found` | No brain named `{instance}` in the caller's organisation |

Errors other than the empty `403` use `{ "error": "message" }`.

Typical `400` messages:

- `from and to are required`
- `from and to must be calendar dates (yyyy-MM-dd)`
- `from must be on or before to`
- `group_by must be one of: none, model, workflow, entity, unitOfWork`

---

## 2. Response

A flat JSON array, camelCase, ordered by `date` then the group value. Days
with no runs are omitted. Zero-fill those days on the client if the chart
needs a continuous axis.

```json
[
  {
    "date": "2026-11-04",
    "llmCostCents": 12.5,
    "brainCostCents": 1.5,
    "totalTokens": 8400,
    "totalMinutes": 1.5,
    "embeddingCount": 3
  }
]
```

| Field | Type | Meaning |
|---|---|---|
| `date` | string | Brain-local calendar day, `yyyy-MM-dd`. |
| `llmCostCents` | number or **null** | Sum of stored `CostCents` (US cents). **Null means no pricing data**, not free. A real zero-cost bucket is `0`. Null is present in the JSON; do not treat a missing field as null. |
| `brainCostCents` | number | `(sum of RunSeconds / 60) ×` the brain's **current** `CentsPerMinute`. Derived at read time. Not a credit-ledger debit. |
| `totalTokens` | number | Sum of `TotalInputTokens + TotalOutputTokens` on the run. Not re-summed from messages. |
| `totalMinutes` | number | `sum of RunSeconds / 60`. Fractional. `RunSeconds` is active Hangfire thread time, not wall-clock, so a sync run can have LLM cost and `0` minutes. |
| `embeddingCount` | number | Sum of run-level `EmbeddingCount`. Do not re-sum messages. |

`llmCostCents` stays null only when **every** run in the bucket has null
`CostCents`. Mixed null and priced runs sum the priced rows.

### Grouped rows

`group_by=none` returns the object above. Any other value adds **one** extra
field and omits the rest:

| `group_by` | Extra field | Value |
|---|---|---|
| `model` | `model` | `WorkflowRun.Model`, or `—` when null |
| `workflow` | `workflow` | Workflow title. Runs whose workflow is missing are left out of this grouping only. |
| `entity` | `entity` | Entity name, or `—` when the run has none |
| `unitOfWork` | `unitOfWork` | Unit-of-work title, or `—` when the run has none |

`—` is an em dash, not a hyphen. One row per day per group value.

```http
GET /brains/acme/cost-series?from=2026-11-04&to=2026-11-04&group_by=model
```

```json
[
  {
    "date": "2026-11-04",
    "llmCostCents": null,
    "brainCostCents": 0,
    "totalTokens": 120,
    "totalMinutes": 0,
    "embeddingCount": 0,
    "model": "—"
  },
  {
    "date": "2026-11-04",
    "llmCostCents": 4.2,
    "brainCostCents": 0.5,
    "totalTokens": 3000,
    "totalMinutes": 0.5,
    "embeddingCount": 1,
    "model": "anthropic/claude-sonnet-4-6"
  }
]
```

---

## 3. How to implement a chart

1. Mint or reuse an organisation API key (**BRA301**). Do not send the brain Execution API key.
2. Address the brain by instance name: `/brains/acme/cost-series`.
3. Pass a bounded `from` and `to`. The server returns every bucket in that range; there is no page size.
4. Plot `date` on the x-axis. Plot `llmCostCents`, `brainCostCents`, `totalTokens`, `totalMinutes`, or `embeddingCount` on the y-axis.
5. Keep null `llmCostCents` as a gap or an "unpriced" series. Do not coerce it to `0` before summing, or unpriced usage looks free.
6. For a stacked chart, set `group_by` and use the extra field as the series name. `—` is the series for runs with no model, entity, or unit of work.
7. Insert a zero (or a gap) for calendar days the array skips. The API does not do that.

The date is the Brain's `TIMEZONE` environment variable (IANA, for example
`Pacific/Auckland`). Unset or unrecognised falls back to UTC. Pass `from` and
`to` as calendar dates in that zone, not as UTC instants. A run just after
local midnight stays on that local day even when UTC is still the previous day.

Every status is included. Simulations and nested child runs are included.
Soft-deleted runs are not. There is no credit-balance or statement data on
this route; brain cost is the minute formula above, using the rate configured
on the brain now.
