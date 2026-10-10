---
name: Learning Centre
code: BRA414
version: 1
description: How the learning centre turns inbox entries from evals, email,
  meetings, and uploads into trusted learning signals. Weight, clustering, and
  when a signal becomes specific tasks. Inject when triaging or planning an
  inbox entry.
---

# Learning Centre

The learning centre collects signals and turns the ones we trust into specific
work. An entry is not a task. Weight is confidence. Tasks exist only when that
confidence is high enough to act.

## Intent

Signals come from evals, email, meetings, agent runs, and manual uploads. A
source does not by itself decide the value.

- An **eval** is a finding from one run. It is over-specific to that case
  unless it is already a general learning you would apply as written.
- A **meeting**, **email**, or **upload** is unknown until you read it. It may
  hold a skill, a durable memory, a process you can infer, or nothing.
- Be flexible. A skill, a memory, and an inferred process are all valid. The
  absence of one is not a reason to invent another. A name or a one-off detail
  is not a skill. It is memory only when it is a durable fact that will still
  be true later. "How we do X" is a skill, not a memory.

Quality means the learning is still useful after client names, project names,
people, and one-off implementation details are removed — or the fact is stated
and durable. When that is already true, act. When it is not, the entry waits
and gathers weight.

## Mechanics

Every entry starts at **weight 1**. It can sit. Related entries of the same
kind cluster, and the cluster's weight is the **sum** of their weights.
**Weight 5** is the trust line.

- **Already clear.** The text states the practice, the rule, the process, or
  the durable fact, and you do not have to stretch. Set weight to **5** and
  create tasks on this entry. Do not cluster it. A meeting counts only when
  the words themselves state the learning; people discussing a change have not
  stated it. An eval counts only when you would apply it as a general
  learning. An entry already at 5 or above is this case: do not cluster it.
- **Real, but too specific.** One client, one meeting, one implementation, or
  one eval fragment — including an earlier cluster still at weight 2, 3, or 4.
  Leave the weight. Cluster with related open entries of the **same kind**.
  Eval with eval. Meetings, email, and imports with each other. Never mix
  those. Prefer the same workflow, then the same source, then the same entity
  or unit of work, then the same pattern in the title and body. Time alone is
  not relatedness. Include every related open entry; two is the minimum, not
  the target. Do not set the cluster weight yourself — the sum is the quality.
  Name each source reference and its source in the cluster description. If the
  sum reaches 5, create tasks on the **new cluster**, not on the sources. If
  it does not, or if nothing related is open, stop. Add no tasks, and do not
  rewrite the body to say it is waiting.
- **No learning.** Small talk, stale scheduling, truisms, empty or boilerplate
  content. Change nothing. Do not cluster. Do not add tasks.

Do not lower a weight. Do not use any weight other than the starting weight
and 5. Do not raise weight to 5 to force a task onto something that is still
too specific. A cluster reaches 5 because enough specific entries have
accumulated, not because the description sounds finished.

## Tasks

Create tasks only on an entry or cluster at weight 5 or higher. Make them
specific. One distinct change is one task: three skill updates are three
tasks. Each task names its destination and a short instruction. Do not paste
the source into the instruction; the work that applies it reads the entry.

A meeting, or a cluster built from meetings, is not a reason to change how the
brain manages itself. That needs an eval learning or a direct instruction that
states the structural change.

No task is a successful outcome. An unjustified extra task is worse than a
miss. Do not mine a long document for more work.

Tool contracts: **BRA405**, **BRA413**.
