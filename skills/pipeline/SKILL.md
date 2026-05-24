---
name: pipeline
description: Pipeline shape framer for multi-stage tasks with fixed sequential ordering. ALWAYS invoke this skill when the user asks to plan, structure, scaffold, or break down a workflow with distinct phases — refactor + test + document, gather + transform + load, build + deploy + verify, audit + fix + ship, or any task naming ≥3 ordered steps. Produces a Pipeline contract document — stages, per-stage worker (Claude / Codex / sub-Agent), inputs/outputs, termination check, on-failure behavior. Does not execute the pipeline; the orchestrator follows it. Do not structure multi-stage work ad-hoc — use this skill first to make stages and handoffs explicit. Skip for single-shot tasks, open-ended objectives with one completion criterion (use task-to-verifiable-loop), parallel-independent work (use shape:swarm), or event-triggered work (use shape:event).
# description-style: directive + negative constraint (Seleznov)
# rationale: Pipeline framing competes with default Claude behavior (would otherwise structure multi-stage work ad-hoc); directive style forces the explicit framing step.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate: "Help me plan a multi-step refactor: audit, map, apply, test, document the migration"
2. SHOULD NOT activate: "Run the test suite and tell me what failed" (one-shot, not a pipeline)
3. BOUNDARY: "I need to migrate the database schema, make sure to verify each step" (could be Pipeline or Contract or composition; description's WHEN NOT clauses must disambiguate)
-->

# pipeline

Frames a multi-stage task as a Pipeline contract — stages, workers, handoffs, termination, on-failure — for the orchestrator to execute. One of an 11-shape family in the `shape` plugin; frame-only, never executes.

## When to Use

- Task has ≥3 distinct phases with fixed ordering (refactor + test + document; gather + transform + load; build + deploy + verify).
- Stages have meaningful handoffs — output of stage A is input to stage B.
- Different stages call for different workers (some Claude, some Codex, some sub-Agent).
- User wants implicit step-by-step work made explicit before execution.
- Composing with other shapes: a Pipeline whose one stage is itself a Critic, Gated, or Swarm.

## When NOT to Use

- Single-shot tasks ("run the tests", "rename this variable") — just do them.
- Open-ended objectives with one completion criterion ("make X work") — use `task-to-verifiable-loop` (Contract shape).
- Parallel-independent work with no fixed ordering — use `shape:swarm`.
- Triggered-by-event work ("when X happens, do Y") — use `shape:event`.
- Conversational exploration where stages aren't known yet — talk first, frame later.

## Process

The skill outputs a **Pipeline contract** — a structured document, not executed work. Frame, write, hand back.

### 1. Confirm the shape fits

Before framing, verify Pipeline is right:
- Is the ordering fixed? If reorderable → recommend `shape:swarm`.
- Is there only one end-state criterion? → recommend `task-to-verifiable-loop`.
- Are stages truly independent? → recommend `shape:swarm`.

If unsure, ask the user one clarifying question. Don't force-fit.

### 2. Derive stages from the concrete task

Do not invent generic stages (Plan / Execute / Verify). Derive from the actual task. Each stage needs all five fields:

- **Name** — verb phrase, specific to the work.
- **Worker** — `claude` / `codex` / `subagent:<type>` / `human`.
- **Input** — artifact or state consumed.
- **Output** — artifact or state produced.
- **Termination** — concrete check (file exists, tests pass, list complete, criterion met).
- **On failure** — `escalate` / `branch to <stage>` / `halt + report` / `retry with <change>`.

A stage missing any field is a plan, not a contract. Fill it or drop it.

### 3. Pick workers per stage intentionally

Default by stage characteristic:

| Stage shape | Worker |
|---|---|
| Heavy single-task, deterministic, long context | Codex (`codex exec`) |
| Multi-tool exploration, many small reads/writes | Claude |
| Critique / adversarial review | sub-Agent (or Codex `/adversarial-review`) |
| Multi-file mechanical edits with clear mapping | Codex |
| User confirmation or input | human |
| Quick lookups, simple Bash, single file edits | Claude |

Do not default every stage to Claude. Do not reach for Codex on small stages. Worker choice is a per-stage decision.

### 4. Define cross-stage state

After per-stage fields, add:

- **Shared state** — where intermediate artifacts live (file path, conversation, named artifact).
- **Top-level termination** — which stages must reach which state for the whole pipeline to be done.
- **Top-level on-failure** — `halt and report` / `continue with partial` / `roll back via <action>`.
- **Compositions** — if any stage is itself a Critic, Gated, or Swarm, name it explicitly. Hidden composition makes the contract lie.

### 5. Output the contract

Write to `/tmp/pipeline-<slug>.md` in this shape:

```markdown
# Pipeline contract: <task-name>

## Stages

1. **<verb-phrase stage name>**
   - Worker: <claude | codex | subagent:<type> | human>
   - Input: <artifact / state>
   - Output: <artifact / state>
   - Termination: <concrete check>
   - On failure: <action>

[repeat for each stage]

## Cross-stage

- Shared state: <location>
- Top-level termination: <criteria>
- Top-level on-failure: <action>
- Compositions: <none | stage N is shape:critic | …>
```

Then surface the path to the orchestrator.

### 6. Stop

The skill is framing-only. Do not start executing stages. Do not spawn Agents. Do not call `codex exec`. The orchestrator (the user, the future router skill, or Claude in a follow-up turn) decides when and whether to run.

## Anti-Patterns

- **Forcing Pipeline on parallel-independent work.** If stages don't consume each other's outputs, this is Swarm. Pipeline imposes serialization for no reason.
- **Stages without termination or on-failure.** A stage with "do X" but no concrete completion check produces a plan, not a contract. Each stage must answer "how do I know it's done?" and "what if it isn't?"
- **All stages assigned to Claude.** Wastes Codex's strengths on long deterministic stages. Audit each stage's worker explicitly.
- **Generic stage names (Plan / Execute / Verify).** Symptom of skipping step 2 — derive stages from the concrete task, not a template.
- **Skill executes the pipeline.** The skill is frame-only. Executing couples framing to runtime decisions and breaks the orchestrator separation. Output the contract and stop.
- **Composition opacity.** If a stage is itself a Critic or Swarm, name it explicitly. A contract that hides nested shapes lies about the work.

## Rules

- The skill always outputs a single Markdown contract file; it never executes stages.
- Every stage must have all five fields: worker, input, output, termination, on-failure.
- Worker assignment is intentional per-stage, never defaulted to Claude across the board.
- If the user's task does not fit Pipeline, recommend the right shape and stop — do not force-fit. When recommending a sibling shape, check whether its skill is installed in `<available_skills>` before suggesting by skill name. If not installed, describe the shape inline (e.g., "this is parallel-independent work — Swarm shape; frame manually as N concurrent slices + merge step") rather than name a skill that doesn't exist.
- Contract path always `/tmp/pipeline-<slug>.md`; slug is kebab-case derived from the task name.

## Key Files

- Output: `/tmp/pipeline-<slug>.md` — the contract document.
- Sibling shape skills (planned, all under the `shape` plugin namespace): `shape:swarm`, `shape:critic`, `shape:gated`, `shape:event`, `shape:blackboard`, `shape:search`, `shape:dialogue`, `shape:one-shot`, `shape:loop`. Related external skills: `task-to-verifiable-loop` (Contract shape), `/loop` (Loop scheduling).
