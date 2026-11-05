---
name: Playbook to schema
code: BRA220
version: 2
description: How to turn a playbook into brain schema — channels, node types,
  connections, and notes. Load this when building schema from a playbook.
  Designing the playbook is BRA221. File format is BRA219. Field formats stay
  in BRA209, BRA214, BRA215, and BRA217.
---

# Playbook to schema

A playbook is a picture of a business process, written so a person can plan
one job at a time. How to design that picture is **BRA221**. This skill
starts once the picture exists. The brain schema is the mechanics that
process will run on. Read every node, every connection, and every note. Then
decide what the schema needs. Do not make one schema file per canvas node,
and do not copy the playbook's split. Five playbooks about pushing financial
data into an accounting system may still be one workflow and one tool.

This pass builds the skeleton: what is a connector, what is a tool, what is
a workflow, and what an agent is allowed to call. It does not fill in the
expertise. Best practice belongs in a skill, and that skill can stay a short
stub until someone teaches it.

How to read the YAML: **BRA219**. How to write each file, load only the one
you are about to write: connectors **BRA209**, tools **BRA214**, workflows
**BRA217**, skills **BRA215** and **BRA208**.

## 1. Channels

The canvas is four systems. The channel says which system a node belongs to.
The node `type` says what it is. Read both.

| `channel` | System | What it means for schema |
| --- | --- | --- |
| `engagement` | System of Engagement | Where the process meets the world. An application here is a connector. A human action here is someone starting the work, not a file. |
| `action` | System of Action | Agents, and the actions they take. An agent is the caller. |
| `intelligence` | System of Intelligence | The brain schema. Workflows, skill books, and the decisions a workflow makes. |
| `record` | System of Record | Where the data lives. An application here is a connector. |

## 2. Applications

A node with `type: connector` is an application. In the schema an application
is a connector (**BRA209**).

Do not create `connectors/{name}.yml`. Connectors arrive by deploy or by
Import connectors. If that connector is not already in the schema, skip its
tools and say so. If it is present, its tools go under
`tools/{connector-name}/`.

## 3. Agents

A node with `type: agent` is the caller. Grok, Claude, and a person driving
an agent are all agents.

- A connection from an agent straight to an application is local setup. The
  agent talks to that application on its own machine. Do not add schema for
  that link.
- A connection from an agent into the intelligence layer — to a workflow, or
  to anything else the brain exposes — is schema. Anything the intelligence
  layer exposes to an agent is an **MCP workflow** (`type: MCP`, **BRA217**).
  That MCP workflow is the surface the agent calls.
- For a workflow to be called through that surface it needs a **tool
  definition**. The tool is a workflow tool (**BRA214**) whose `workflow.code`
  is the workflow the agent is reaching. List that tool on the MCP workflow.
  The target workflow is normally `type: TOOL`.

Ask this of every agent connection: does the agent talk to the application
itself, or does it go through a workflow? Direct to the application is not
schema. Through a workflow is an MCP workflow plus a tool.

## 4. Workflows

A node with `type: workflow` is a workflow in the schema (**BRA217**).

A workflow is a guardrail. It holds the instructions and the list of tools.
When the words become expertise or best practice, move that text into a
skill. Skills are what the brain can iterate and learn on. The workflow
should say when to use the skill, not contain the expertise itself.

- If the workflow always depends on one skill, put that skill in
  `injected-skills`. Do not make the run search for it. Searching costs turns.
- If which skill applies depends on the case, put `find_available_skills` and
  `get_skill` on the workflow, and write in the instructions that searching
  for a skill is a required step before the work. Do not inject a skill the
  run might not need.

## 5. Skill books

A node with `type: skill-book` says this process uses that book. Build the
plumbing: the book if it is missing, and a stub skill for each area the
playbook names. The stub says when to use the skill. It does not teach the
practice. Leave the content to be learned later.

Then wire the workflows that use the book, using the inject-or-search rule
in §4.

## 6. Actions

A node with `type: action` is a step in the picture. `kind` says what kind of
step. The channel says where it sits. An action is often a way of drawing a
flowchart, not a separate schema file.

| Where it sits | What to write |
| --- | --- |
| `kind: human`, on engagement or action | A person giving an instruction to an agent. Not a schema file. |
| `kind: tool`, or a connection that lists `tools` | A tool definition. The line can name the tool as well as the node. |
| An action on the intelligence channel | Part of the workflow it hangs off. |
| Several actions leaving one workflow | Still one workflow. Triage that can do A, B, or C is one workflow whose instructions say: decide A, B, or C. If A calls a tool and B writes memory, those are branches in that same workflow. Do not create three workflows. |
| `kind: scheduled` or `kind: ai` | A step inside the workflow that owns it, unless the picture already has a distinct workflow node for it. |

## 7. Tools hide the implementation

Name a tool for the caller, not for the machinery behind it.

A tool called "read financial info" is what the caller needs when they want
financial information. The caller does not need to know that the workflow
behind it can reach four Xero organisations and two Stripe accounts. That
detail stays in the workflow and the connectors. The tool description says
when to call it and what the caller must supply.

## 8. Notes

Read the notes on every node and every connection. They are the conversation
that produced the picture. "This can only be done by Bruce" is not a rule
about a person named Bruce if Bruce is the CFO. Write the role. Do not put a
person's name into a tool, a workflow, or a skill when the note is really a
role, a permission, or a condition.

Prefer the general rule. A schema that names today's exception will not
survive the next client.

## 9. What this pass does not create

- `connectors/{name}.yml`
- Blueprint entries. If a blueprint node is on the canvas, name it in the
  summary and leave it.
- One file for every node.
- One workflow for every branch of an action.
