---
name: Create Inbox Cluster System Tool
code: BRA413
version: 1
description: The create_inbox_cluster system tool — consolidate related inbox
  entries into a single PROCESSED cluster entry. Source entries become
  COMPLETED and their open tasks are cancelled. Complements the inbox tools
  in BRA405.
tools:
  - create_inbox_cluster
---

# Create Inbox Cluster

BRA405 covers the inbox system tools for creating, listing, and updating
individual entries. This skill covers **`create_inbox_cluster`**: collapsing
related or near-duplicate signals into one stronger entry before human review.

This is an ordinary `system` tool (BRA201 §5.2). Declaration lives under
`tools/inbox/create-inbox-cluster.yml`. This skill lists it in frontmatter
`tools:` so a workflow that keeps it under `available-tools` can promote it via
`get_skill`.

**Scope:** always the current Brain (`BrainId` harness-injected). Never pass a
brain id. Identity is by 8-character lowercase alphanumeric **reference** — never
UUID.

---

## Purpose

Clustering reduces noise: related or near-duplicate signals become one
inbox entry. Weight is the sum of the source entries' current weights
(manual overrides included) and is not recalculated afterwards. The
cluster is created as `PROCESSED` with no trigger tasks — triage adds
tasks afterwards.

---

## `create_inbox_cluster`

| | |
| --- | --- |
| **Purpose** | Consolidate related inbox entries into one PROCESSED cluster entry |
| **When** | Related signals from the same recurring failure, near-duplicates found during triage, or groups of weight-1 entries around one pattern |
| **YAML** | `tools/inbox/create-inbox-cluster.yml` |

### Parameters

All parameters are strings. `BrainId` is harness-injected — never a parameter.

| Parameter | Required | Notes |
| --- | --- | --- |
| `inbox_entry_references` | Yes | Comma-separated 8-character references (at least two). Whitespace is trimmed. |
| `cluster_title` | Yes | Title for the new cluster entry (max 500 characters). |
| `cluster_description` | Yes | Body/description for the new cluster entry (markdown). |

### Return value

Plain markdown confirmation including the new cluster entry's **reference** and
how many sources were consolidated, for example:

`Cluster entry created with reference ab12cd34. 3 source entries consolidated.`

### What the tool does (atomically)

1. Loads every source entry in this brain and rejects the call if any reference
   is missing or malformed. Already `COMPLETED` or already-clustered
   (`ClusterId` set) references are **ignored** — they do not fail the call
   and they do not contribute to the new cluster's weight.
2. Creates a new `PROCESSED` inbox entry (`cluster_title` /
   `cluster_description`). Trigger matching is skipped — triage adds tasks
   after create. Fails only when fewer than two clusterable sources remain.
3. Stamps each source's `ClusterId` to the new entry (internal self-FK — not
   shown to the AI).
4. Cancels open source tasks (`PENDING`, `AWAITING_APPROVAL` → `CANCELLED`).
   Tasks already `RUNNING`, `COMPLETED`, `FAILED`, or `CANCELLED` are left
   unchanged.
5. Sets each source entry to `COMPLETED`.

All of those writes commit or roll back together. A failed call leaves no
side effects.

### When to use

- Several granular weight-1 entries about the same recurring failure or pattern
- Near-duplicate entries identified during triage
- Groups of signals that together build toward the ~5+ weight threshold

### When not to use

- A single clear, self-contained learning that should stand alone
- A list that, after ignoring completed / already-clustered refs, has fewer
  than two remaining sources
- Unrelated signals that happen to arrive at the same time
- Entries that belong to a different brain (they read as not found)

### Errors

Failures return a plain-English error (not a success string):

- Missing required parameters
- Fewer than two references
- Invalid reference format (`[a-z0-9]{8}`)
- One or more references not found in this brain
- Fewer than two remaining sources after ignoring completed / already-clustered
  references

---

## See also

- **BRA405** — inbox system tools (`create_inbox_entry`, `list_inbox_entries`, …)
- **BRA404** — Execution API inbox and inbox trigger stages
- **BRA322** — `Weight` and `ClusterId` on `InboxEntries`
