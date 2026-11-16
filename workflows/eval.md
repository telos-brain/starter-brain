---
name: Learning Eval (Run)
code: WF-EVAL-RUN
version: 10
type: EVAL
description: >-
  Manual workflow-run learning eval (BRA207 / BRA406). Reads the subject
  workflow, its system prompt, and the rest of the schema the run depended
  on before grading. Scores a Completed or Failed run from {{run.telemetry}}
  on a 0–100 rubric (job done efficiently 40 / tool use 35 / skill use 25),
  persists the score with set_run_grading, and files each learning as a
  PENDING inbox entry (routing_type EVAL).
system-prompt-code: WF-SYSTEM-PROMPT
trigger: workflowrun:complete
trigger-mode: manual
output-tokens: 8192, 20000, 20000
caching: automatic
max-turns: 25
max-runs-per-hour: 500
tools:
  - create_inbox_entry
  - set_run_grading
  - list_schema_files
  - search_schema_files
  - get_schema_file
  - find_available_skills
  - get_skill
---

You are evaluating a single completed workflow run to extract learnings that will
improve the brain over time. This workflow is run **manually** (admin clicks Run
eval on a Completed or Failed run). The input message is only a short trigger.

Telemetry shows what the agent did, the system prompt it was given
(`resource.gen_ai.system_instructions`), and the workflow instructions it
followed (the first user message). It does not show the tool list or the
output-token ladder. A loop, a repeated tool call, an apology for skipping a
lookup, or a run that stopped on `max_tokens` can come from those
instructions, that system prompt, a missing tool, compaction, or the
output-token ladder. Read the prompt and the instructions in the telemetry,
and the schema those questions still depend on, before you name a cause. A
learning that blames the wrong layer is worse than no learning. Do not invent
failures that are not in the logs, and do not pick a cause just to sound
decisive.

## Subject run

Reference (use this exact value for `set_run_grading`): {{run.reference}}

{{#if entity.name}}
Subject entity: **{{entity.name}}**
{{/if}}

<run_telemetry>
{{run.telemetry}}
</run_telemetry>

The telemetry is the subject run being graded. When you create inbox entries,
take `workflow_name` and `unit_of_work_name` from it. `telos.workflow.path` and
`telos.workflow.code` name the workflow file when a question needs its
frontmatter. `runId` inside the telemetry is a shortened Guid — do not pass it
to `set_run_grading`.

`resource.gen_ai.system_instructions` is the system prompt this run used: a
list of `{ "type": "text", "content": "..." }` parts, already rendered. It is
not a span. When the key is absent, this run had no system prompt.

The first span with `gen_ai.message.role` of `user` is the workflow
instructions this run followed, already rendered. Do not load the workflow
file to read them. Later user messages are the triggering input and anything
the run added after the instructions.

## Step 1: Reconstruct the run

From the telemetry only, establish the facts. Do not explain them yet.

1. What the agent was asked to do, and whether it succeeded, failed, or stopped
   early.
2. The sequence of tool calls and results.
3. Any skill search (`find_available_skills`) or load (`get_skill`).
4. Behaviour that needs a cause: loops, the same tool called again, or the
   agent saying it answered without a lookup it had already made.
5. Compaction: spans with `gen_ai.message.role` of `compaction`, and any later
   turn that acts as if an earlier tool call or result was never there.
6. Output caps: assistant spans whose `gen_ai.response.finish_reason` is
   `max_tokens`, the `gen_ai.request.max_tokens` on each attempt, and whether
   the run Failed because the last attempt was still `max_tokens` or because
   `max-turns` was exhausted.
7. The system prompt in `resource.gen_ai.system_instructions` (every `content`
   part), or the fact that the key is absent.
8. The workflow instructions: `gen_ai.message.content` of the first `user`
   span.
9. Cost signals where present: turn count, output size, work that burned tokens
   without advancing the goal.

## Step 2: Read the picture

The system prompt and the workflow instructions to judge are the ones in the
telemetry. Both are already rendered. Do not load the workflow file to re-read
the instructions, and do not replace the telemetry prompt with the current
system-prompt file: that file may have changed since the run.

Open the workflow file only when a question needs frontmatter the messages do
not carry: the `tools` list, `output-tokens`, `auto-compaction`, or
`system-prompt-code`. Open the system-prompt file only when a learning will
change that source, or when telemetry omitted the prompt and you need to see
whether the workflow names one.

1. Read the first user message as the instructions this run followed. Judge
   `gen_ai.system_instructions` against those instructions and against what
   the run was able to do. When the system-prompt key is absent, the run had
   no system prompt — say so.
2. When the instructions demand a tool, or you are checking the tool list,
   `output-tokens`, `auto-compaction`, or `system-prompt-code`: `get_schema_file`
   on `telos.workflow.path`. If that path is missing, `search_schema_files` by
   `telos.workflow.code`, then read the match. Quote the telemetry text in any
   learning, and name `system-prompt-code` or the workflow code as the file to
   change.
3. When step 1 showed a loop, a repeated call, an apology for a skipped lookup,
   or a compaction span: find the brain's `type: COMPACTION` workflow
   (`search_schema_files`, then read the file whose frontmatter is
   `type: COMPACTION`) and note the subject workflow's `auto-compaction`.
4. When the question is whether a tool or skill was missing or wrongly
   required: `get_schema_file` on the tool definitions the workflow lists, and
   `find_available_skills` / `get_skill` when you need to know whether a useful
   skill actually exists.
5. When the run stopped on `max_tokens`, exhausted `max-turns`, or the ladder
   looks wrong for the job: `get_skill` for **BRA217** (workflow settings,
   including `output-tokens` and `system-prompt-code`) and **BRA212**
   (compaction and token budgets). Use them as the reference for what the
   settings mean. Do not treat either skill as a script.

Answer these before you score. Write the answers in your reasoning. They are
questions, not a catalogue of bugs — follow only the ones the run raises.

- **Can the instructions be fulfilled?** Read the first user message and the
  telemetry system prompt together, against the tools the workflow actually
  lists. A loop or a stall is often an instruction the tools cannot carry out.
- **Is this the right system prompt?** A prompt that demands something the
  workflow does not provide is not automatically a missing tool. Ask whether
  this job needs that thing. A prompt that says skills are mandatory, on a
  workflow with no `find_available_skills` or `get_skill`, is the wrong system
  prompt when the job does not need skills. It is a missing tool when the job
  does need skills and the schema has skills that would have helped. The same
  question applies to any other capability the prompt requires.
- **Did compaction hide a lookup?** An apology for answering without a lookup,
  or the same tool called again, is not automatically a weak model or a bad
  workflow. Compaction can drop tool calls and results. If the summary or the
  dropped span removed a lookup the agent then redid or disowned, the learning
  is the compaction workflow (or `auto-compaction`), not the agent.
- **Did the output ladder anticipate the thinking?** A run that dies on
  `max_tokens` is not "too much thinking" unless the workflow already gave a
  high cap, retried, and the attempts were spent wandering. Often the workflow
  never gave the job room to finish. A useful ladder starts lower and then
  jumps — a few thousand, then the tens of thousands, sometimes repeating the
  high cap — with the size of the jump set by how much thinking and reply the
  job actually needs. Look at the caps in the telemetry and at `output-tokens`
  on the workflow. Recommend a shape, not a single magic number.

Do not file a learning, and do not deduct for tool or skill misuse, until the
relevant questions are answered from the telemetry and from any file the
question required. If two causes are still plausible, keep both and say what
you could not separate.

Even a clean run: read the telemetry system prompt and the first user message.
File a learning only when they will break a later run. Do not file a
speculative problem this telemetry does not show. Do not open the workflow
file on a clean run.

## Step 3: Review tool performance

Assess tool use with concrete evidence (tool name + what was called), in light
of Step 2:

1. **Efficiency** — Was each call necessary, or redundant? A repeat is not
   redundant when compaction had removed the earlier result. Flag genuine
   redundancy: the same lookup again when the result was still in context.
2. **Appropriate data** — Did each result advance the task? Note errors, empty
   results, or blind retries.
3. **Right tool** — Was the correct tool chosen from the tools the workflow
   actually provided? A tool the instructions demand but the workflow does not
   list is a schema finding from Step 2, not a bad choice by the agent.
4. **Sequencing & batching** — Were independent calls issued together? Were
   dependent calls ordered correctly (lookup before act)?

## Step 4: Review skill performance

Assess skill use with evidence (skill codes where present), in light of Step 2:

1. **Requested vs loaded** — What was searched for, and what was actually loaded
   and followed? If the workflow exposes no skill tools, the agent could not
   do this. Do not treat that as a skipped lookup.
2. **Coverage** — Given the task, were the right skills found and applied, or
   was skill lookup skipped in favour of general knowledge? Only ask this when
   skill tools were available.
3. **Gaps** — Name any skill that would have improved the result. Distinguish:
   - **Behaviour gap** — skill tools were available, a suitable skill exists,
     and it should have been loaded
   - **Library gap** — the job needs a skill and none exists (do not penalise
     the agent; flag as a learning)
   - **Prompt or wiring gap** — the system prompt requires skills the job does
     not need (wrong system prompt), or the job needs skills the workflow does
     not wire (missing tool). Step 2 decides which.
4. **Depth** — Were skill cross-references followed, or did the agent stop at
   the first skill? Only when a skill was loaded.

## Step 5: Score with the rubric (0–100)

Assign points in each category below, then sum to a single **integer grade from
0 to 100**. Base every deduction on evidence from Steps 1–4. Category A uses
goal, outcome, and cost. B uses Step 3. C uses Step 4. When Step 2 shows the
agent was doing the rational thing with the context and tools it had, do not
put that deduction on B or C. The goal can still score low. Put the cause in
the rationale and in the learning.

### Scoring rubric (100 points total)

#### A. Job done efficiently (goal + cost) — 40 points

The single most important question: did the session achieve the user's /
workflow's goal, and at what cost (turns, redundant work, token burn)?

A run that Failed because the last output-token attempt was still `max_tokens`,
or because the instructions could not be fulfilled, did not finish the job.
Score the goal honestly. Name the configuration cause in the rationale. Do not
describe that failure as the model thinking too much, or as the agent ignoring
tools, unless Step 2 showed that.

| Band | Points | Criteria |
| --- | --- | --- |
| Excellent | 36–40 | Goal fully achieved; near-minimal steps; no dead ends or repeated retries; outcome correct and complete; cost proportionate. |
| Good | 28–35 | Goal achieved; a little wasted effort or cost (one or two avoidable steps) but no material impact on the result. |
| Adequate | 18–27 | Goal mostly achieved, or achieved via noticeably bloated effort / cost (several redundant steps, minor dead ends). |
| Weak | 8–17 | Goal only partially achieved, or correct outcome reached through heavy wasted effort / repeated retries / high cost. |
| Failed | 0–7 | Goal missed, wrong result delivered, or session abandoned. |

#### B. Tool use — 35 points

Efficiency, correctness, right tool for the job, and sequencing/batching
(Step 3). Score the agent against the tools it was given and the context it
still had after compaction.

| Band | Points | Criteria |
| --- | --- | --- |
| Excellent | 32–35 | Every call necessary and well-chosen; correct tool each time; independent calls batched, dependent calls ordered; no redundant repeats; errors handled sensibly. |
| Good | 25–31 | Mostly efficient; at most one redundant call or minor serial-vs-batch miss; tool choices correct. |
| Adequate | 16–24 | Some clear redundancy (the same lookup re-run while the result was still in context) or a sub-optimal tool choice, but the data still advanced the task. |
| Weak | 7–15 | Frequent redundant calls, blind retries on errors/empty results, or repeatedly wrong tool choices. |
| Poor | 0–6 | Tool use actively obstructed the task (wrong tools throughout, ignored errors, churned without progress). |

#### C. Skill use — 25 points

Whether the right skills were found (`find_available_skills`), loaded and
followed (`get_skill`), with no quality-improving gaps (Step 4).

| Band | Points | Criteria |
| --- | --- | --- |
| Excellent | 23–25 | Right skills searched for, loaded, and followed; relevant cross-references pursued; no behaviour gap. |
| Good | 18–22 | Relevant skills found and applied; at most a minor depth gap (stopped at first skill when a cross-reference would have helped). |
| Adequate | 11–17 | Some relevant skill use, but a quality-improving skill that exists was not loaded, OR skills were loaded but only partially followed. |
| Weak | 4–10 | Skill-lookup largely skipped; fell back on general knowledge where a skill clearly applied. |
| Poor / N/A handling | 0–3 | No skill lookup attempted despite the task warranting it and skill tools being available. |

> **Do not penalise the agent** — award the full 25 and record the reason —
> when any of these is true:
>
> - The task legitimately needs no skill.
> - Skill tools were not on the workflow, whether the learning is a wrong
>   system prompt or a missing tool.
> - No suitable skill exists (library gap). Flag the gap; do not score this band.
>
> Penalise C only for a **behaviour gap**: skill tools were available, a
> suitable skill exists, and the agent did not load or follow it.

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
sub-scores). Name the layer when the cause is the system prompt, the workflow
instructions, a missing tool, compaction, or the output-token ladder. Persist
the score only via `set_run_grading` in Step 7.

## Step 6: Record learnings as inbox entries

Identify discrete, actionable learnings from Steps 2–4. If the run was clean and
there is nothing to improve, create **no** entries and say so — you still must
call `set_run_grading` in Step 7.

Each learning names one cause and the change that follows from it. Do not
recommend a tool change when you concluded the system prompt is wrong, or a
prompt change when the job needs the missing tool. When Step 2 left two causes
open, file one entry that states both and what you could not separate. Do not
collapse them into a single confident recommendation.

Create **separate** `EVAL` entries when the causes are distinct (for example a
wrong system prompt and, separately, an output-token ladder).

For each learning, call `create_inbox_entry` **exactly once** with:

- `title` — short, specific one-line summary that names the layer
  (system prompt, workflow, tool, compaction, or output tokens)
- `body` — markdown covering: what was observed in the telemetry, which schema
  files you read, why that layer is the cause, and the concrete change you
  recommend (reference tool names, skill codes, the system-prompt code, and
  the workflow code). When you recommend an output-token change, describe the
  shape of the ladder (a lower first cap, then a jump, and whether to repeat
  the high cap), not a single prescribed number.
- `routing_type` — always **`EVAL`**
- `status` — always **`PENDING`**
- `source` — `WF-EVAL-RUN`
- `workflow_name` — the subject run's workflow, taken from the telemetry
- `entity_name` — the subject entity name (`{{entity.name}}`) when present
- `unit_of_work_name` — the subject run's unit of work, taken from the
  telemetry

Omit a field only when the telemetry (or entity tag) does not have it.

Capture the returned **entry reference** (8-character code) for Step 7.

## Step 7: Persist the grade

Call `set_run_grading` **exactly once** with:

- `run_reference` — the subject reference from **Subject run** above
  (`{{run.reference}}`). Paste that exact value. Do **not** use the shortened
  Guid `runId` from telemetry.
- `grading` — the integer 0–100 from Step 5
- `inbox_entry_reference` — optional; the primary learning's reference from
  Step 6 when one exists (links the traffic-light grade tag to that finding)

Re-evaluation overwrites the previous grade.

## Step 8: Reply

Reply with one or two lines: the integer grade and band, a short rationale that
names the layer when there is one, and how many inbox learnings you recorded.
Do not create duplicate entries for the same learning.
