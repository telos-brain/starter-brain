---
name: "Workflow run variables & pre-called input-tools"
code: BRA409
version: 1
description: How harness apps pass string variables into a workflow run via the
  Execution API, how those values surface as {{input.*}} template tags, and how
  workflows declare input-tools to call tools automatically at startup using
  those variables. Companion to BRA403 (run endpoints), BRA204 (input scope),
  and BRA201 §8.0a (YAML schema).
---

# Workflow run variables & pre-called input-tools

This skill is the practical guide for **parameterising a workflow at run time**
from outside the brain schema:

1. Pass a `variables` map on the Execution API run body.
2. Read those values in Instructions (and elsewhere) as `{{input.<key>}}`.
3. Optionally declare `input-tools` so tools run **before** the AI's first turn,
   with parameters filled from those same `{{input.*}}` tags.

See **BRA403** for the full run-endpoint contract, **BRA204** §3.6 for the
`input` scope taxonomy, and **BRA201** §8.0a for the YAML field reference.

---

## 1. Mental model

```
Harness app
  └─ POST /workflows/{code}/run/sync|async
       body.variables  →  WorkflowRun.Variables (JSON)
                              ↓
                     Template `input` scope
                              ↓
              ┌───────────────┴───────────────┐
              │                               │
     {{input.key}} in                 input-tools parameters
     Instructions / prompts           "{{input.key}}" → Tool Router
                                              ↓
                                   <pre_called_tool name="…">
                                   injected before first LLM turn
```

- **Variables** are caller-owned string key/value pairs for one run.
- They are **not** entity variables (BRA402), environment secrets (BRA202), or
  workflow-tool parameters (those are a separate source that merges into the
  same `input` bag — see BRA204 §3.6).
- Omitting `variables` changes nothing for existing callers.

---

## 2. Passing variables via the Execution API

Both sync and async endpoints accept the same optional field:

```json
{
  "inputMessage": "Produce the briefing.",
  "variables": {
    "topic": "Telos Brain Execution API",
    "seed_question": "What is the Execution API used for in one sentence?"
  }
}
```

| Rule | Detail |
| --- | --- |
| Types | Keys and values are **strings** only — no coercion at the API boundary |
| Optional | Omit the field, pass `null`, or pass `{}` → no variables stored |
| Persistence | Non-empty maps are stored on `WorkflowRun` before the run starts |
| Nested runs | Parent variables propagate into `run_workflow` / workflow-tool children |
| Missing keys | `{{input.missing}}` renders blank — never an error |

### Sync example

```http
POST /workflows/WF-INPUT-VARIABLES/run/sync
Authorization: Bearer {brainApiKey}
Content-Type: application/json

{
  "inputMessage": "Produce the briefing.",
  "variables": {
    "topic": "Telos Brain Execution API",
    "seed_question": "What is the Execution API used for in one sentence?"
  }
}
```

### Async example

```http
POST /workflows/WF-INPUT-VARIABLES/run/async
Authorization: Bearer {brainApiKey}
Content-Type: application/json

{
  "variables": {
    "topic": "Telos Brain Execution API",
    "seed_question": "What is the Execution API used for in one sentence?"
  },
  "callbackUrl": "https://app.example.com/brain/callbacks/run-complete"
}
```

Async jobs read variables from the persisted `WorkflowRun` row — you do not need
to put them on the Hangfire payload yourself.

---

## 3. Using variables in workflow Instructions

Once passed, every key is available as `{{input.<key>}}` anywhere the Template
Service runs (Instructions, system-prompt workflow body, tool response /
error markdown, and `input-tools` parameter strings).

```markdown
# Instructions

Topic from the caller: **{{input.topic}}**

Write a short briefing on that topic.
```

Tips:

- Prefer stable, snake_case or kebab-free keys (`widget_reference`, `topic`).
- Document expected keys in the workflow `description` so harness authors know
  what to send.
- Combine with other scopes as usual (`{{entity.name}}`, `{{#unitOfWork.context}}`, …).

---

## 4. Declaring `input-tools` to pre-call tools

When the model should start with **already-fetched** context (instead of spending
a turn calling a tool), declare an `input-tools` block in the workflow
frontmatter:

```yaml
---
name: Input Variables Demo
code: WF-INPUT-VARIABLES
type: RUNNABLE
tools:
  - ask_question
input-tools:
  - variable: seed_answer
    tool: ask_question
    parameters:
      question: "{{input.seed_question}}"
---
```

| Field | Required | Meaning |
| --- | --- | --- |
| `variable` | yes | Injection name — becomes the `name` attribute on the result block |
| `tool` | yes | Tool Router name (`Tools.Name` or system tool name) |
| `parameters` | no | String map; values may include `{{…}}` tags (including `{{input.*}}`) |

### Runtime sequence

1. API `variables` are loaded into the template `input` scope.
2. Each `input-tools` entry runs in declaration order.
3. Parameter values are template-rendered, then dispatched via `IToolRouter`.
4. Results are injected as a user-role context message **before** the first
   Claude turn:

   ```xml
   <pre_called_tool name="seed_answer">…tool result text…</pre_called_tool>
   ```

5. On failure, the run **continues** with a warning block:

   ```xml
   <pre_called_tool name="seed_answer" status="error">Tool call failed: …</pre_called_tool>
   ```

6. Results are capped at 200,000 characters (same limit as LLM-loop tool results).

Extract omits the `input-tools` key entirely when there are no rows — never
writes `input-tools: []`.

### Choosing tools for `input-tools`

- The tool must be routable by name (declared tool or system tool).
- Prefer cheap, deterministic tools for pre-calls; nested workflow tools (e.g.
  `ask_question`) work but start a full child run.
- List tools the AI may still call mid-run under `tools:` / `available-tools:`
  as usual — `input-tools` does not replace that permission envelope.

---

## 5. End-to-end worked example

**Workflow** — `workflows/WF-INPUT-VARIABLES.md` (deployed in this brain schema):

1. Caller sends `topic` and `seed_question` in `variables`.
2. `input-tools` pre-calls `ask_question` with `question: "{{input.seed_question}}"`.
3. Instructions read `{{input.topic}}` and use the injected
   `<pre_called_tool name="seed_answer">` block to write a short briefing.

**Harness call:**

```bash
curl -sS -X POST "$TELOS_API_URL/workflows/WF-INPUT-VARIABLES/run/sync" \
  -H "Authorization: Bearer $BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "inputMessage": "Produce the briefing.",
    "variables": {
      "topic": "Telos Brain Execution API",
      "seed_question": "What is the Execution API used for in one sentence?"
    }
  }'
```

**What the model sees (simplified):**

1. System prompt (if configured)
2. Invoking Instructions (with `{{input.topic}}` already resolved)
3. `<pre_called_tool name="seed_answer">…</pre_called_tool>`
4. User message: `Produce the briefing.`

---

## 6. Related skills & examples

| Resource | Role |
| --- | --- |
| **BRA403** | Run endpoints; `variables` on the shared request body |
| **BRA204** §3.6 | `input` scope taxonomy (API variables + tool params) |
| **BRA201** §8.0a | YAML `input-tools` schema reference |
| `WF-INPUT-VARIABLES` | Live example workflow in `workflows/` |
| `WF-ASK-QUESTION` | Child workflow used by the `ask_question` tool (`{{input.question}}`) |
