---
name: composer
description: Composition planner for the shape family — frames a task into a tree of nested shape contracts. ALWAYS invoke this skill when the user asks to plan, compose, design a workflow, structure a system, scope a multi-shape effort, frame an approach, or describes a task with multiple shape signals (sequential + parallel; quality-critical + irreversible; trigger-fired + scheduled). Phase 1 extracts a Problem Statement via structured intake (end state, current state, gap, deliverable, verification, constraints, known/unknown, user role, stakes). Phases 2-5 map shapes, compose the nested tree, sanity-check cost vs value, write root + child contracts. Phase 7 supports adaptive recomposition when stage results diverge. Frame-only; does not execute. Do not structure multi-shape work ad-hoc — use this skill first. Skip when task obviously fits one shape (use that shape directly) or when the user is in execution mode.
# description-style: directive + negative constraint (Seleznov)
# rationale: Composer is the apex of the shape family. Without explicit framing, multi-shape tasks default to a single shape (under-structured) or to ad-hoc multi-tool work (no shape contract). Directive style forces the explicit composition step. Phase 1 (Problem extraction) is load-bearing — composition correctness is bounded by problem-statement precision.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (coding multi-shape): "Plan the OAuth2 migration: audit, map, refactor, test, document, then deploy to prod"
2. SHOULD activate (knowledge work multi-shape): "Help me compose a research project on X — gather sources, synthesize, critique, write the paper"
3. SHOULD NOT activate (single shape obvious): "Run the test suite and tell me what failed" (shape:one-shot)
4. BOUNDARY: "Help me plan a refactor" (could be shape:pipeline alone for simple refactor, or shape:composer if multi-shape — skill should run Phase 1 intake before deciding)
-->

# composer

Composition planner for the shape family. Reshapes itself in response to the task — extracts a precise Problem Statement first, then maps to a tree of nested shape contracts, hands off for execution. Frame-only, never executes. The single most important phase is Phase 1 (problem extraction): composition correctness is bounded by problem-statement precision.

## When to Use

- Task has multiple shape signals (sequential + parallel; quality-critical + irreversible; trigger-fired + scheduled).
- Problem definition is unclear or partially-formed — needs structured extraction before structuring.
- Multi-step work where no single shape obviously dominates.
- High-stakes or hard-to-undo work that earns explicit framing.
- Mixed coding + knowledge work in one effort (research → synthesize → build → review → ship).
- User explicitly invokes `/shape:composer` or says "compose," "plan a workflow," "design a system," "how should we approach."

## When NOT to Use

- Task obviously fits one shape — invoke that shape directly (`shape:pipeline`, `shape:swarm`, etc.).
- Trivial one-shot work — use `shape:one-shot` or just do it.
- User has already extracted the problem precisely and just wants execution — read the room.
- User is in pure execution mode — talking the action to death is friction.
- Composition cost would exceed the work itself — single shape or ad-hoc execution is the right call.

## Process

The skill outputs a **composition tree**: a root composition contract + per-shape child contracts referenced by path. Frame, write, hand back.

### Phase 1: Problem extraction (load-bearing)

**The most important phase.** Composition that doesn't anchor to a precise problem statement compounds errors downstream.

Apply `shape:dialogue` mode internally with a 9-target extraction template. Either run a real conversation (default) or, when prompt + context is already explicit, write the Problem Statement directly and mark `extraction-confidence: high`.

Extraction targets:

1. **Desired end state** — concrete, observable. "When this is done, what's true that isn't true now?"
2. **Current state** — what's actually true now (often differs from user's framing; surface honest gaps).
3. **Gap** — what's missing / broken / needed to bridge current → end.
4. **Deliverable shape** — artifact / decision / behavior change / knowledge produced / communication sent. Different deliverables invite different shapes.
5. **Verification criterion** — how do we know it's done? Deterministic (tests, lint, schema)? Judgment-bound (LLM critic, human review)? Human-only (taste, business decision)? This drives Critic vs Contract selection.
6. **Constraints** — time, cost, quality floor, scope boundary, reversibility, dependencies, blast radius.
7. **Known vs unknown** — information state. If unknown is high, invoke `/research` before Phase 3.
8. **User role** — where do they want gates (explicit Gated composition), where autonomy.
9. **Stakes** — why it matters. High stakes → more Critic, more Gated, less aggressive composition. Low stakes → fewer layers, simpler tree.

Write Problem Statement to `<contracts-root>/composition-<slug>-problem.md` BEFORE Phase 2. This file is the stable anchor; adaptive recomposition (Phase 7) reads from it.

**Quick-extract path:** Acceptable only when the user's prompt explicitly states all three of: (a) what success looks like (desired end state), (b) the verification criterion (how we'll know it's done), (c) the binding constraints (time, scope, reversibility, blast radius). If any one of these three is implicit or inferred, run the dialogue instead. Write the Problem Statement directly, mark `extraction-confidence: low|medium|high` and list `assumptions:` for anything inferred rather than stated. If composition fails later, low-confidence extractions are the first place to revisit.

**Intake fatigue is a real anti-pattern.** Do not over-extract. "Good enough to start composing" beats "perfectly defined." Aim for 2–4 minutes of dialogue; surface remaining unknowns as `Assumptions:` rather than blocking on them.

### Phase 2: Shape mapping

Given the Problem Statement, identify candidate shapes. Multiple matches is the composition signal.

Decision table:

| Signal | Shape |
|---|---|
| Multi-stage with handoffs, fixed ordering | `shape:pipeline` |
| Parallel-independent slices + aggregation step | `shape:swarm` |
| Quality is judgment-bound, iteration helps | `shape:critic` |
| Steps are irreversible / blast-radius / shared-state | `shape:gated` |
| Triggered by an event, not user-initiated | `shape:event` |
| Scheduled temporal recurrence | `shape:loop` |
| Broad/ambiguous/iterative, verifier-driven, fake-progress-prone | `shape:contract` |
| Explore-evaluate-prune across candidates | `shape:search` |
| Conversation IS the deliverable | `shape:dialogue` |
| Opportunistic multi-agent over shared workspace | `shape:blackboard` |
| Single action, no decomposition needed | `shape:one-shot` |

For each candidate shape, note WHY it matches (link back to specific Problem Statement fields). Vague matches are a sign of Phase 1 incompleteness — revisit before Phase 3.

If Phase 2 surfaces unknowns that block shape selection (e.g., "I don't know if this work is judgment-bound or deterministically-checkable"), invoke `/research` or extend Phase 1 before Phase 3.

### Phase 3: Composition planning

Build the tree.

**Top-level shape selection.** Pick the shape that dominates the whole task's structure:
- Sequential work with handoffs → `shape:pipeline` at the top
- Goal with verification spine → `shape:contract` at the top (Pipeline as a stage if multi-step)
- Single high-stakes action → `shape:gated` at the top
- Pure exploration → `shape:search` or `shape:dialogue` at the top

**Recursive descent.** For each stage/slice/producer/handler in the top-level shape:
- Re-apply the Phase 2 decision table to THAT sub-task
- If it matches a shape, nest
- If it's atomic, leave as direct work

**Worker assignment per stage (defaults):**

| Stage shape | Worker |
|---|---|
| Direction-setting, judgment, orchestration, design | Claude (this harness) |
| Long single-task execution, mechanical edits, deterministic refactors | Codex (`codex exec`) |
| Adversarial / critic review | Codex `/adversarial-review` |
| Isolated exploration in own context | sub-Agent **(model: opus)** |
| Pure read + transform (small input, single pass) | Claude inline |
| Pure read + transform (large input, deep analysis) | Codex |
| User authorization, taste decisions, business judgment | human |

Worker default lean: **Claude for direction, Codex for execution.** This is a per-stage decision, not a global default — re-evaluate per stage.

**Composition heuristics (cost-aware):**

- **Default to less.** Add a layer only when it earns its weight. A Pipeline alone is better than Pipeline+Critic+Gated when the work doesn't need all three.
- **Max nesting depth: 2.** Pipeline-with-Critic-stage is fine. Pipeline-with-Critic-whose-producer-is-Pipeline-with-Swarm-stage is opaque. If you need >2 levels, restructure or document the explicit reason.
- **Don't wrap atomic stages.** A single-action stage doesn't need a One-shot wrapper inside a Pipeline — that's ceremony.
- **Critic earns Critic when the output is judgment-bound + quality matters + producer is over-confident-prone.** Otherwise let the stage be plain.
- **Gate at the smallest dangerous step.** Don't gate everything; gate the actually-irreversible step.
- **Contract wraps the whole tree when fake-progress risk is high.** It's the verification spine. Use sparingly.

**/research as callable:** invoke `/research` from inside a Pipeline gather stage when sources need to be collected, or as a Phase 1 sub-step when problem extraction needs external context. Never auto-invoke without explicit need.

### Phase 4: Sanity check

Cost-vs-value review before writing the composition. Five checks:

1. **Composition cost.** Would the composition contract take longer to write than the actual work? If yes, simplify or skip framing.
2. **Layer justification.** Each layer (Critic, Gated, etc.) earns its weight per Phase 3 heuristics. Strip layers that don't.
3. **Nesting depth.** ≤2 levels unless explicit reason. Beyond that, restructure.
4. **Termination chain.** Each shape's termination must link to the parent's termination. Orphan terminations leak.
5. **Gate placement.** Gates at irreversible steps only. Reversible-local-work gates are friction without value.

If any check flags, revise before Phase 5.

### Phase 5: Output

Write the composition tree to `<contracts-root>/`:

- **Root**: `composition-<slug>.md` — the composition contract. Describes the top-level shape, references all per-shape contracts by path, captures rationale.
- **Top-level shape contract**: `<shape>-<slug>.md` — the contract for the dominant shape (e.g., `pipeline-<slug>.md`). No subname suffix.
- **Nested children**: `<shape>-<slug>-<subname>.md` for each nested shape. `<subname>` is a short kebab-case identifier describing the role the shape plays in its parent (e.g., `stage3-apply`, `slice-review`, `producer`, `reducer`, `judge`). Subnames must be unique within the composition.

Root contract shape:

```markdown
# Composition contract: <task-name>

## Problem Statement
- See `composition-<slug>-problem.md` for full extraction.
- Summary: <one-paragraph distillation>
- Verification: <how we'll know the whole composition is done>

## Top-level shape
- Shape: <pipeline | contract | gated | search | dialogue | …>
- Rationale: <why this is the dominant shape>
- Reference: `<shape>-<slug>.md`

## Nested shapes
| Stage / slice / producer / role | Shape | Contract file | Why nested |
|---|---|---|---|
| <stage 1 name> | `shape:critic` | `critic-<slug>-stage1.md` | <reason> |
| <stage 2 name> | `shape:swarm` | `swarm-<slug>-stage2.md` | <reason> |

## Worker map
- <stage> → <worker> (rationale, 1 line)

## Cost / sanity notes
- Nesting depth: <N>
- Layer-justification summary: <one line per layer>
- Composition cost estimate: <quick assessment>
- Nested-of-nested: <if any child contract is itself composed, note here. The "Nested shapes" table covers direct nesting; this line tracks deeper composition. Hidden composition is forbidden.>

## On adaptive recomposition
- Trigger: <stage failure | stage output diverges from expectation | explicit user signal>
- Behavior: re-read this composition root + Problem Statement; revise downstream stages or restart from current point; update this file with the revision.
```

Each child contract is the output of the corresponding shape skill, written to its own file at `<contracts-root>/<shape>-<slug>-<subname>.md`. The composer doesn't re-implement the per-shape framing — it conceptually invokes each shape skill's Process internally and writes the resulting contract.

**Child contract's `Compositions` field** (every shape skill's contract template has one): the composer populates it with `parent: composition-<slug>.md` (link upward to the composition root) plus any sibling references the child needs (e.g., a Critic child that reviews the output of a Pipeline sibling lists the sibling). This makes the tree navigable in both directions.

### Phase 6: Hand off

Stop. Do not execute. Surface to the orchestrator (the user, the future runtime, or Claude in a follow-up turn):
- Path to composition root
- Tree summary (top-level shape + nested shapes with their rationale)
- Any open assumptions or low-confidence items from Phase 1 that should be revisited on first execution

### Phase 7: Adaptive recomposition (when re-invoked)

Phase 7 fires only when composer is explicitly re-invoked with context about a completed stage — typically by the user or orchestrator saying "stage X ran, here's what came back, should we adjust?" The composer does not auto-track stage execution; the trigger is a fresh invocation pointing at an existing composition root.

If composer is invoked after a stage has executed:
1. **Read existing state**: composition root, Problem Statement, the stage that just ran + its actual output.
2. **Compare**: expected output (from the relevant child contract) vs actual.
3. **Decide**:
   - Continue with original plan (output matches expectation)
   - Adjust downstream stages (output diverges but plan still valid with tweaks)
   - Re-plan from current point (plan no longer fits the actual state)
4. **Update**: write the revision into the composition root + affected child contracts. Mark revision with timestamp + reason.
5. **Surface diff** to user before continuing.

Adaptive recomposition is opt-in — only invoked when stage results suggest the plan is wrong. Default to the original plan if results match.

## Anti-Patterns

- **Skipping Phase 1 (problem extraction).** The compounding error. Composition that doesn't anchor to a precise problem statement is theater. The 9 extraction targets are non-optional; mark `extraction-confidence: low` and `assumptions:` rather than skipping.
- **Over-composing on simple tasks.** Adding Critic + Gated + Contract layers to a task that needs Pipeline alone. Default to less; earn each layer per the Phase 3 heuristics.
- **Composition contract longer than the underlying work.** Symptom of over-composing or treating composer as a deliverable in itself. Composition cost should be a fraction of the work it frames.
- **Intake fatigue.** Phase 1 dialogue stuck on perfecting the problem statement when "good enough" would unblock. Surface remaining unknowns as `Assumptions:` and move on.
- **Hidden composition.** A stage that's actually a Critic-around-Pipeline but the root contract just calls it "Pipeline stage 3." Compositions must be explicit; the root's `Nested shapes` table is the audit trail.
- **Nesting depth >2 without explicit reason.** Trees deeper than 2 levels become unreadable. Restructure or document why.
- **Stale composition.** Stage results diverge from expectations, composer not re-invoked, original plan continues blindly. Adaptive recomposition exists for this; surface the divergence before continuing.
- **Composer replaces user judgment.** Composer surfaces structure; it doesn't decide what the user actually wants. When in doubt, ask in Phase 1, don't infer.
- **/research auto-invoked without need.** Burning research calls every composition is wasteful. Only invoke when Phase 1 surfaces a specific blocking unknown.
- **Worker assignment defaulted to "Claude for everything."** Wastes Codex's strengths on long-mechanical stages. Worker assignment is per-stage; audit each in Phase 3.

## Rules

- The skill always outputs a composition tree (root + children); it never executes.
- Phase 1 (Problem extraction) is non-optional. If skipped via quick-extract path, the Problem Statement file must mark `extraction-confidence` and `assumptions`.
- Every composition root must declare: Problem Statement reference, top-level shape, nested shapes with rationale, worker map, sanity-check notes, adaptive-recomposition trigger.
- Nested shapes are always written as separate child contract files (option (b) — not embedded inline in the root).
- Max nesting depth: 2 levels. Beyond that requires explicit reason documented in the root's `Cost / sanity notes`.
- Worker assignment per stage is intentional, not defaulted. Default lean: Claude for direction-setting/judgment/orchestration; Codex for execution.
- When a worker is a Claude sub-Agent, the contract MUST specify `model: opus`. Never Sonnet, never Haiku.
- `/research` is callable only when Phase 1 or Phase 2 surfaces a specific blocking unknown; never auto-invoke.
- Adaptive recomposition is opt-in: invoked only when stage results diverge from expectation. Default to original plan if results match.
- Contract paths: `<contracts-root>/composition-<slug>.md` (root), `<contracts-root>/composition-<slug>-problem.md` (problem statement), `<contracts-root>/<shape>-<slug>-<subname>.md` (children). `<contracts-root>` resolution (in order): user-specified path > task-implied project folder > cwd if it is a project (has `.git/` or `CLAUDE.md`) → `<project>/.claude/contracts/` > fallback `/tmp/shape-contracts/`.
- If the task doesn't warrant composition (one shape suffices, or work is trivial), say so and recommend the single shape — don't force-compose. The Problem Statement file is still written when Phase 1 ran a real dialogue (the extraction has value beyond composition); it is not written when the quick-extract path was used and the conclusion is "no composition needed."

## Key Files

- Output (root): `<contracts-root>/composition-<slug>.md` — composition contract referencing children.
- Output (problem statement): `<contracts-root>/composition-<slug>-problem.md` — the Phase 1 extraction.
- Output (children): `<contracts-root>/<shape>-<slug>-<subname>.md` — per-shape contracts.
- Sibling shape skills (all under the `shape` plugin namespace, all live): `shape:pipeline`, `shape:swarm`, `shape:critic`, `shape:gated`, `shape:event`, `shape:contract`, `shape:loop`, `shape:one-shot`, `shape:search`, `shape:dialogue`, `shape:blackboard`. Composer invokes their Process conceptually when generating child contracts.
- External callable: `/research` (sourced findings on a topic; invoked when Phase 1/2 surfaces a blocking unknown).
- External executors (post-composition, invoked by orchestrator not composer): `codex exec`, `Agent` tool, `/loop`, `/schedule`, `/update-config`.

## Worked example

User prompt: *"I need to migrate the auth module to OAuth2, make sure it's safe, and document the change for the team."*

**Phase 1 — Problem Statement** (written to `<contracts-root>/composition-auth-oauth-problem.md`):
- End state: auth module uses OAuth2; existing user sessions still work; migration documented at `docs/auth-migration.md`.
- Current state: custom session-token auth across ~15 files.
- Gap: replace session tokens with OAuth2 + adapter for legacy sessions.
- Deliverable: code change + migration doc.
- Verification: tests green + OAuth2 login smoke + docs reviewed.
- Constraints: cannot break existing sessions; rollback path required; production deploy is gated.
- Known/unknown: lib chosen (Auth0); unknown legacy quirks.
- User role: gate on production deploy; autonomy on local.
- Stakes: high (security-critical, user-facing).
- `extraction-confidence: high` (all 9 targets stated or directly inferable).

**Phase 2 — Shape mapping:**
- `shape:pipeline` (sequential audit → map → apply → test → document)
- `shape:critic` (apply stage is security-critical, judgment-bound review)
- `shape:gated` (production deploy is irreversible)
- `shape:contract` (high-stakes, fake-progress-prone — wrap the spine)

**Phase 3 — Composition:**
- Top-level: `shape:contract` (verification spine wrapping the whole effort)
- Stage spine: `shape:pipeline` (audit → map → apply → test → document → deploy)
- Apply stage nests `shape:critic` (Codex `/adversarial-review` on the diff)
- Deploy stage nests `shape:gated` (human approves production)

**Phase 4 — Sanity check:**
- Nesting depth: 2 (Contract → Pipeline → Critic; Contract → Pipeline → Gated). Within cap.
- Layers earn weight: Contract for fake-progress risk, Critic for security, Gated for irreversibility. None redundant.
- Termination chain: Pipeline stages have concrete checks; Contract spine has named verifiers; Gated has explicit approve/reject.

**Phase 5 — Output files at `<contracts-root>/`:**
- `composition-auth-oauth.md` (root)
- `composition-auth-oauth-problem.md` (Problem Statement)
- `contract-auth-oauth.md` (top-level Contract)
- `pipeline-auth-oauth-spine.md` (child Pipeline, the verification spine)
- `critic-auth-oauth-apply-review.md` (child Critic for apply stage)
- `gated-auth-oauth-deploy.md` (child Gated for production deploy)

**Phase 6 — Hand off:** surface root path + tree summary. Note open assumption: "legacy session quirks unknown, will surface in audit stage."

## Scope boundary

The composer FRAMES; it does not EXECUTE. It produces a composition tree; the orchestrator (the user, Claude in a follow-up turn, or the future runtime) follows the tree. The composer's product is documents, not actions. The one exception is `/research` invocation when Phase 1/2 surfaces a blocking unknown — and even then, the research result feeds back into Phase 1 (Problem Statement update), not into action.
