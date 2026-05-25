---
name: blackboard
description: Blackboard approach framer for opportunistic multi-agent work over a shared workspace. ALWAYS invoke this skill when the user asks to coordinate multiple specialist agents that read and write to a shared scratchpad, when work emerges from agents picking up tasks matching their capability rather than from a fixed schedule, when the right decomposition isn't known upfront, or for "let several agents loose on this and see what emerges" patterns. Produces a Blackboard contract — shared workspace location and structure, agent specialists with capabilities and triggers, write-access policy, conflict resolution, termination, captured artifact. Does not execute. Do not spawn N agents on shared state ad-hoc — use this skill first to make the workspace, specialists, and termination explicit. Skip when work is cleanly parallel-independent (use approach:swarm), sequential with fixed handoffs (use approach:pipeline), or single-agent (use approach:one-shot).
# description-style: directive + negative constraint (Seleznov)
# rationale: Blackboard is the most exotic approach — without explicit framing it's just "spawn agents and hope." Directive style forces explicit workspace + specialists + termination, all of which are non-obvious decisions for this approach.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (knowledge work): "Multi-disciplinary synthesis: history + economics + sociology agents looking at one topic, find what each surfaces"
2. SHOULD activate (coding): "Let multiple agents loose on this codebase to surface issues across security, performance, and readability"
3. SHOULD NOT activate: "Review these 5 PRs" (predetermined slices, no shared state — approach:swarm)
4. BOUNDARY: "Solve this puzzle with multiple approaches" (could be Blackboard with shared workspace where agents pick up sub-problems, could be Search with candidates — skill must clarify whether agents coordinate or compete)
-->

# blackboard

Frames opportunistic multi-agent work over a shared workspace as a Blackboard contract — workspace, specialists, write access, conflict resolution, termination, captured artifact. Distinct from Swarm (parallel-independent, no shared state) and Pipeline (sequential, fixed handoffs). One of the 11 named approaches in the `approach` plugin; frame-only, never executes.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- Multiple agents have distinct specialist capabilities (security, performance, readability, accessibility, history, economics, sociology, …) and the work can be sliced across capabilities, not items.
- Right decomposition isn't known upfront — work emerges as agents pick up what they can contribute.
- Shared workspace is a natural artifact (planning doc, scratchpad, partial-results buffer).
- Composing with other approaches: each specialist may itself be a Pipeline; a Critic specialist; a Search specialist. Blackboard is often the meta-approach for genuinely complex multi-agent work.

**Examples — knowledge work + research:**
- Multi-disciplinary research synthesis (history + economics + sociology agents on one topic)
- Open-ended thesis exploration where specialist agents bring framings
- Multi-source claim verification across N angles
- Comparative analysis across N analytical lenses
- Discovery-style consulting work where specialist lenses (operations / margin / knowledge / decision-rights / cognitive-load) all examine the same corpus

**Examples — code / ops:**
- Codebase audit across specialist lenses (security, performance, accessibility, readability)
- Multi-perspective design critique
- Opportunistic refactoring where each specialist applies the changes it sees

## When NOT to Use

- Cleanly parallel-independent work, no shared state needed → `approach:swarm`.
- Sequential with fixed handoffs → `approach:pipeline`.
- Single agent suffices → `approach:one-shot` or another single-worker approach.
- Quality iteration on one output → `approach:critic`.
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

- **Name** — what role it plays (security-reviewer, perf-optimizer, accessibility-auditor, margin-analyst, knowledge-mapper, …).
- **Worker** — Claude sub-Agent **(model: opus per [PRINCIPLES.md §2](../../PRINCIPLES.md#2-sub-agent-workers-always-use-opus))** with role-specific prompt, or Codex with role framing, or a specific subagent_type if one matches.
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

### 7. Apply the Direct-Use Rule

If the blackboard concerns work that has a real surface (a live codebase, a real client corpus, a deployed system), specialists should read from the real surface as their primary source, not from a summary. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible).

### 8. Output the contract

Write to `<contracts-root>/blackboard-<slug>.md` (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)):

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
- <none | each specialist is approach:pipeline | synthesis is approach:critic | …>

## Assumptions
<inferences during framing — specialist independence, workspace bounds — or "none">

## Extraction-confidence
<low | medium | high>
```

### 9. Stop

The skill is framing-only. Do not initialize the blackboard. Do not spawn specialists. The orchestrator follows.

## Anti-Patterns

- **Blackboard when Swarm would suffice.** Shared state for no reason is overhead. If specialists don't need to read what others wrote, use Swarm.
- **Too many specialists.** Coordination overhead dominates beyond ~7. Trim to the load-bearing roles.
- **Loose write access producing chaos.** Multiple specialists overwriting each other = lost work + confusion. Default to append-only.
- **No termination.** Blackboard work sprawls. Always cap.
- **Workspace structure invented per-fire.** Specialists need to know where to read/write *before* starting. Structure is part of the contract, not improvised at runtime.
- **Specialist prompts that don't reference the blackboard.** If the specialist's prompt doesn't tell it "read the blackboard at path X, append your findings to section Y," the approach isn't actually Blackboard — it's just N parallel agents producing isolated outputs (Swarm).
- **Hidden coordination through the orchestrator.** If the orchestrator is routing messages between specialists (specialist A finishes → orchestrator hands result to specialist B), that's Pipeline, not Blackboard. Blackboard agents pick up work autonomously by reading shared state.

## Rules

- The skill always outputs a single Markdown contract file; it never initializes or runs the blackboard.
- Every Blackboard contract must declare: workspace location/structure/format/initial-state, ≥3 specialists with worker/capability/trigger/output, write-access policy, termination, captured artifact, Assumptions, Extraction-confidence.
- Default write-access policy is append-only.
- Specialist count cap is ~7; beyond that, restructure.
- When a specialist is a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- Each specialist prompt must reference the blackboard explicitly (path + read/write sections); otherwise the approach isn't actually Blackboard.
- If the task doesn't fit Blackboard, recommend the right approach and stop — don't force-fit. Check `<available_skills>` before suggesting by name; if not installed, describe inline.
- Contract path: `<contracts-root>/blackboard-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/blackboard-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:event`, `approach:dialogue`, `approach:search`. Related external skills: `/loop` (Loop scheduling executor). Blackboard is the most exotic approach — use sparingly, restructure to Swarm/Pipeline when possible.
