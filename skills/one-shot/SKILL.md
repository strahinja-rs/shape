---
name: one-shot
description: One-shot shape framer for tasks that need no decomposition — single action, single worker, single completion criterion, no iteration. ALWAYS invoke this skill when the orchestrator (or user) has considered other shapes and concluded none apply — single tool call, single sub-Agent invocation, single Codex run, "just do the thing" requests. Produces a One-shot contract — the action, the worker, the input, the success criterion. Acts as the explicit "no shape needed" declaration so the orchestrator's choice is auditable rather than implicit. Skip for anything with multiple stages (use shape:pipeline), parallel slices (use shape:swarm), quality iteration (use shape:critic), human gates (use shape:gated), or trigger-fired work (use shape:event). Do not skip framing entirely on small tasks — use this skill to declare one-shot explicitly so the decision is on record.
# description-style: directive + negative constraint (Seleznov)
# rationale: One-shot is the degenerate shape — its value is being explicit about "I checked, no decomposition needed." Without this skill, the orchestrator silently defaults to one-shot for everything, including tasks that should have been decomposed.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate: "Read the README and tell me what this project does" (single action, single deliverable)
2. SHOULD NOT activate: "Refactor the auth module and make sure tests pass" (multi-stage — shape:pipeline)
3. BOUNDARY: "Fix this bug" (could be one-shot if it's a one-line fix, could be Pipeline if it's a multi-file refactor with tests — skill should clarify scope before framing)
-->

# one-shot

Frames a task that needs no decomposition as a One-shot contract — the action, the worker, the input, the success criterion. The degenerate shape; its value is explicit "no decomposition needed" so the orchestrator's choice is auditable. One of an 11-shape family in the `shape` plugin; frame-only, never executes.

## When to Use

- Task fits in a single action — one tool call, one sub-Agent invocation, one Codex run, one prompt.
- No multi-step dependency, no parallel slices, no quality iteration, no human gate, no trigger.
- Success criterion is simple (single check, single artifact).
- The orchestrator wants to mark "I considered shapes and concluded one-shot" rather than defaulting silently.
- Examples (coding/ops): fix a one-line typo, generate a single config snippet, format some text, run a single command. Examples (knowledge work): read and summarize a file, summarize one paper, answer a factual question, define a term, draft one paragraph, extract one claim from a source.

## When NOT to Use

- Multi-stage with handoffs → `shape:pipeline`.
- Parallel-independent slices → `shape:swarm`.
- Quality-critical with iteration → `shape:critic`.
- Irreversible action needing human approval → `shape:gated`.
- Trigger-fired work → `shape:event`.
- Open-ended objective with "make sure it actually works" framing → `shape:contract`.
- Recurring → `/loop` or `/schedule`.

If any of those apply, use that shape instead and skip One-shot.

## Process

The skill outputs a **One-shot contract** — a minimal structured document, not executed work.

### 1. Confirm shape fits

Run the "would another shape do better" check explicitly:
- Multiple stages with handoffs? → Pipeline.
- N independent slices? → Swarm.
- Quality is judgment-bound and iteration helps? → Critic.
- Irreversible? → Gated (compose with whatever else fits).
- Trigger-fired? → Event.

If all answers are no, One-shot fits.

### 2. Pick the worker

- Claude inline (you) — for simple read + write, single tool call.
- Codex (`codex exec`) — for long-context single-task work that benefits from Codex's depth (large refactor, deep analysis of one input).
- sub-Agent **(model: opus)** — for an isolated context that doesn't pollute the main conversation.

### 3. Output the contract

Write to `/tmp/one-shot-<slug>.md`:

```markdown
# One-shot contract: <task-name>

## Action
<one-sentence description of the single action>

## Worker
<claude | codex | subagent:<type> (model: opus)>

## Input
<the input the worker needs>

## Success criterion
<single concrete check — file exists, answer given, output produced>

## Why not another shape
<one line explaining what was ruled out, e.g., "no handoffs (not Pipeline), no slices (not Swarm), no judgment loop (not Critic), reversible local action (not Gated)">
```

### 4. Stop

The skill is framing-only. Do not execute. The orchestrator (or user) acts.

## Anti-Patterns

- **Defaulting to One-shot on tasks that should be decomposed.** The whole point of this skill is to FORCE the "would another shape do better" check. Skipping that check defeats the purpose.
- **Adding ceremony to trivial work.** If the user said "what's 2+2" you shouldn't write a One-shot contract. This skill is for tasks worth marking as a deliberate choice; not for trivia.
- **One-shot for irreversible actions.** Even single actions that are irreversible (delete file, send message, deploy) want `shape:gated` composition, not bare One-shot.
- **Missing "why not another shape" rationale.** The line is small but load-bearing — it's the audit trail that justifies the One-shot decision.

## Rules

- The skill always outputs a single Markdown contract file; it never executes the action.
- Every One-shot contract must declare: action, worker, input, success criterion, why-not-another-shape rationale.
- If the work is so trivial that writing the contract takes longer than doing the work, skip the contract entirely — One-shot is for *deliberate* one-shots, not for everything small.
- When the worker is a Claude sub-Agent, the contract MUST specify `model: opus`. Never Sonnet, never Haiku.
- If any other shape fits, recommend it and stop — don't force-fit. When recommending a sibling shape, check `<available_skills>` for the sibling skill before suggesting by name; if not installed, describe the shape inline.
- Contract path always `/tmp/one-shot-<slug>.md`.

## Key Files

- Output: `/tmp/one-shot-<slug>.md` — the contract document.
- Sibling shape skills (planned, all under the `shape` plugin namespace): `shape:pipeline` (✓ live), `shape:swarm` (✓ live), `shape:critic` (✓ live), `shape:gated` (✓ live), `shape:event` (✓ live), `shape:blackboard`, `shape:search`, `shape:dialogue`, `shape:loop`. Related external skills: `shape:contract`, `/loop` (Loop scheduling).
