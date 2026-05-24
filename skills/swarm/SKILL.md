---
name: swarm
description: Swarm shape framer for parallel-independent multi-slice tasks with explicit aggregation. ALWAYS invoke this skill when the user asks to fan-out, parallelize, batch, run-in-parallel, or process N independent items simultaneously — review N PRs, fetch N URLs, summarize N documents, generate N variants, run N tests, score N candidates, or any task where slices have no order dependency. Produces a Swarm contract — N slices with per-slice worker, the reducer that merges N outputs into one, the failure mode (all-or-nothing / partial-OK / quorum), and the parallelism cap. Does not execute the swarm; the orchestrator follows it. Do not fan out work ad-hoc — use this skill first to make slices and the reducer explicit. Skip for sequential-handoff work (use shape:pipeline), single-shot tasks, exploration without a defined slice set (use shape:search), or work needing inter-slice feedback (use shape:blackboard).
# description-style: directive + negative constraint (Seleznov)
# rationale: Swarm framing competes with default Claude behavior (would otherwise fan out ad-hoc, often without a reducer); directive style forces the explicit framing step including the load-bearing reducer.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate: "Review these 5 PRs in parallel and give me a security summary"
2. SHOULD NOT activate: "Refactor the auth module step by step" (sequential — Pipeline)
3. BOUNDARY: "Run the linter on all files and fix the errors" (lint is parallel-independent, but fixes can conflict — Swarm for lint + Pipeline for fix? skill should call this out and recommend composition)
-->

# swarm

Frames a parallel-independent multi-slice task as a Swarm contract — N slices, per-slice worker, reducer that merges N outputs into one, failure mode, parallelism cap. One of an 11-shape family in the `shape` plugin; frame-only, never executes.

## When to Use

- Task partitions cleanly into N independent slices with no order or handoff dependency.
- Total time is bounded by the slowest slice, not the sum.
- A reducer (merge / dedupe / vote / pick-best / summarize) maps N outputs to one final output.
- Examples (coding/ops): review N PRs, fetch N URLs, run N test files in parallel, score N candidates, generate N code variants. Examples (knowledge work): summarize N documents, extract claims from N transcripts, analyze N research papers, gather quotes on topic X from N sources, tag N inbox items by category.
- Composing with other shapes: a Pipeline stage that is itself a Swarm, or a Critic where multiple critics vote.

## When NOT to Use

- Slices depend on each other's outputs (handoff required) → use `shape:pipeline`.
- The work is conceptually one thing, not partitionable → just do it (one-shot).
- Aggregation IS the work, slice processing is trivial → use `shape:search`.
- Workers need to coordinate or react to each other's intermediate state → use `shape:blackboard`.
- N is unknown — task is "explore until done" rather than "process this specific set" → use `shape:search`.

## Process

The skill outputs a **Swarm contract** — a structured document, not executed work. Frame, write, hand back.

### 1. Confirm the shape fits

Independence test: would the same final answer come back if slices ran in any order, or none at all? If no → not Swarm. If yes → Swarm fits.

Reducer test: can you state how N outputs become one in a single sentence? If no → the work isn't ready to frame as Swarm; either the reducer needs more design or the shape is actually Search.

### 2. Partition into slices

Enumerate the N concrete inputs. Each slice is one input. Slices should be:

- **Independent** — no shared mutable state.
- **Roughly comparable in cost** — wildly uneven slices waste parallelism (the swarm waits on the slowest).
- **Self-contained per worker** — each worker starts fresh; the per-slice prompt must include enough context to do the slice without seeing the others.

If N is large, consider whether to batch slices (e.g., 100 items in 10 batches of 10). The parallelism cap (step 5) usually forces batching.

### 3. Pick worker per slice

Default by slice characteristic:

| Slice shape | Worker |
|---|---|
| Long single-task, deterministic (e.g., refactor one file, analyze one log) | Codex (`codex exec`) |
| Multi-tool exploration of one input (e.g., review one PR with multiple lookups) | sub-Agent (model: opus) |
| Pure read + simple transform (e.g., fetch and parse one URL) | Claude inline or sub-Agent (model: opus) |
| Heterogeneous slices (different per slice) | mix workers per slice; declare each |

Uniform worker across all slices is the common case. Mixed assignment is valid but must be declared per-slice. **Claude sub-Agents always use Opus — never Sonnet or Haiku.**

### 4. Define the reducer

**Load-bearing**. A Swarm without a reducer is fan-out with no convergence — abandoned parallel work. The reducer answers: how do N outputs become one?

Common reducer patterns:

- **Concatenate** — append all outputs in order.
- **Merge + dedupe** — combine, drop duplicates by key.
- **Vote** — majority / unanimous / quorum.
- **Pick-best** — score each, return top.
- **Summarize** — feed N outputs to one synthesis pass (Claude or Codex).
- **Group + structure** — bucket by category, output table or report.

Declare the reducer explicitly: pattern + concrete merge logic + output shape.

### 5. Define failure mode and parallelism cap

**Failure mode**:
- `all-or-nothing` — any slice failure halts; final output requires all N.
- `partial-OK` — tolerate ≤k failures, report them in the final output.
- `quorum` — proceed once ≥m slices succeed.

Pick based on the work. Per-PR review tolerates one PR being unreachable (partial-OK). Per-file refactor that must compile usually demands all-or-nothing.

**Parallelism cap**: per anti-pattern #19 in the skill catalog, cap at ~8 concurrent workers. Coordination overhead exceeds parallelism gain beyond that. For N > 8, batch.

### 6. Output the contract

Write to `<contracts-root>/swarm-<slug>.md` in this shape:

```markdown
# Swarm contract: <task-name>

## Slices
- N: <count>
- Worker (default): <claude | codex | subagent:<type>>
- Parallelism cap: <number, usually ≤8>

| # | Input | Worker (override) | Expected output |
|---|---|---|---|
| 1 | <input 1> | <— or override> | <shape> |
| 2 | <input 2> | <— or override> | <shape> |
| … |

## Reducer
- Pattern: <concatenate | merge+dedupe | vote | pick-best | summarize | group>
- Logic: <one-paragraph spec of the merge>
- Output shape: <file path or in-line shape>

## Failure mode
- <all-or-nothing | partial-OK (≤k) | quorum (≥m)>
- On failure: <action — include in output as "failed: …" / halt / retry slice>

## Compositions
- <none | slice is itself shape:critic | reducer is shape:pipeline | …>
```

Then surface the path to the orchestrator.

### 7. Stop

The skill is framing-only. Do not start spawning Agents. Do not call `codex exec`. The orchestrator decides when and whether to run.

## Anti-Patterns

- **Fan-out without a reducer.** The skill's load-bearing constraint. N parallel results with no merge step is abandoned work. If the reducer isn't designable, the task isn't a Swarm — likely Search or just one-shot.
- **Forcing parallel where there's a dependency.** If slice B's input requires slice A's output, this is Pipeline, not Swarm. Test: can slices run in any order? If no, wrong shape.
- **Too many slices (>8 concurrent).** Per anti-pattern #19, coordination overhead eats the parallelism gain. Batch into ≤8 concurrent workers.
- **Per-slice prompts that omit context.** Each worker starts fresh — no shared memory of the overall task. The per-slice prompt must include the framing the worker needs to do the slice well in isolation.
- **Heterogeneous slices that should be a different shape.** If slices need to react to each other → Blackboard. If slices include a "judge" — Critic. If one slice's failure should branch the rest — Pipeline+Gated. Don't shoehorn into Swarm.
- **Uneven slice cost.** Wildly uneven slices waste the parallelism — the whole swarm waits on the slowest. Re-partition or re-batch to balance.
- **Skill executes the swarm.** Frame-only. Executing couples framing to runtime; output the contract and stop.

## Rules

- The skill always outputs a single Markdown contract file; it never executes the swarm.
- Every Swarm contract must declare: N, per-slice worker(s), reducer, failure mode, parallelism cap.
- The reducer is non-optional. A Swarm without a reducer is rejected.
- When a slice worker is a Claude sub-Agent, the contract MUST specify `model: opus`. Never Sonnet, never Haiku.
- Parallelism cap defaults to ≤8 (per anti-pattern #19); larger N batches.
- If the task doesn't fit Swarm, recommend the right shape and stop — don't force-fit. When recommending a sibling shape, check `<available_skills>` for the sibling skill before suggesting by skill name; if not installed, describe the shape inline.
- Contract path: `<contracts-root>/swarm-<slug>.md`. Slug is kebab-case from the task name. `<contracts-root>` resolution (in order): user-specified path > task-implied project folder > cwd if it is a project (has `.git/` or `CLAUDE.md`) → `<project>/.claude/contracts/` > fallback `/tmp/shape-contracts/`.

## Key Files

- Output: `<contracts-root>/swarm-<slug>.md` — the contract document.
- Sibling shape skills (planned, all under the `shape` plugin namespace): `shape:pipeline`, `shape:critic`, `shape:gated`, `shape:event`, `shape:blackboard`, `shape:search`, `shape:dialogue`, `shape:one-shot`, `shape:loop`. Related external skills: `shape:contract`, `/loop` (Loop scheduling).
