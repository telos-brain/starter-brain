---
name: Tool Audit and Approval
code: BRA218
version: 4
description: How an optional audit block on a tool records the call on Pro and
  Enterprise, and how requires-approval pauses the run until a person approves
  or declines. Load this when adding audit config, explaining a paused run, or
  using the Brain audit log.
---

# Tool Audit and Approval

An optional `audit:` block on a tool records that call on the Brain. On Pro and
Enterprise, a call with `requires-approval: yes` does not run until a person
approves or declines it. PAYG ignores the block. Tool YAML shape: **BRA214**.
Template tags: **BRA204**. Run statuses: **BRA403**.

This is not tool `roles:` (**BRA214**), not the membership access role
(Admin / Member), and not an Inbox task waiting for MANUAL approval.

## 1. When a call is audited

The block is honoured only when the organisation plan is **Pro** or
**Enterprise**.

| Situation | What happens |
| --- | --- |
| No `audit:` block, or the block is empty | Nothing is written. The tool runs as usual. |
| Plan is PAYG, or the organisation has no plan | Nothing is written and the run does not pause, even if the YAML has `audit:`. |
| Pro or Enterprise, audit configured, approval not required | A **Recorded** row is written, then the tool runs. If that write fails, the tool still runs. |
| Pro or Enterprise, `requires-approval: yes` | A **Pending** row is written, the tool does **not** run, and the run becomes **AwaitingApproval**. If the pending row cannot be stored, the tool does not run. |
| A simulation run | Nothing is written, including when the tool would otherwise require approval. |

A missing member id does not fail the tool. An organisation API key still
pauses and emails when approval is required; the row simply has no initiating
member.

The check runs before the tool executes, on API, MCP, system, and workflow
tools, on nested `run_workflow`, on pre-called input-tools, and on MCP
`tools/call`. It runs after the simulation check.

## 2. Authoring the block

Put `audit:` on the tool file, beside the one execution block. Omit it to turn
auditing off. An empty block is the same as omitting it.

```yaml
name: send_invoice
description: Sends an invoice.
version: 1
audit:
  title: Send invoice to {{arguments.customer}}
  body: |
    Amount: {{arguments.amount}}
    Customer: {{entity.name}}
  requires-approval: yes
  approver-role: Finance
  approver-email: finance@example.com
api:
  method: POST
  path: https://billing.example.com/invoices
```

| Field | Meaning |
| --- | --- |
| `title` | Short label stored on the row. Markdown. Template tags allowed. A blank render falls back to the tool name. |
| `body` | Longer markdown stored on the row. Template tags allowed. Unresolved tags are blank. |
| `requires-approval` | `yes` or `no`. YAML `true` / `false` is accepted. Anything else fails deploy. Omit it, or set `no`, to record without pausing. |
| `approver-role` | Optional tag-role label. Every organisation member who holds that label is emailed. |
| `approver-email` | Optional. Emailed only when that address belongs to a member of the organisation. |

`approver-role` and `approver-email` may be used alone, together, or not at
all. With neither, a required approval still pauses; nobody is emailed, and an
organisation Admin opens the audit log to decide.

`title` and `body` are rendered **before** the tool runs. `{{result.*}}` is
empty. `{{arguments.key}}` is that call's arguments (case-insensitive). The
same Template Service scopes as elsewhere (`entity`, `unitOfWork`, `input`,
`inboxEntry`) are available. `{{arguments.*}}` is for this block only. It is
not populated in workflow Instructions or in `response-markdown`.

Deploy, extract, clone, and update-from copy the block. Extract omits `audit:`
when nothing is set, and omits `requires-approval` when it is `no`.

## 3. Pause

On approval the run stays **AwaitingApproval**. It is not Completed, Failed,
AwaitingInput, or Scheduled.

- A synchronous stream ends. The HTTP connection is not held open.
- An async run returns without posting the callback and without settling the inbox.
- MCP `tools/call` returns immediately and leaves the run waiting. An API, system, or outbound MCP tool that requires approval returns a run id and `get_mcp_tool_result` instructions, instead of holding the HTTP call. A workflow tool does the same unless its `execution-mode` is `sync` (**BRA214**): `sync` waits for the child, including through approval. When the polled run is **AwaitingApproval**, `get_mcp_tool_result` says the tool is waiting for approval and will resume once approved. The text "Awaiting approval." is not a finished tool result. If a run was closed with that text, approving or declining reopens it.
- A nested workflow tool that waits parks the child and the parent. Approving the child continues the parent. A nested workflow tool set to `async` does not park the parent: the parent already has the kick-off text and may finish while the child is still waiting for approval.
- The runs list treats AwaitingApproval as in progress, including the status
  filter. The timeline draws it hollow, the same as AwaitingInput. It is not a
  chat session and it is not a learning-eval target.

Approving or declining is atomic. A second decision does nothing. Decline
requires a reason.

## 4. Who is emailed, and who may decide

Email uses the same Postmark path as invites. If Postmark is not configured,
the email is skipped. The row is still written and the run still pauses.

Recipients, deduplicated by address:

- `approver-email`, only when that address is an existing organisation member (case-insensitive).
- Every member whose tag roles contain `approver-role` (case-insensitive). Tag roles are the labels on the membership, not tool `roles:` and not Admin / Member.

The message says approval is required before the tool can be used, includes the
rendered title, and links to `/brains/{instance}/audit/{id}`.

These people may open that row and approve or decline, even without Full access
to the brain:

- an organisation Admin who can open the brain
- the member whose email matches `approver-email`
- a member who holds `approver-role`

The audit **list** stays Admin-only. On PAYG the audit API is not found and
the Audit log nav is hidden.

## 5. Resume

Approve runs the original call, then the workflow continues. Decline does not
run the tool. The tool result is:

```text
Declined by {actor name}: {reason}
```

The workflow can retry or stop from that text. A native tool the provider has
already executed (Claude server tools such as web search) is not executed
again on approve; the run continues. Declining still supplies the declined
text. OpenAI and xAI web search run inside the provider and are not a separate
audited call.

Resume is queued on that brain's own queue. It does not use the scheduled-run
queue. A second resume does nothing while the first is still running.

`get_mcp_tool_result` does not treat "Awaiting approval." as the tool's
answer. While the audit is still pending it says the tool is waiting. Once
the audit is approved it says the tool is now running, and once declined it
says the run is finishing. The next check returns the tool result, or the
decline text.

## 6. Audit log

Brain section **Audit log**, Pro and Enterprise only.

**List** (Admin): date, title, status, initiating user, tool name. Filter by
status and date. Newest first. Page size 100. A row opens the detail.

**Detail:** title, body rendered as markdown, status, who started it and when,
who decided and when, approver role and email. Parameters are a key/value
table, collapsed until opened. An empty parameter set is an empty state.
Pending rows have Approve, and Decline opens a dialog that requires a reason.

Status colours: Pending amber, Approved green, Declined red, Recorded neutral.

## 7. Retention

The midday message-retention job clears stored **parameters** once
`RetainUntilUtc` has passed. Title, body, status, actors, and timestamps stay.
The audit row is not deleted. Message deletion is unchanged.
