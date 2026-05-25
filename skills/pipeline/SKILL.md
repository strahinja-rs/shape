---
name: pipeline
description: Pipeline approach framer for multi-stage tasks with fixed sequential ordering. ALWAYS invoke this skill when the user asks to plan, structure, scaffold, or break down a workflow with distinct phases — refactor + test + document, gather + transform + load, build + deploy + verify, audit + fix + ship, research + synthesize + write, ingest + tag + report, or any task naming ≥3 ordered steps. Produces a Pipeline contract document — stages, per-stage worker (Claude / Codex / sub-Agent), inputs/outputs, termination check, on-failure behavior. Does not execute the pipeline; the orchestrator follows it. Do not structure multi-stage work ad-hoc — use this skill first to make stages and handoffs explicit. Skip for single-shot tasks, open-ended objectives with one completion criterion (use approach:contract), parallel-independent work (use approach:swarm), or event-triggered work (use approach:event).
# description-style: directive + negative constraint (Seleznov)
# rationale: Pipeline framing competes with default Claude behavior (would otherwise structure multi-stage work ad-hoc); directive style forces the explicit framing step.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (coding): "Help me plan a multi-step refactor: audit, map, apply, test, document the migration"
2. SHOULD activate (knowledge work): "Walk me through structuring a research synthesis: gather sources, extract claims, cross-reference, write the summary"
3. SHOULD NOT activate: "Run the test suite and tell me what failed" (one-shot, not a pipeline)
4. BOUNDARY: "I need to migrate the database schema, make sure to verify each step" (could be Pipeline or Contract or composition; description's WHEN NOT clauses must disambiguate)
-->

# pipeline

Frames a multi-stage task as a Pipeline contract — stages, workers, handoffs, termination, on-failure — for the orchestrator to execute. One of the 11 named approaches in the `approach` plugin; frame-only, never executes.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- Task has ≥3 distinct phases with fixed ordering.
- Stages have meaningful handoffs — output of stage A is input to stage B.
- Different stages call for different workers (some Claude, some Codex, some sub-Agent).
- User wants implicit step-by-step work made explicit before execution.
- Composing with other approaches: a Pipeline whose one stage is itself a Critic, Gated, or Swarm.

**Examples — knowledge work:**
- Research → synthesize → write → review (paper / report / strategy doc)
- Outline → draft → edit → publish (longform writing)
- Read papers → extract claims → cross-reference → produce summary
- Discover → frame → diagnose → recommend (consulting deliverable)
- Ingest transcripts → extract themes → cluster → write narrative

**Examples — code / ops:**
- Refactor + test + document
- Gather + transform + load
- Build + deploy + verify
- Audit + fix + ship
- Spec + scaffold + implement + test

## When NOT to Use

- Single-shot tasks ("run the tests", "rename this variable") — just do them.
- Open-ended objectives with one completion criterion ("make X work") — use `approach:contract`.
- Parallel-independent work with no fixed ordering — use `approach:swarm`.
- Triggered-by-event work ("when X happens, do Y") — use `approach:event`.
- Conversational exploration where stages aren't known yet — talk first, frame later (use `approach:dialogue`).

## Process

The skill outputs a **Pipeline contract** — a structured document, not executed work. Frame, write, hand back.

### 1. Confirm the shape fits

Before framing, verify Pipeline is right:
- Is the ordering fixed? If reorderable → recommend `approach:swarm`.
- Is there only one end-state criterion? → recommend `approach:contract`.
- Are stages truly independent? → recommend `approach:swarm`.

If unsure, ask the user one clarifying question. Don't force-fit.

### 2. Derive stages from the concrete task

Do not invent generic stages (Plan / Execute / Verify). Derive from the actual task. Each stage needs all five fields:

- **Name** — verb phrase, specific to the work.
- **Worker** — `claude` / `codex` / `subagent:<type> (model: opus)` / `human`. Claude sub-Agents always use Opus per [PRINCIPLES.md §2](../../PRINCIPLES.md#2-sub-agent-workers-always-use-opus).
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

### 4. Apply the Direct-Use Rule

If any stage can exercise the real intended surface (deployed endpoint, installed MCP, real browser, real client, real document), make that part of the stage as early as possible. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible). For a research pipeline, the analog is exercising the real corpus / real sources as early as possible rather than working from summaries.

### 5. Define cross-stage state

After per-stage fields, add:

- **Shared state** — where intermediate artifacts live (file path, conversation, named artifact).
- **Top-level termination** — which stages must reach which state for the whole pipeline to be done.
- **Top-level on-failure** — `halt and report` / `continue with partial` / `roll back via <action>`.
- **Compositions** — if any stage is itself a Critic, Gated, or Swarm, name it explicitly. Hidden composition is forbidden per [PRINCIPLES.md §6](../../PRINCIPLES.md#6-composition-is-explicit--no-hidden-nesting).
- **Assumptions** — anything the framer inferred rather than the user stated (scope, expected input shape, worker capability). See [PRINCIPLES.md §3](../../PRINCIPLES.md#3-assumptions--extraction-confidence--surface-uncertainty-dont-hide-it).

### 6. Output the contract

Write to `<contracts-root>/pipeline-<slug>.md` in this shape (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)):

```markdown
# Pipeline contract: <task-name>

## Stages

1. **<verb-phrase stage name>**
   - Worker: <claude | codex | subagent:<type> (model: opus) | human>
   - Input: <artifact / state>
   - Output: <artifact / state>
   - Termination: <concrete check>
   - On failure: <action>

[repeat for each stage]

## Cross-stage

- Shared state: <location>
- Top-level termination: <criteria>
- Top-level on-failure: <action>
- Compositions: <none | stage N is approach:critic | …>
- Assumptions: <inferences during framing, or "none">
- Extraction-confidence: <low | medium | high>
```

Then surface the path to the orchestrator.

### 7. Stop

The skill is framing-only. Do not start executing stages. Do not spawn Agents. Do not call `codex exec`. The orchestrator (the user, the future router skill, or Claude in a follow-up turn) decides when and whether to run.

## Anti-Patterns

- **Forcing Pipeline on parallel-independent work.** If stages don't consume each other's outputs, this is Swarm. Pipeline imposes serialization for no reason.
- **Stages without termination or on-failure.** A stage with "do X" but no concrete completion check produces a plan, not a contract. Each stage must answer "how do I know it's done?" and "what if it isn't?"
- **All stages assigned to Claude.** Wastes Codex's strengths on long deterministic stages. Audit each stage's worker explicitly.
- **Generic stage names (Plan / Execute / Verify).** Symptom of skipping step 2 — derive stages from the concrete task, not a template.
- **Skill executes the pipeline.** The skill is frame-only per [PRINCIPLES.md §1](../../PRINCIPLES.md#1-frame-onlynever-executes). Executing couples framing to runtime decisions and breaks the orchestrator separation. Output the contract and stop.
- **Composition opacity.** If a stage is itself a Critic or Swarm, name it explicitly. A contract that hides nested approaches lies about the work.
- **Mocks-only verification when the real surface is available.** If stage X could exercise the real artifact / endpoint / system and instead only checks a mock, the Direct-Use Rule was missed.

## Rules

- The skill always outputs a single Markdown contract file; it never executes stages.
- Every stage must have all five fields: worker, input, output, termination, on-failure.
- Worker assignment is intentional per-stage, never defaulted to Claude across the board.
- When a stage worker is a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- Every contract must include `Assumptions:` and `Extraction-confidence:` fields per PRINCIPLES.md §3.
- If the user's task does not fit Pipeline, recommend the right approach and stop — do not force-fit. When recommending a sibling, check `<available_skills>` first; if not installed, describe inline.
- Contract path: `<contracts-root>/pipeline-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/pipeline-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:event`, `approach:dialogue`, `approach:search`, `approach:blackboard`. Related external skills: `/loop` (Loop scheduling executor).
