---
name: blackboard
description: Blackboard shape framer for opportunistic multi-agent work over a shared workspace. ALWAYS invoke this skill when the user asks to coordinate multiple specialist agents that read and write to a shared scratchpad, when work emerges from agents picking up tasks matching their capability rather than from a fixed schedule, when the right decomposition isn't known upfront, or for "let several agents loose on this and see what emerges" patterns. Produces a Blackboard contract — shared workspace location and structure, agent specialists with capabilities and triggers, write-access policy, conflict resolution, termination, captured artifact. Does not execute. Do not spawn N agents on shared state ad-hoc — use this skill first to make the workspace, specialists, and termination explicit. Skip when work is cleanly parallel-independent (use shape:swarm), sequential with fixed handoffs (use shape:pipeline), or single-agent (use shape:one-shot).
# description-style: directive + negative constraint (Seleznov)
# rationale: Blackboard is the most exotic shape — without explicit framing it's just "spawn agents and hope." Directive style forces explicit workspace + specialists + termination, all of which are non-obvious decisions for this shape.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate: "Let multiple agents loose on this codebase to surface issues across security, performance, and readability"
2. SHOULD NOT activate: "Review these 5 PRs" (predetermined slices, no shared state — shape:swarm)
3. BOUNDARY: "Solve this puzzle with multiple approaches" (could be Blackboard with shared workspace where agents pick up sub-problems, could be Search with candidates — skill must clarify whether agents coordinate or compete)
-->

# blackboard

Frames opportunistic multi-agent work over a shared workspace as a Blackboard contract — workspace, specialists, write access, conflict resolution, termination, captured artifact. Distinct from Swarm (parallel-independent, no shared state) and Pipeline (sequential, fixed handoffs). One of an 11-shape family in the `shape` plugin; frame-only, never executes.

## When to Use

- Multiple agents have distinct specialist capabilities (security, performance, readability, accessibility, …) and the work can be sliced across capabilities, not items.
- Right decomposition isn't known upfront — work emerges as agents pick up what they can contribute.
- Shared workspace is a natural artifact (planning doc, scratchpad, partial-results buffer).
- Examples (coding/ops): codebase audit across specialist lenses (security, performance, accessibility, readability), multi-perspective design critique, opportunistic refactoring where each specialist applies the changes it sees. Examples (knowledge work + research): multi-disciplinary research synthesis (history + economics + sociology agents on one topic), open-ended thesis exploration where specialist agents bring framings, multi-source claim verification, comparative analysis across N angles.
- Composing with other shapes: each specialist may itself be a Pipeline; a Critic specialist; a Search specialist. Blackboard is often the meta-shape for genuinely complex multi-agent work.

## When NOT to Use

- Cleanly parallel-independent work, no shared state needed → `shape:swarm`.
- Sequential with fixed handoffs → `shape:pipeline`.
- Single agent suffices → `shape:one-shot` or another single-worker shape.
- Quality iteration on one output → `shape:critic`.
- Coordination overhead would exceed the work itself — Blackboard is heavyweight; small tasks don't earn it.
- Agents would all do approximately the same thing — that's Swarm or Search.

## Process

The skill outputs a **Blackboard contract** — a structured document, not executed work.

### 1. Confirm shape fits

Blackboard fits if:
- ≥3 distinct specialist capabilities are needed.
- Work emerges from specialists picking up what they can contribute (not a fixed pre-plan).
- Shared state is genuinely shared (not just an output bucket).

If specialists could just work independently with no read-from-shared-state, it's Swarm, not Blackboard.

### 2. Define the shared workspace

**Load-bearing.** The blackboard IS the coordination mechanism. Spec:

- **Location** — file path, conversation, named artifact. Concrete.
- **Structure** — sections by topic, by specialist, by item, or free-form?
- **Format** — markdown, JSON, append-only log?
- **Read access** — all specialists read freely.
- **Write access** — append-only? section-owned? conflict-resolved (see step 4)?
- **Initial state** — what's on the blackboard when specialists start?

### 3. Define specialists

For each specialist:

- **Name** — what role it plays (security-reviewer, perf-optimizer, accessibility-auditor, …).
- **Worker** — Claude sub-Agent **(model: opus)** with role-specific prompt, or Codex with role framing, or a specific subagent_type if one matches.
- **Capability** — what it can contribute (what kinds of changes it can write to the blackboard).
- **Trigger** — when does it pick up work? On blackboard change? Periodically? On a specific signal?
- **Output to blackboard** — what kind of entries does it produce (findings, edits, questions, claims)?

3-7 specialists is the typical range. Fewer → just use Swarm. More → coordination overhead dominates.

### 4. Define write-access and conflict resolution

If multiple specialists can write to the same section, conflicts happen. Resolution options:

- **Append-only log** — no conflicts; all writes preserved with attribution.
- **Section ownership** — each specialist owns specific sections; no cross-writes.
- **Last-write-wins** — risky; only OK for additive findings.
- **Vote / merge** — a coordinator (Claude or a dedicated agent) resolves conflicts.

State the policy. Append-only is safest default.

### 5. Define termination

When does the blackboard work stop?

- **Convergence** — N rounds with no new specialist contributions.
- **Coverage** — every specialist has had at least one contribution, AND each has explicitly "passed" on further work.
- **Budget** — N total writes / M minutes.
- **External signal** — user calls "done" / a triggered event marks it complete.

Blackboard sprawls without explicit termination.

### 6. Define captured artifact

The blackboard itself is often the output, but it may need synthesis:
- **Raw blackboard** — hand the workspace itself to the user.
- **Synthesis pass** — a final agent (Claude or Codex) reads the blackboard and produces a structured report.
- **Per-specialist summaries** — each specialist writes a closing summary, collected.

State which.

### 7. Output the contract

Write to `<contracts-root>/blackboard-<slug>.md`:

```markdown
# Blackboard contract: <task-name>

## Workspace
- Location: <path>
- Structure: <sections by X | free-form | …>
- Format: <markdown | json | append-log>
- Initial state: <empty | seeded with Y>

## Specialists
| Name | Worker | Capability | Trigger | Output type |
|---|---|---|---|---|
| <security-reviewer> | subagent:security-review (model: opus) | <flags vulns> | <on file change in blackboard> | <findings.json entries> |
| … |

## Write access
- Policy: <append-only | section-owned | vote/merge | last-write-wins>
- Conflict resolution: <how>

## Termination
- Rule: <convergence | coverage | budget | external-signal>
- Cap: <N rounds | T minutes | M writes>

## Captured artifact
- Form: <raw blackboard | synthesis pass | per-specialist summaries>
- Synthesizer (if applicable): <claude | codex | subagent:<type> (model: opus)>
- Output path: <path>

## Compositions
- <none | each specialist is shape:pipeline | synthesis is shape:critic | …>
```

### 8. Stop

The skill is framing-only. Do not initialize the blackboard. Do not spawn specialists. The orchestrator follows.

## Anti-Patterns

- **Blackboard when Swarm would suffice.** Shared state for no reason is overhead. If specialists don't need to read what others wrote, use Swarm.
- **Too many specialists.** Coordination overhead dominates beyond ~7. Trim to the load-bearing roles.
- **Loose write access producing chaos.** Multiple specialists overwriting each other = lost work + confusion. Default to append-only.
- **No termination.** Blackboard work sprawls. Always cap.
- **Workspace structure invented per-fire.** Specialists need to know where to read/write *before* starting. Structure is part of the contract, not improvised at runtime.
- **Specialist prompts that don't reference the blackboard.** If the specialist's prompt doesn't tell it "read the blackboard at path X, append your findings to section Y," the shape isn't actually Blackboard — it's just N parallel agents producing isolated outputs (Swarm).
- **Hidden coordination through the orchestrator.** If the orchestrator is routing messages between specialists (specialist A finishes → orchestrator hands result to specialist B), that's Pipeline, not Blackboard. Blackboard agents pick up work autonomously by reading shared state.

## Rules

- The skill always outputs a single Markdown contract file; it never initializes or runs the blackboard.
- Every Blackboard contract must declare: workspace location/structure/format/initial-state, ≥3 specialists with worker/capability/trigger/output, write-access policy, termination, captured artifact.
- Default write-access policy is append-only.
- Specialist count cap is ~7; beyond that, restructure.
- When a specialist is a Claude sub-Agent, the contract MUST specify `model: opus`. Never Sonnet, never Haiku.
- Each specialist prompt must reference the blackboard explicitly (path + read/write sections); otherwise the shape isn't actually Blackboard.
- If the task doesn't fit Blackboard, recommend the right shape and stop — don't force-fit. Check `<available_skills>` before suggesting by name; if not installed, describe inline.
- Contract path: `<contracts-root>/blackboard-<slug>.md`. Slug is kebab-case from the task name. `<contracts-root>` resolution (in order): user-specified path > task-implied project folder > cwd if it is a project (has `.git/` or `CLAUDE.md`) → `<project>/.claude/contracts/` > fallback `/tmp/shape-contracts/`.

## Key Files

- Output: `<contracts-root>/blackboard-<slug>.md` — the contract document.
- Sibling shape skills (planned, all under the `shape` plugin namespace): `shape:pipeline` (✓ live), `shape:swarm` (✓ live), `shape:critic` (✓ live), `shape:gated` (✓ live), `shape:event` (✓ live), `shape:one-shot`, `shape:search`, `shape:dialogue`, `shape:loop`. Related external skills: `shape:contract`, `/loop` (Loop scheduling). Blackboard is the most exotic shape — use sparingly, restructure to Swarm/Pipeline when possible.
