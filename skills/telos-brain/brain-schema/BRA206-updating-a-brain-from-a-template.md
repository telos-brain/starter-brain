---
name: Updating a Brain from a Template
code: BRA206
version: 1
description: How to update an existing brain's configuration from a source
  (template) brain via the Management API — POST
  /brains/{targetInstance}/update-from/{sourceInstance} — including version
  precedence, what is synced, embeddings, and the DeploySummary response.
---

# Updating a Brain from a Template

After a brain has been cloned from a template (see **BRA205**), the destination
may diverge — it can learn, bump versions, or receive ad-hoc deploys. When the
template itself gains new or newer configuration, use **update-from** to pull
those improvements into the destination **without overwriting** resources the
destination already holds at a higher version.

Update-from is a version-gated configuration sync. It mirrors CLI redeploy
precedence (BRA201 §9) over an HTTP call between two brains in the same
organisation. Runtime data and destination environment variables are never
touched.

This is a Management API operation (deploy plane), not an Execution API call.
See BRA401 for the Management vs Execution distinction.

---

## 1. Endpoint

```http
POST /brains/{targetInstance}/update-from/{sourceInstance}
Authorization: Bearer <clerk-org-jwt>
# or
X-Telos-Api-Key: <organisation-api-key>
```

| Part | Meaning |
|---|---|
| `{targetInstance}` | Destination brain's **instance name** (the brain being updated) |
| `{sourceInstance}` | Source / template brain's **instance name** |
| Auth | Clerk org session or organisation API key; **Admin** role required |
| Tenant scope | Both brains must belong to the caller's organisation |
| Body | None — path parameters alone identify source and target |

Both path segments are organisation-scoped DNS slugs (`InstanceName`), not GUIDs.

### Response `200 OK` or `409 Conflict`

The body is a **DeploySummary** — the same shape returned by the versioned upload
endpoints (`/skills`, `/workflows`, `/tools`, `/memory`, `/schema`):

```json
{
  "items": [
    {
      "resourceType": "Skill",
      "code": "BRA201",
      "action": "Updated",
      "incomingVersion": 3,
      "storedVersion": 2,
      "message": null
    },
    {
      "resourceType": "Workflow",
      "code": "WF-CHAT",
      "action": "VersionConflict",
      "incomingVersion": 1,
      "storedVersion": 4,
      "message": "Workflow 'WF-CHAT' not updated: incoming version 1 is less than stored version 4."
    },
    {
      "resourceType": "Tool",
      "code": "list_inbox_entries",
      "action": "Created",
      "incomingVersion": 1,
      "storedVersion": 0,
      "message": null
    }
  ]
}
```

| Field | Notes |
|---|---|
| `action` | `Created`, `Updated`, or `VersionConflict` |
| `incomingVersion` | Version on the source (template) |
| `storedVersion` | Version on the destination before the operation (0 when Created) |

**Non-conflicting resources are always applied**, even when the response is
`409`. Treat `409` as a warning (same as CLI redeploy), not an abort.

### Status codes

| Status | Meaning |
|---|---|
| `200 OK` | Sync succeeded with no version conflicts |
| `400 Bad Request` | Source and target are the same instance |
| `401 Unauthorized` | No authenticated organisation |
| `403 Forbidden` | Caller lacks Admin role |
| `404 Not Found` | `{targetInstance}` or `{sourceInstance}` not found in the caller's organisation |
| `409 Conflict` | At least one resource was a `VersionConflict`; all other applicable resources were still applied |
| `500 Internal Server Error` | Unexpected failure; the whole operation is rolled back (no partial state) |

Errors (other than the DeploySummary `409` body) use the standard Management API
shape: `{ "error": "message" }`.

---

## 2. Version precedence

Applied independently per versioned resource (same rule as BRA201 §9 / CLI
redeploy):

| Condition | Outcome |
|---|---|
| Resource absent on destination | **Created** (always) |
| Source version **≥** destination version | **Updated** |
| Source version **<** destination version | **VersionConflict** — destination left untouched |

### Versioned resources

| Layer | Key | Notes |
|---|---|---|
| SkillBooks | `(BrainId, Code)` | Categories reconciled even when the book itself conflicts (skills are independently versioned) |
| Skills | `(BrainId, Code)` | Embeddings copied or nullified — see §4 |
| ToolGroups | `(BrainId, Name)` | Upserted by name (not independently versioned) |
| Tools | `(BrainId, Name)` | ToolParameters replaced wholesale when the tool is Created/Updated |
| Workflows | `(BrainId, Code)` | WorkflowTools replaced wholesale when the workflow is Created/Updated; Tool IDs re-linked by **tool name** on the destination |
| Blueprints | `(BrainId, Code)` | Categories, entries, chunks, and chunk embeddings replaced wholesale when the blueprint is Created/Updated |

### Versionless resources (upsert-always)

`EntityType`, `EntityVariableKey`, and `UnitOfWorkType` have no version field.
They are always created or refreshed (`IsDeleted` reset to `false`). They never
produce `VersionConflict`. Versions in the summary are reported as `0`.

---

## 3. What is synced / what is not

### Synced (configuration layer only)

- Tools: ToolGroups → Tools → ToolParameters
- Workflows → WorkflowTools (re-linked to destination Tool IDs by name)
- Skills: SkillBooks → SkillCategories → Skills
- Memory: Blueprints → BlueprintCategories → BlueprintEntries → EntryChunks → ChunkEmbeddings
- Schema types: EntityTypes → EntityVariableKeys; UnitOfWorkTypes

### Not touched

| Layer | Why |
|---|---|
| **BrainEnvironmentVariables** | Not versioned; destination secrets stay as-is (unlike clone, which copies/merges them — BRA205) |
| **Brain header** | `InstanceName`, `ApiKey`, `EmbeddingModel`, display name — left unchanged on the destination |
| **Runtime data** | WorkflowRuns, Entities, UnitsOfWork, InboxEntries, InboxTasks, etc. |

### Transactionality

The whole sync runs in a **single database transaction**. Any failure rolls back
completely; partial updates do not persist.

---

## 4. Embeddings

Skills, tools, and blueprint chunk embeddings are portable only when both brains
share the same `EmbeddingModel`.

| Condition | Behaviour |
|---|---|
| `source.EmbeddingModel == target.EmbeddingModel` | Copy embedding bytes / model / dimensions from source |
| Models differ | Nullify skill/tool embeddings; omit chunk embedding rows so the destination can regenerate on a later index/deploy |

Update-from does **not** change the destination's `EmbeddingModel`.

---

## 5. Typical lifecycle

```text
1. Clone template → new instance          POST /brains/{template}/clone
2. Destination diverges                   (learning, deploys, version bumps)
3. Template gains newer config            (authors bump versions on the template)
4. Pull template improvements forward     POST /brains/{destination}/update-from/{template}
```

```bash
# Pull newer template config into a previously cloned staging brain
curl -X POST "$TELOS_API_URL/brains/salesmate-staging/update-from/salesmate" \
  -H "X-Telos-Api-Key: $TELOS_ORG_API_KEY"
```

Inspect the DeploySummary: `Created` / `Updated` rows were applied;
`VersionConflict` rows mean staging already had a newer version and was left
alone. Destination `.env` / environment variables are unchanged — keep managing
those via deploy or `POST /brains/{instance}/environment-variables` (BRA202).

---

## Related skills

- **BRA201** — schema authoring and deploy versioning (same precedence rule)
- **BRA202** — environment variables (left untouched by update-from)
- **BRA205** — cloning a brain (creates the destination twin this endpoint updates)
- **BRA401** — Management vs Execution API authentication
