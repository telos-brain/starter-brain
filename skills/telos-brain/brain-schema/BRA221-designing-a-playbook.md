---
name: Designing a Playbook
code: BRA221
version: 2
description: How to turn one problem into a playbook on the four-channel
  canvas. Load this when a user wants to solve a problem and the first
  artefact is a playbook YAML. Decides the system, the nodes, the notes,
  and the connections. The file format is BRA219. Turning the playbook
  into schema is BRA220.
---

# Designing a playbook

When someone describes a problem, the first artefact is a playbook: one YAML
file that pictures that one job. This skill decides what to draw. The file
format is **BRA219**. Turning a finished picture into workflows, tools, and
skills is **BRA220**, and it comes after the picture is agreed.

A playbook is a simple flowchart of one job. It is how a messy process gets
systematised. The canvas has four systems, left to right. Aim to put
something in each system the job actually uses, and keep each system to
between one and five nodes. Past that, the job is more than one playbook, or
the extra detail belongs in a note.

## 1. One job

Name the job in one sentence: "This playbook does X."

That sentence is the `explanation`. The file name is the same job in
kebab-case (`update-sale-from-email`). Write `playbooks/{name}.yml`.

Split before you draw when the sentence needs a second, unrelated outcome.
Five problems are five playbooks. Variations of the same job — the same
systems, the same outcome, a different case — stay in one playbook, and the
cases go in notes.

## 2. The four systems

The channel is decided by the job, not by the product's brand. Salesforce can
be engagement, record, or both. Google Drive can be either. An application
that does both jobs in this playbook is two nodes.

| `channel` | System | What belongs here |
| --- | --- | --- |
| `engagement` | System of Engagement | Where the organisation meets a customer or the outside world. |
| `action` | System of Action | The actor, and the actions that show how the work moves. |
| `intelligence` | System of Intelligence | The brain: workflows, skill books, blueprints. |
| `record` | System of Record | Where the truth of this job is stored. |

### Engagement

Email, calendar, Slack, a meeting note-taker, and sending an invoice are
engagement. They are applications (`type: connector`). A person starting the
work can be an action on this channel.

An engagement application connects to an agent. It does not connect to the
intelligence layer. The agent is what reaches the brain.

### Action

This is where the actor sits (`type: agent`): Claude, Grok, or a piece of
software that acts as the agent. One actor per playbook. A later step taken
by that same actor is an action, not a second agent.

Actions (`type: action`) are how the flowchart is drawn. They are useful on
the picture and they do not become their own schema files (**BRA220**). Use
one for:

- The thing that starts the job. A schedule, a person, or another trigger.
  The usual home is this channel. The start often reads from an engagement
  application or from a system of record; that read is a connection into the
  action.
- A short choice in the intelligence layer: option A, option B, option C as
  three actions under the workflow that chooses.

`kind` says who does the action. A schedule omits `kind` (it defaults to
scheduled). A person is `human`. The keys are in **BRA219**.

### Intelligence

Workflows, skill books, and blueprints live here. A workflow is the brain's
part in this one job. A skill book or a blueprint is drawn when this job
uses it.

MCP is not a node and not a fifth system. An MCP connection is the line from
the agent to the workflow it calls.

A choice with a few visible outcomes can be three action nodes on this
channel, under the workflow. A choice with a long list of outcomes is one
workflow node. The outcomes go in that node's notes.

### Record

An accounting system is a system of record. So is the application that holds
the canonical object of this job: the sale, the invoice, the file of record.

The same product can sit here and on engagement when the job uses it both
ways. Two nodes, and the notes on each say which job that node is doing.
One node when it only does one of those jobs.

## 3. What is a node

Draw a node for:

- Each application, once for each role it plays in this job.
- The agent.
- The start of the job.
- Each workflow this job calls.
- A skill book or blueprint this job uses.
- A branch, as an action, when there are only a few and seeing the choice is
  the point of the picture.

Leave it as a note, not a node, when it is:

- One of many outcomes.
- A tool. Name tools on the connection, or in a note. Draw a tool as its own
  action only when that tool is a step the picture has to show.
- A condition, a role, an exception, or field-level detail.
- MCP. That is the connection from the agent to a workflow.

Count the nodes in each system. One to five is the range. Above five, fold
the extras into the notes of the node they belong to, or split into a second
playbook. A system the job never touches stays empty. Say that in the
explanation. Do not invent a node to fill a column.

## 4. Notes

Notes are where the process is allowed to be detailed. The canvas stays a
simple flowchart because the detail sits on the node.

A workflow that can end ten ways is one node. Its notes name the ways. The
notes on a connection hold the condition for that line. Write a role ("the
finance lead") when the note is about who is allowed to do the step.

## 5. Connections

A connection is the direction information moves. `from` is where it is. `to`
is where it goes. `title` says what is happening, in a few words. An arrow
head at either end is optional and stays off unless that end needs the
direction drawn. The keys are in **BRA219**.

| What is happening | Draw |
| --- | --- |
| An agent or an action reads an application | Application → agent or action. "Reads the inbox." |
| The start hands work to the agent | Action → agent. |
| The agent asks the brain to do the job | Agent → workflow. |
| The brain writes the record | Workflow → record. |
| The agent writes the record itself, with no workflow in between | Agent → record. |
| A workflow chooses among a few visible options | Workflow → each of those actions. |

There is no connection from an engagement application to a workflow. The
engagement application connects to the agent, and the agent connects to the
workflow.

A line may run right to left when an action reads a record, or reads an
inbox. The rest of the job then continues left to right.

## 6. How the picture reads

The columns default to left to right: engagement, action, intelligence,
record. A playbook can store a different order. Do not move a node into
another column to make a line shorter. Move it because that is the system it
belongs to.

Time runs top to bottom. What happens first sits higher. What happens later
sits lower. Two nodes that are the same moment — an inbox and the action
that reads it — share a y. The next step is lower.

Space the nodes so the lines and their titles can be read:

- Centre a node in its column (`x` about `0.5`).
- Put the first moment near `y: 0.2`, and each later moment about `0.18`
  lower.
- Keep nodes off the edges: `x` between `0.2` and `0.8`, `y` between `0.15`
  and `0.85`.
- When two lines would cross, move the later node down.

`x` and `y` are fractions. The exact fields are in **BRA219**.

## 7. Build it in this order

1. Restate the problem as one sentence. If you cannot, name the separate
   playbooks and design one of them.
2. Name the file for that sentence.
3. List every application. For each, decide engagement, record, or both,
   from this job.
4. Name the one agent.
5. Find what starts the job and put that action in the system of action.
6. Put the brain's part on intelligence: the workflow, and a skill book or
   blueprint only when this job uses it. Put a long decision in the
   workflow's notes.
7. Put the system of record on the record channel.
8. Draw the connections in §5. Title each one.
9. Count. More than five nodes in a system means collapse or split. Read the
   picture from the top: the one job should be visible without opening the
   notes.
10. Write the file with **BRA219**. Set `schema-name` when the connector or
    workflow code is already known.

## 8. A worked picture

The job: when a customer emails a change to a sale, update that sale in
Salesforce. Salesforce is the record of the sale. This job does not talk to
the customer inside Salesforce, so Salesforce is not also on engagement. The
many ways a sale can change stay in the workflow note.

```yaml
name: update-sale-from-email
title: Update a sale from email
explanation: |
  When a customer emails a change to a sale, update that sale in Salesforce.
nodes:
  - id: n1
    type: connector
    channel: engagement
    name: Email
    x: 0.5
    y: 0.22
    notes: |
      Customer email about a sale.
  - id: n2
    type: action
    channel: action
    name: Check inbox
    x: 0.5
    y: 0.22
    notes: |
      Starts the job each morning.
  - id: n3
    type: agent
    channel: action
    name: Claude
    x: 0.5
    y: 0.4
    notes: |
      Reads the email and asks the brain to update the sale.
  - id: n4
    type: workflow
    channel: intelligence
    name: Process sale update
    x: 0.5
    y: 0.58
    notes: |
      One workflow. It can record a new sale, change an amount, cancel,
      or leave an unknown sender for a person. Those cases stay here.
  - id: n5
    type: connector
    channel: record
    name: Salesforce
    x: 0.5
    y: 0.76
    notes: |
      System of record for the sale.
connections:
  - from: n1
    to: n2
    title: Reads the inbox
  - from: n2
    to: n3
    title: Passes the email
  - from: n3
    to: n4
    title: Asks the brain to process the sale
  - from: n4
    to: n5
    title: Updates the sale
```

If Claude writes Salesforce itself, remove the workflow and connect Claude to
Salesforce with the title "Posts the sale". If the picture exists to show a
choice, put two or three actions under the workflow on intelligence and
connect the workflow to each. Past a few options, delete those actions and
write the options in the workflow note.

## 9. Then stop

This pass produces the playbook file. It does not author workflows, tools, or
skills. When the picture is agreed, **BRA220** is the pass that turns it into
schema.
