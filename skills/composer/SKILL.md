---
name: composer
description: Composition planner for the approach family — frames a task into a tree of nested approach contracts. ALWAYS invoke this skill when the user asks to plan, compose, design a workflow, structure a system, scope a multi-approach effort, frame how to tackle something, or describes a task with multiple approach signals (sequential + parallel; quality-critical + irreversible; trigger-fired + scheduled). Phase 1 extracts a Problem Statement via structured intake (end state, current state, gap, deliverable, verification, constraints, known/unknown, user role, stakes) — frame work, transitional inline; long-term migrates to a future /frame plugin. Phases 2-5 map approaches, compose the nested tree, sanity-check cost vs value, write root + child contracts. Phase 7 supports adaptive recomposition when stage results diverge. Frame-only; does not execute. Do not structure multi-approach work ad-hoc — use this skill first. Skip when task obviously fits one approach (use that approach directly) or when the user is in execution mode.
# description-style: directive + negative constraint (Seleznov)
# rationale: Composer is the apex of the approach family. Without explicit framing, multi-approach tasks default to a single approach (under-structured) or to ad-hoc multi-tool work (no contract). Directive style forces the explicit composition step. Phase 1 (Problem extraction) is load-bearing — composition correctness is bounded by problem-statement precision.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (coding multi-approach): "Plan the OAuth2 migration: audit, map, refactor, test, document, then deploy to prod"
2. SHOULD activate (knowledge work multi-approach): "Help me compose a research project on X — gather sources, synthesize, critique, write the paper"
3. SHOULD NOT activate (single approach obvious): "Run the test suite and tell me what failed" (approach:one-shot)
4. BOUNDARY: "Help me plan a refactor" (could be approach:pipeline alone for simple refactor, or approach:composer if multi-approach — skill should run Phase 1 intake before deciding)
-->

# composer

Composition planner for the `approach` family. Reshapes itself in response to the task — extracts a precise Problem Statement first, then maps to a tree of nested approach contracts, hands off for execution. Frame-only, never executes. The single most important phase is Phase 1 (problem extraction): composition correctness is bounded by problem-statement precision.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions+extraction-confidence, Direct-Use Rule, contracts-root resolution, composition explicitness, knowledge-work parity, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model and where this plugin sits.

## When to Use

- Task has multiple approach signals (sequential + parallel; quality-critical + irreversible; trigger-fired + scheduled).
- Problem definition is unclear or partially-formed — needs structured extraction before structuring.
- Multi-step work where no single approach obviously dominates.
- High-stakes or hard-to-undo work that earns explicit framing.
- Mixed coding + knowledge work in one effort (research → synthesize → build → review → ship).
- User explicitly invokes `/approach:composer` or says "compose," "plan a workflow," "design a system," "how should we approach."

## When NOT to Use

- Task obviously fits one approach — invoke that approach directly (`approach:pipeline`, `approach:swarm`, etc.).
- Trivial one-shot work — use `approach:one-shot` or just do it.
- User has already extracted the problem precisely and just wants execution — read the room.
- User is in pure execution mode — talking the action to death is friction.
- Composition cost would exceed the work itself — single approach or ad-hoc execution is the right call.

## Process

The skill outputs a **composition tree**: a root composition contract + per-approach child contracts referenced by path. Frame, write, hand back.

### Phase 0: Check for /frame plugin hand-off

Before running Phase 1's inline extraction, check whether the user (or an upstream call) supplied a `frame:problem-statement` artifact from the `/frame` plugin:

**Trigger conditions** (any one is sufficient):
- User invocation includes a path to a `problem-statement.md` (e.g., `/approach:composer ./frame-<slug>/problem-statement.md`)
- User invocation references a `frame-<slug>/` folder produced by `/frame:composer`
- Conversation context indicates the user just ran `/frame:composer` and pointed at its output

**If a Problem Statement artifact is supplied:**

1. **Read the artifact** via Read tool from the supplied path.
2. **Validate** that it contains the 9 Polya targets (end state, current state, gap, deliverable shape, verification criterion, constraints, known/unknown, user role, stakes) plus `Extraction-confidence` and `Assumptions` fields. If any required field is missing, recommend the user run `/frame:problem-statement` to complete the artifact and stop.
3. **Inherit** the `Extraction-confidence` and `Assumptions` markers — these flow through to the composition root.
4. **Surface to user**: *"Consuming Problem Statement from /frame at `<path>` (extraction-confidence: `<level>`). Skipping Phase 1 inline extraction. Proceeding to Phase 2 approach mapping."*
5. **Skip Phase 1 entirely**. Proceed to Phase 2.
6. **In the composition root**, the "Problem Statement" section references the supplied artifact path rather than the inline `composition-<slug>-problem.md` — *no need to re-write the extraction*.

This is the architecturally clean hand-off documented in `/frame`'s ARCHITECTURE.md. When `/frame` is in use, composer's frame work is delegated cleanly upstream.

**If no Problem Statement artifact is supplied:**

Proceed to Phase 1 (transitional inline frame extraction) below.

### Phase 1: Problem extraction (load-bearing — *transitional frame work*)

**The most important phase when Phase 0 didn't fire.** If `/frame` produced a Problem Statement upstream (Phase 0 above), skip this entire phase and go to Phase 2.

Composition that doesn't anchor to a precise problem statement compounds errors downstream.

> **Architectural note.** Phase 1 is *frame-space* work — problem articulation, perception, what to include/exclude. The rest of composer (Phases 2-5, 7) is *approach-space* work. Phase 1 is inlined here transitionally for users who don't have the `/frame` plugin installed; users with `/frame` installed should run `/frame:composer` first and then invoke `/approach:composer` with the Problem Statement path (Phase 0 above handles this).
>
> The practical consequence today: when this phase runs (no upstream /frame), it deserves more depth than the approach-selection phases that follow. Frame work pays for itself across every downstream choice.

Apply `approach:dialogue` mode internally with a 9-target extraction template. Either run a real conversation (default) or, when prompt + context is already explicit, write the Problem Statement directly and mark `extraction-confidence: high`.

Extraction targets:

1. **Desired end state** — concrete, observable. "When this is done, what's true that isn't true now?"
2. **Current state** — what's actually true now (often differs from user's framing; surface honest gaps).
3. **Gap** — what's missing / broken / needed to bridge current → end.
4. **Deliverable shape** — artifact / decision / behavior change / knowledge produced / communication sent. Different deliverables invite different approaches.
5. **Verification criterion** — how do we know it's done? Deterministic (tests, lint, schema)? Judgment-bound (LLM critic, human review)? Human-only (taste, business decision)? This drives Critic vs Contract selection.
6. **Constraints** — time, cost, quality floor, scope boundary, reversibility, dependencies, blast radius.
7. **Known vs unknown** — information state. If unknown is high, invoke `/research` before Phase 3.
8. **User role** — where do they want gates (explicit Gated composition), where autonomy.
9. **Stakes** — why it matters. High stakes → more Critic, more Gated, less aggressive composition. Low stakes → fewer layers, simpler tree.

Write Problem Statement to `<contracts-root>/composition-<slug>-problem.md` BEFORE Phase 2. This file is the stable anchor; adaptive recomposition (Phase 7) reads from it.

**Quick-extract path:** Acceptable only when the user's prompt explicitly states all three of: (a) what success looks like (desired end state), (b) the verification criterion (how we'll know it's done), (c) the binding constraints (time, scope, reversibility, blast radius). If any one of these three is implicit or inferred, run the dialogue instead. Write the Problem Statement directly, mark `extraction-confidence: low|medium|high` and list `Assumptions:` for anything inferred rather than stated. If composition fails later, low-confidence extractions are the first place to revisit (see [PRINCIPLES.md §3](../../PRINCIPLES.md#3-assumptions--extraction-confidence--surface-uncertainty-dont-hide-it)).

**Intake fatigue is real, but frame depth pays.** Frame work deserves more time than approach selection — typically 4-8 minutes of dialogue for non-trivial problems. The earlier "2-4 minutes, surface unknowns as Assumptions" guidance was calibrated to approach-selection intake; for problem extraction specifically, error here compounds across every downstream phase. That said, do not become stuck on perfecting the problem statement — surface remaining unknowns as `Assumptions:` and move forward when the core 9 targets are populated, even if some are at `medium` confidence.

### Phase 2: Approach mapping

Given the Problem Statement, identify candidate approaches. Multiple matches is the composition signal.

Decision table:

| Signal | Approach |
|---|---|
| Multi-stage with handoffs, fixed ordering | `approach:pipeline` |
| Parallel-independent slices + aggregation step | `approach:swarm` |
| Quality is judgment-bound, iteration helps | `approach:critic` |
| Steps are irreversible / blast-radius / shared-state | `approach:gated` |
| Triggered by an event, not user-initiated | `approach:event` |
| Scheduled temporal recurrence | `approach:loop` |
| Broad/ambiguous/iterative, verifier-driven, fake-progress-prone | `approach:contract` |
| Explore-evaluate-prune across candidates | `approach:search` |
| Conversation IS the deliverable | `approach:dialogue` |
| Opportunistic multi-agent over shared workspace | `approach:blackboard` |
| Single action, no decomposition needed | `approach:one-shot` |

**Strict citation requirement.** For each candidate approach, **explicitly cite the specific Problem Statement field(s) that justify it**. Format: `approach:critic — justified by Verification (judgment-bound, no green-test signal) + Stakes (high — security-critical)`. Vague matches ("seems like a Pipeline") are a sign of Phase 1 incompleteness — kick back to Phase 1 before proceeding.

This citation requirement enforces the architectural invariant: frame defines the approach space. If a candidate approach has no field-level justification, it's not really licensed by the problem — it's just an inference, and the inference should be made visible as an Assumption rather than baked into the composition.

If Phase 2 surfaces unknowns that block approach selection (e.g., "I don't know if this work is judgment-bound or deterministically-checkable"), invoke `/research` or extend Phase 1 before Phase 3.

### Phase 3: Composition planning

Build the tree.

**Top-level approach selection.** Pick the approach that dominates the whole task's structure:
- Sequential work with handoffs → `approach:pipeline` at the top
- Goal with verification spine → `approach:contract` at the top (Pipeline as a stage if multi-step)
- Single high-stakes action → `approach:gated` at the top
- Pure exploration → `approach:search` or `approach:dialogue` at the top

**Recursive descent.** For each stage/slice/producer/handler in the top-level approach:
- Re-apply the Phase 2 decision table to THAT sub-task
- If it matches an approach, nest
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

Worker default lean: **Claude for direction, Codex for execution.** This is a per-stage decision, not a global default — re-evaluate per stage. See [PRINCIPLES.md §2](../../PRINCIPLES.md#2-sub-agent-workers-always-use-opus) for the model-opus rule on Claude sub-Agents.

**Composition heuristics (cost-aware):**

- **Default to less.** Add a layer only when it earns its weight. A Pipeline alone is better than Pipeline+Critic+Gated when the work doesn't need all three.
- **Max nesting depth: 2.** Pipeline-with-Critic-stage is fine. Pipeline-with-Critic-whose-producer-is-Pipeline-with-Swarm-stage is opaque. If you need >2 levels, restructure or document the explicit reason.
- **Don't wrap atomic stages.** A single-action stage doesn't need a One-shot wrapper inside a Pipeline — that's ceremony.
- **Critic earns Critic when the output is judgment-bound + quality matters + producer is over-confident-prone.** Otherwise let the stage be plain.
- **Gate at the smallest dangerous step.** Don't gate everything; gate the actually-irreversible step.
- **Contract wraps the whole tree when fake-progress risk is high.** It's the verification spine. Use sparingly.
- **Adversarial review at completion for long-running work.** Compositions spanning multiple stages, hours of work, or high-stakes output should include a final `approach:critic` pass (preferably with Codex `/adversarial-review` as the critic worker). Quality compounds with time invested; the critic catches what the producer missed and prevents fake-progress shipping. Default-on for long-running compositions; skip only when work is genuinely short and low-stakes.

**Direct-Use Rule applies.** Any stage that can exercise the real intended surface (deployed endpoint, installed MCP, real browser, real client) should do so as early as possible. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible).

**/research as callable:** invoke `/research` from inside a Pipeline gather stage when sources need to be collected, or as a Phase 1 sub-step when problem extraction needs external context. Never auto-invoke without explicit need.

### Phase 4: Sanity check

Cost-vs-value review before writing the composition. Five checks:

1. **Composition cost.** Would the composition contract take longer to write than the actual work? If yes, simplify or skip framing.
2. **Layer justification.** Each layer (Critic, Gated, etc.) earns its weight per Phase 3 heuristics. Strip layers that don't.
3. **Nesting depth.** ≤2 levels unless explicit reason. Beyond that, restructure.
4. **Termination chain.** Each approach's termination must link to the parent's termination. Orphan terminations leak.
5. **Gate placement.** Gates at irreversible steps only. Reversible-local-work gates are friction without value.

If any check flags, revise before Phase 5.

### Phase 5: Output

Write the composition tree to `<contracts-root>/`:

- **Root**: `composition-<slug>.md` — the composition contract. Describes the top-level approach, references all per-approach contracts by path, captures rationale.
- **Top-level approach contract**: `<approach>-<slug>.md` — the contract for the dominant approach (e.g., `pipeline-<slug>.md`). No subname suffix.
- **Nested children**: `<approach>-<slug>-<subname>.md` for each nested approach. `<subname>` is a short kebab-case identifier describing the role the approach plays in its parent (e.g., `stage3-apply`, `slice-review`, `producer`, `reducer`, `judge`). Subnames must be unique within the composition.

Root contract shape:

```markdown
# Composition contract: <task-name>

## Problem Statement
- See `composition-<slug>-problem.md` for full extraction.
- Summary: <one-paragraph distillation>
- Verification: <how we'll know the whole composition is done>
- Extraction-confidence: <low | medium | high>
- Assumptions: <inferences made during framing, or "none">

## Top-level approach
- Approach: <pipeline | contract | gated | search | dialogue | …>
- Rationale: <why this is the dominant approach, citing Problem Statement fields>
- Reference: `<approach>-<slug>.md`

## Nested approaches
| Stage / slice / producer / role | Approach | Contract file | Why nested (Problem Statement citation) |
|---|---|---|---|
| <stage 1 name> | `approach:critic` | `critic-<slug>-stage1.md` | <reason + cited field> |
| <stage 2 name> | `approach:swarm` | `swarm-<slug>-stage2.md` | <reason + cited field> |

## Worker map
- <stage> → <worker> (rationale, 1 line)

## Cost / sanity notes
- Nesting depth: <N>
- Layer-justification summary: <one line per layer>
- Composition cost estimate: <quick assessment>
- Nested-of-nested: <if any child contract is itself composed, note here. The "Nested approaches" table covers direct nesting; this line tracks deeper composition. Hidden composition is forbidden per PRINCIPLES.md §6.>

## On adaptive recomposition
- Trigger: <stage failure | stage output diverges from expectation | explicit user signal>
- Behavior: re-read this composition root + Problem Statement; revise downstream stages or restart from current point; update this file with the revision.
```

Each child contract is the output of the corresponding approach skill, written to its own file at `<contracts-root>/<approach>-<slug>-<subname>.md`. The composer doesn't re-implement the per-approach framing — it conceptually invokes each approach skill's Process internally and writes the resulting contract.

**Child contract's `Compositions` field** (every approach skill's contract template has one): the composer populates it with `parent: composition-<slug>.md` (link upward to the composition root) plus any sibling references the child needs (e.g., a Critic child that reviews the output of a Pipeline sibling lists the sibling). This makes the tree navigable in both directions.

### Phase 6: Hand off

Stop. Do not execute. Surface to the orchestrator (the user, the future runtime, or Claude in a follow-up turn):
- Path to composition root
- Tree summary (top-level approach + nested approaches with their rationale)
- Any open assumptions or low-confidence items from Phase 1 that should be revisited on first execution

### Phase 7: Adaptive recomposition (when re-invoked)

Phase 7 fires only when composer is explicitly re-invoked with context about a completed stage — typically by the user or orchestrator saying "stage X ran, here's what came back, should we adjust?" The composer does not auto-track stage execution; the trigger is a fresh invocation pointing at an existing composition root.

If composer is invoked after a stage has executed:
1. **Read existing state**: composition root, Problem Statement, the stage that just ran + its actual output.
2. **Compare**: expected output (from the relevant child contract) vs actual.
3. **Decide**:
   - Continue with original plan (output matches expectation)
   - Adjust downstream stages (output diverges but plan still valid with tweaks) — **re-compose**
   - Re-plan from current point (plan no longer fits the actual state) — **re-compose at scale**
   - Problem Statement itself is wrong (the divergence reveals the frame was off) — **re-frame**, which means re-running Phase 1
4. **Update**: write the revision into the composition root + affected child contracts. Mark revision with timestamp + reason.
5. **Surface diff** to user before continuing.

**Re-compose vs re-frame distinction.** Re-composing adjusts the approach tree within the existing Problem Statement. Re-framing rewrites the Problem Statement itself (which then forces a re-compose). Re-framing is more expensive but sometimes correct — surfacing it as a possibility is part of Phase 7's job, even though the actual re-frame happens upstream (in Dialogue or, future, in `/frame`).

Adaptive recomposition is opt-in — only invoked when stage results suggest the plan is wrong. Default to the original plan if results match.

## Anti-Patterns

- **Skipping Phase 1 (problem extraction).** The compounding error. Composition that doesn't anchor to a precise problem statement is theater. The 9 extraction targets are non-optional; mark `extraction-confidence: low` and `Assumptions:` rather than skipping.
- **Over-composing on simple tasks.** Adding Critic + Gated + Contract layers to a task that needs Pipeline alone. Default to less; earn each layer per the Phase 3 heuristics.
- **Composition contract longer than the underlying work.** Symptom of over-composing or treating composer as a deliverable in itself. Composition cost should be a fraction of the work it frames.
- **Intake fatigue at the wrong level.** Phase 1 dialogue stuck on perfecting the problem statement when "good enough on the 9 targets" would unblock. The fix is *not* to skip frame work — it's to surface remaining unknowns as `Assumptions:` and move on, while still giving frame work the depth it deserves. Frame work shallow ≠ frame work fast.
- **Vague approach citations in Phase 2.** "Seems like a Pipeline" without naming the Problem Statement field that justifies it. Symptom of Phase 1 incompleteness or of mapping by vibes rather than by license. Kick back to Phase 1 or rewrite the citation explicitly.
- **Hidden composition.** A stage that's actually a Critic-around-Pipeline but the root contract just calls it "Pipeline stage 3." Compositions must be explicit; the root's `Nested approaches` table is the audit trail (PRINCIPLES.md §6).
- **Nesting depth >2 without explicit reason.** Trees deeper than 2 levels become unreadable. Restructure or document why.
- **Stale composition.** Stage results diverge from expectations, composer not re-invoked, original plan continues blindly. Adaptive recomposition exists for this; surface the divergence before continuing.
- **Composer replaces user judgment.** Composer surfaces structure; it doesn't decide what the user actually wants. When in doubt, ask in Phase 1, don't infer.
- **/research auto-invoked without need.** Burning research calls every composition is wasteful. Only invoke when Phase 1 surfaces a specific blocking unknown.
- **Worker assignment defaulted to "Claude for everything."** Wastes Codex's strengths on long-mechanical stages. Worker assignment is per-stage; audit each in Phase 3.
- **Conflating re-compose with re-frame in Phase 7.** Adjusting downstream stages is cheap; rewriting the Problem Statement is expensive but sometimes necessary. Don't silently re-frame by hand-editing the Problem Statement — surface it as a re-frame decision.

## Rules

- The skill always outputs a composition tree (root + children); it never executes.
- Phase 1 (Problem extraction) is non-optional. If skipped via quick-extract path, the Problem Statement file must mark `extraction-confidence` and `Assumptions`.
- Every composition root must declare: Problem Statement reference, top-level approach, nested approaches with rationale (citing Problem Statement fields), worker map, sanity-check notes, adaptive-recomposition trigger.
- Phase 2 candidate citations must reference specific Problem Statement fields. Vague matches are rejected.
- Nested approaches are always written as separate child contract files (option (b) — not embedded inline in the root).
- Max nesting depth: 2 levels. Beyond that requires explicit reason documented in the root's `Cost / sanity notes`.
- Worker assignment per stage is intentional, not defaulted. Default lean: Claude for direction-setting/judgment/orchestration; Codex for execution.
- When a worker is a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- `/research` is callable only when Phase 1 or Phase 2 surfaces a specific blocking unknown; never auto-invoke.
- Adaptive recomposition is opt-in: invoked only when stage results diverge from expectation. Default to original plan if results match.
- Phase 7 must distinguish re-compose (approach tree adjustment) from re-frame (Problem Statement rewrite). Re-framing is a deliberate, surfaced decision.
- Long-running compositions (multiple stages, hours of work, or high-stakes output) MUST include an adversarial review pass at completion — typically a final `approach:critic` stage with Codex `/adversarial-review` as the critic worker. Default-on for long-running work; explicit opt-out required for short low-stakes compositions.
- Contract paths: `<contracts-root>/composition-<slug>.md` (root), `<contracts-root>/composition-<slug>-problem.md` (problem statement), `<contracts-root>/<approach>-<slug>-<subname>.md` (children). `<contracts-root>` resolution per PRINCIPLES.md §5.
- If the task doesn't warrant composition (one approach suffices, or work is trivial), say so and recommend the single approach — don't force-compose. The Problem Statement file is still written when Phase 1 ran a real dialogue (the extraction has value beyond composition); it is not written when the quick-extract path was used and the conclusion is "no composition needed."

## Key Files

- Output (root): `<contracts-root>/composition-<slug>.md` — composition contract referencing children.
- Output (problem statement): `<contracts-root>/composition-<slug>-problem.md` — the Phase 1 extraction.
- Output (children): `<contracts-root>/<approach>-<slug>-<subname>.md` — per-approach contracts.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:event`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:search`, `approach:dialogue`, `approach:blackboard`. Composer invokes their Process conceptually when generating child contracts.
- External callable: `/research` (sourced findings on a topic; invoked when Phase 1/2 surfaces a blocking unknown).
- External callable: `/gut-check` — the canonical lightweight human-invoked alignment check during multi-stage execution. When the user wants the active stage worker to honestly self-assess (understanding, telos, progress, definition-of-done, what to keep/change), they invoke `/gut-check`. Composer should NOT auto-invoke it; surface it as available so the orchestrator/user knows it exists. Useful between stages or whenever stage results feel off and the user wants a plain-language read before deciding whether to recompose (Phase 7).
- External executors (post-composition, invoked by orchestrator not composer): `codex exec`, `Agent` tool, `/loop`, `/schedule`, `/update-config`.
- Future sibling plugin: `/frame` — when it ships, Phase 1 delegates to it (likely `/frame:composer` or `/frame:problem-statement`).

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
- `Assumptions:` legacy session quirks are surface-discoverable in audit stage; rollback via feature flag (not deployment-level rollback).

**Phase 2 — Approach mapping (with citations):**
- `approach:pipeline` — justified by *Deliverable* (sequential: audit → map → apply → test → document) + *Gap* (multi-step bridge work)
- `approach:critic` — justified by *Stakes* (high — security-critical) + *Verification* (apply stage is judgment-bound: "is this OAuth2 implementation actually correct"; not deterministically checkable from tests alone)
- `approach:gated` — justified by *Constraints* (production deploy is irreversible)
- `approach:contract` — justified by *Stakes* (high) + fake-progress risk on security work

**Phase 3 — Composition:**
- Top-level: `approach:contract` (verification spine wrapping the whole effort)
- Stage spine: `approach:pipeline` (audit → map → apply → test → document → deploy)
- Apply stage nests `approach:critic` (Codex `/adversarial-review` on the diff)
- Deploy stage nests `approach:gated` (human approves production)

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

Phase 1 (problem extraction) is *frame-space* work that this skill currently absorbs inline. When the future `/frame` plugin ships, this scope tightens — composer becomes a pure approach-space tool that consumes a Problem Statement from `/frame:composer` or equivalent. The transition is opaque to users: today, invoking composer also does frame work; tomorrow, composer assumes the framing was already done and produces the approach tree from it.
