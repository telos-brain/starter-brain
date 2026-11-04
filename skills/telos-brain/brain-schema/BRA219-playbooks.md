---
name: Playbooks
code: BRA219
version: 5
description: How to author playbooks/{name}.yml — a named design of a business
  process on the four-channel canvas. Load this when creating or editing a
  playbook file. Playbooks are not workflows, not memory, and not executable.
---

# Playbooks

A playbook documents a business process. It is a sales and design layer: a
named YAML file the canvas can draw and save. It does not run. It is not a
workflow (**BRA217**), not a skill book (**BRA215**), and not a blueprint
(**BRA216**).

Overview of where the file sits: **BRA201**. Schema tools that read and write
it: **BRA203**.

## 1. Where the file lives

```
playbooks/
  accounts-payable.yml
```

- The path is `playbooks/{name}.yml`. No extra folders.
- `{name}` is kebab-case (`accounts-payable`) and must equal the `name` field.
- Playbooks are **discovered by folder**. Do not list them in
  `brain-compose.yml`. A file that is present on deploy is upserted. A
  playbook that is missing from the bundle is left as it is — deploy does not
  delete it.
- There is no `version`. Each deploy overwrites the playbook with that name.
  A previously deleted playbook with the same name is restored.

## 2. File shape

```yaml
name: accounts-payable          # REQUIRED. Kebab-case. Matches the filename.
title: Accounts payable         # optional display label; omit when it equals name
explanation: |                  # optional. What the process is.
  Accounts payable runs on three inbound channels.
nodes:
  - id: n1                      # REQUIRED. Unique within this file.
    type: connector             # REQUIRED. See §3.
    channel: engagement         # REQUIRED. See §4.
    name: Email                 # REQUIRED. Label on the canvas.
    schema-name: gmail          # optional link into this brain's schema
    x: 0.5                      # REQUIRED. 0–1 across the channel. Default 0.5 if omitted.
    y: 0.3                      # REQUIRED. 0–1 down the canvas. Default 0.5 if omitted.
    notes: |                    # optional markdown
      Inbound invoices land here.
    kind: human                 # actions only. Omit for scheduled.
connections:
  - from: n1                    # REQUIRED. A node id in this file.
    to: n4                      # REQUIRED. A node id in this file.
    title: Posts the bill       # optional text, drawn at the middle of the line
    notes: |                    # optional text
      Only approved invoices.
    tools:                      # optional tool names, usually into a connector
      - gmail_search
```

Unknown keys are ignored. Empty `explanation`, `notes`, `tools`, connection
`title`, and `schema-name` are omitted when the file is written back. The
playbook `title` is omitted when it is the same as `name`. `kind` is omitted
when the action is `scheduled`, and is never written on any other node type.

`x` and `y` are fractions of the channel and the canvas, not pixels. Each must
be from 0 to 1.

## 3. Node types

| `type` | What it stands for | `schema-name` |
| --- | --- | --- |
| `connector` | An external system the process touches. | Connector `name` (**BRA209**). |
| `agent` | The actor that does the work. | Workflow `code` (**BRA217**). |
| `workflow` | A process step in the brain. | Workflow `code` (**BRA217**). |
| `skill-book` | Knowledge the process uses. | Skill book `code` (**BRA215**). |
| `blueprint` | Memory the process reads. | Blueprint `code` (**BRA216**). |
| `action` | A scheduled, human, AI, or tool step. Not linked to a schema file. | Omit `schema-name`. |

`kind` is only valid on `action`:

| `kind` | Meaning |
| --- | --- |
| `scheduled` | Default. Omit the key. |
| `human` | A person does this step. |
| `ai` | An AI does this step. |
| `tool` | A tool does this step. |

## 4. Channels

The canvas is four systems. Any node type may sit on any channel; these are
the usual homes.

| `channel` | System | Usual nodes |
| --- | --- | --- |
| `engagement` | System of Engagement | Connector, action |
| `action` | System of Action | Agent, action |
| `intelligence` | System of Intelligence | Workflow, skill book, blueprint, action |
| `record` | System of Record | Connector, action |

## 5. Connections

A connection joins two node ids in the same file. `title` is plain text, drawn
at the middle of the line. `notes` is text kept with the connection. `tools`
lists tool names that travel with that connection, typically the tools of a
connector at one end. Omit `title`, `notes`, and `tools` when they are empty.

## 6. What a playbook is not

- It is not executed and it does not start a workflow run.
- It is not copied by update-from (**BRA206**). Clone does copy playbooks
  (**BRA205**).
- It is not part of a checkpoint.
- It is not listed in `brain-compose.yml`.
