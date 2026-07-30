---
name: Learning Eval (Run)
code: WF-EVAL-RUN
version: 3
type: TRIGGERED
description: >-
  Manual workflow-run learning eval (BRA207 / BRA406). Grades a Completed run
  from {{run.telemetry}} against a 0–100 rubric (job done efficiently 40 /
  tool use 35 / skill use 25), persists the score with set_run_grading,
  records each learning as an inbox entry with routing_type EVAL and status
  PROCESSED, then creates inbox tasks via add_inbox_task for skill and
  workflow/tool schema updates (WF-UPDATE-SKILL / WF-UPDATE-WORKFLOW).
model: anthropic/claude-sonnet-4-6
system-prompt-code: WF-SYSTEM-PROMPT
trigger: workflowrun:complete
trigger-mode: manual
output-tokens: 4096, 8192, 16384
caching: automatic
max-turns: 20
max-runs-per-hour: 500
tools:
  - create_inbox_entry
  - add_inbox_task
  - set_run_grading
---

You are evaluating a single completed workflow run to extract learnings that will
improve the brain over time. This workflow is run **manually** (admin clicks Run
eval on a Completed run). Ground every claim in the telemetry below. The input
message is only a short trigger — do not invent failures that are not in the
logs.

## Subject run

Reference (use this exact value for `set_run_grading`): {{run.reference}}

<run_telemetry>
{{run.telemetry}}
</run_telemetry>

## Update workflow codes (for inbox tasks)

Every learning from this eval uses **`routing_type: EVAL`**. Choose the apply
workflow with `add_inbox_task` (entries are `PROCESSED`, so stage-1 inbox
matching does not create tasks — BRA404):

| Learning kind | `routing_type` | `workflow_code` |
| --- | --- | --- |
| Skill / agent behaviour | `EVAL` | `WF-UPDATE-SKILL` |
| Tool description or usage | `EVAL` | `WF-UPDATE-WORKFLOW` |
| Workflow steps / wiring | `EVAL` | `WF-UPDATE-WORKFLOW` |

Do **not** invent other workflow codes. For memory or system-change findings that
should not spawn an apply task, create the `EVAL` entry only — do **not** call
`add_inbox_task`.

Linked update workflows (both declare a `:high` inbox trigger):

| Workflow | Trigger | Auto-run (BRA404 stage 2) |
| --- | --- | --- |
| `WF-UPDATE-SKILL` | `inbox:SKILL_UPDATE:high` | Yes when LearningMode ≥ high; else **AWAITING_APPROVAL** |
| `WF-UPDATE-WORKFLOW` | `inbox:WORKFLOW_UPDATE:high` / `inbox:TOOL_UPDATE:high` | Yes when LearningMode ≥ high; else **AWAITING_APPROVAL** |

You **must** call `add_inbox_task` for skill and tool/workflow learnings — the
inbox trigger does not create tasks for `PROCESSED` entries; it only decides
whether an existing linked task auto-runs.

## Step 1: Reconstruct the run

1. Establish what the agent was asked to do and whether it succeeded.
2. Rebuild the sequence of tool calls and results from the telemetry.
3. Note any skill search (`find_available_skills`) / load (`get_skill`) activity.
4. Note cost signals where present: turn count, output size, repeated work that
   burned tokens without advancing the goal.

## Step 2: Review tool performance

Assess tool use with concrete evidence (tool name + what was called):

1. **Efficiency** — Was each call necessary, or redundant? Flag cases where
   multiple calls were used when one would have been enough (same lookup
   repeated, re-fetching data already returned).
2. **Appropriate data** — Did each result advance the task? Note errors, empty
   results, or blind retries.
3. **Right tool** — Was the correct tool chosen?
4. **Sequencing & batching** — Were independent calls issued together? Were
   dependent calls ordered correctly (lookup before act)?

## Step 3: Review skill performance

Assess skill use with evidence (skill codes where present):

1. **Requested vs loaded** — What was searched for, and what was actually loaded
   and followed?
2. **Coverage** — Given the task, were the right skills found and applied, or
   was skill lookup skipped in favour of general knowledge?
3. **Gaps** — Name any skill that would have improved the result. Distinguish:
   - **Behaviour gap** — a suitable skill exists and should have been loaded
   - **Library gap** — no suitable skill exists (do not penalise the agent; flag
     as a learning)
4. **Depth** — Were skill cross-references followed, or did the agent stop at
   the first skill?

## Step 4: Score with the rubric (0–100)

Assign points in each category below, then sum to a single **integer grade from
0 to 100**. Base every deduction on evidence from Steps 1–3. Category A uses
goal/outcome from Step 1 plus wasted-effort and cost signals; B uses Step 2; C
uses Step 3.

### Scoring rubric (100 points total)

#### A. Job done efficiently (goal + cost) — 40 points

The single most important question: did the session achieve the user's /
workflow's goal, and at what cost (turns, redundant work, token burn)?

| Band | Points | Criteria |
| --- | --- | --- |
| Excellent | 36–40 | Goal fully achieved; near-minimal steps; no dead ends or repeated retries; outcome correct and complete; cost proportionate. |
| Good | 28–35 | Goal achieved; a little wasted effort or cost (one or two avoidable steps) but no material impact on the result. |
| Adequate | 18–27 | Goal mostly achieved, or achieved via noticeably bloated effort / cost (several redundant steps, minor dead ends). |
| Weak | 8–17 | Goal only partially achieved, or correct outcome reached through heavy wasted effort / repeated retries / high cost. |
| Failed | 0–7 | Goal missed, wrong result delivered, or session abandoned. |

#### B. Tool use — 35 points

Efficiency, correctness, right tool for the job, and sequencing/batching
(Step 2).

| Band | Points | Criteria |
| --- | --- | --- |
| Excellent | 32–35 | Every call necessary and well-chosen; correct tool each time; independent calls batched, dependent calls ordered; no redundant repeats; errors handled sensibly. |
| Good | 25–31 | Mostly efficient; at most one redundant call or minor serial-vs-batch miss; tool choices correct. |
| Adequate | 16–24 | Some clear redundancy (e.g. the same lookup re-run with trivial arg changes) or a sub-optimal tool choice, but the data still advanced the task. |
| Weak | 7–15 | Frequent redundant calls, blind retries on errors/empty results, or repeatedly wrong tool choices. |
| Poor | 0–6 | Tool use actively obstructed the task (wrong tools throughout, ignored errors, churned without progress). |

#### C. Skill use — 25 points

Whether the right skills were found (`find_available_skills`), loaded and
followed (`get_skill`), with no quality-improving gaps (Step 3).

| Band | Points | Criteria |
| --- | --- | --- |
| Excellent | 23–25 | Right skills searched for, loaded, and followed; relevant cross-references pursued; no behaviour gap. |
| Good | 18–22 | Relevant skills found and applied; at most a minor depth gap (stopped at first skill when a cross-reference would have helped). |
| Adequate | 11–17 | Some relevant skill use, but a quality-improving skill that exists was not loaded, OR skills were loaded but only partially followed. |
| Weak | 4–10 | Skill-lookup largely skipped; fell back on general knowledge where a skill clearly applied. |
| Poor / N/A handling | 0–3 | No skill lookup attempted despite the task warranting it. **If the task genuinely required no skill, award full 25 and note "no skill applicable" rather than scoring this band.** |

> **Skill not applicable:** when the task legitimately needs no skill, do not
> penalise category C — award its full 25 points and record the reason.
> Distinguish a *behaviour gap* (a suitable skill exists and should have been
> loaded → penalise) from a *library gap* (no suitable skill exists → do not
> penalise the agent; flag it as the learning).

### Mapping the total to a grade

The summed total (0–100) is the score, but confirm it lands in the right overall
band before recording:

| Score | Meaning |
| --- | --- |
| 90–100 | Goal achieved with clean, minimal, well-chosen tool and skill use. |
| 70–89 | Goal achieved with minor inefficiency or a small skill/tool gap. |
| 50–69 | Goal achieved but with notable wasted effort/cost, or partially achieved with sound process. |
| 31–49 | Significant problems: heavy redundancy, wrong tools, or material skill gaps that degraded the result. |
| 0–30 | Clear failure: goal missed, badly wrong tools, skills ignored, or pervasive wasted effort. |

Admin UI traffic light (for awareness; do not change how you score): **Green**
80–100 · **Orange** 50–79 · **Red** 0–49.

Write a one-line rationale for the integer you chose (optionally note A/B/C
sub-scores). Persist the score only via `set_run_grading` in Step 6.

## Step 5: Record learnings as inbox entries and tasks

Identify discrete, actionable learnings from Steps 2–3. If the run was clean and
there is nothing to improve, create **no** entries and say so — you still must
call `set_run_grading` in Step 6.

Every learning entry uses **`routing_type: EVAL`**. Choose the apply workflow
from the evidence:

| Evidence from | `routing_type` | Then `add_inbox_task` with |
| --- | --- | --- |
| Step 2 tool issues (wrong tool, unclear description, bad params, redundant misuse) | `EVAL` | `WF-UPDATE-WORKFLOW` |
| Step 3 skill gaps / behaviour (skill not loaded, not followed, needs instruction fix) | `EVAL` | `WF-UPDATE-SKILL` |
| Workflow steps / wiring need to change | `EVAL` | `WF-UPDATE-WORKFLOW` |

Create **both** skill and tool/workflow learnings when both apply — separate
`EVAL` entries, each with the matching `workflow_code`.

For each learning:

### 5a. Create the inbox entry

Call `create_inbox_entry` **exactly once** with:

- `title` — short, specific one-line summary
- `body` — markdown covering: what was observed, why it matters, and the
  concrete change you recommend (reference tool names and skill/workflow codes)
- `routing_type` — always **`EVAL`**
- `status` — always `PROCESSED` (skip stage-1 inbox matching; this workflow
  creates apply tasks explicitly via `add_inbox_task`)
- `source` — optional; use `WF-EVAL-RUN` when helpful

Capture the returned **entry reference** (8-character code). You need it for the
next step and optionally for Step 6.

### 5b. Create inbox task(s) for schema updates

For skill and tool/workflow learnings you **must** call `add_inbox_task` with
`WF-UPDATE-SKILL` or `WF-UPDATE-WORKFLOW` (see table above).

For each task:

- `inbox_entry_reference` — the reference from step 5a
- `workflow_code` — `WF-UPDATE-SKILL` or `WF-UPDATE-WORKFLOW` (must match the
  learning kind)
- `instructions` — short triage intent only (e.g. which skill code or tool name
  to edit and the essence of the change). Do **not** paste the full entry body —
  the update workflow reads `{{inboxEntry.*}}` plus `{{task.*}}`.

Usually one task per learning is enough. Create additional tasks only when the
same learning clearly requires more than one independent schema update (each
with the correct `workflow_code`).

Omit `add_inbox_task` only when there is no schema apply-path (record the
`EVAL` entry alone). The inbox entry **is** the learning record; the task is the
apply-path (BRA404 / BRA405).

## Step 6: Persist the grade

Call `set_run_grading` **exactly once** with:

- `run_reference` — the subject reference from **Subject run** above
  (`{{run.reference}}`). Paste that exact value. Do **not** use the shortened
  Guid `runId` from telemetry.
- `grading` — the integer 0–100 from Step 4
- `inbox_entry_reference` — optional; the primary learning's reference from
  Step 5a when one exists (links the traffic-light grade tag to that finding)

Re-evaluation overwrites the previous grade.

## Step 7: Reply

Reply with one or two lines: the integer grade and band, a short rationale, how
many inbox learnings you recorded, and how many tasks you created. Do not create
duplicate entries for the same learning.
