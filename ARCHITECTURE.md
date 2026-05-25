# ARCHITECTURE

The conceptual model the `approach` plugin sits inside, and where this plugin's responsibility starts and stops.

---

## The four-space model

Any non-trivial task can be located in one of four cognitive spaces:

| Space | What | Cognitive operation | Where AI helps |
|---|---|---|---|
| **Problem** | the situation in the world | (pre-language; no operation yet) | — |
| **Frame** | how the situation is articulated, what filter you apply, what you let through | articulation + perception | **questioner / probe** — clarifies seeing, surfaces what's assumed |
| **Approach** | how you'll work on the framed problem (which structural pattern, which decomposition) | structural composition | **pattern-matcher** — proposes patterns from a known family |
| **Solution** | the artifact you ship (code, document, decision, message) | generation + curation | **generator** — produces candidates, human curates |

The relationships are not just sequential — they are **space-defining**:

- Frame defines the approach space. (Once you've framed the problem, only certain approaches are coherent.)
- Approach defines the solution space. (Once you've chosen the approach, only certain solutions are reachable.)
- Wrong frame compounds the worst, because it silently cuts off the approaches that would have worked, which silently cuts off the solutions that would have moved anything.

Cost of error compounds upstream:

- Wrong solution: hours to fix. Iterate.
- Wrong approach: weeks to recover. You've built around the wrong method; rebuilding is expensive.
- Wrong frame: months or years lost. You'll perfect something that wasn't the right thing.

---

## Where the `approach` plugin lives

The `approach` plugin operates in the **approach space**. Each skill takes a framed problem (Problem Statement) and produces an *approach contract* — a structured representation of how the work will be decomposed and executed.

The plugin does not solve the problem (that's solution-space). It does not articulate the problem (that's frame-space). It produces a *plan with structure* — what shape the execution takes, who does what, how termination is decided, how failure is handled.

```
Problem        Frame                   Approach                Solution
   │             │                        │                       │
   │             │                        │                       │
   ▼             ▼                        ▼                       ▼
"the agency   "the bottleneck         "11 named patterns,      "code, docs,
margin        is brief→PPT             pick one (or composed   slides, configs,
oscillates"   compression"             tree); produce          decisions, sent
              (Problem Statement)      contract"               messages"
                                       (this plugin)
                                                               (orchestrator
                                                                executes from
                                                                the contract)
```

**Inputs to this plugin**: a Problem Statement (framed problem). Currently, composer's Phase 1 can extract one inline via Dialogue intake; long-term, this is delegated to the `/frame` plugin.

**Outputs from this plugin**: one or more contract files in `<contracts-root>/`. Composer outputs a composition tree (root + children). Individual skills output a single contract.

**Not in this plugin's scope**:
- Frame work (problem articulation, perception probes, lens-application, JTBD extraction, reframing) — handled by Dialogue mode in the interim, by the `/frame` plugin long-term
- Solution-space execution (code, content, decisions) — handled by the orchestrator, by Codex, by Agent sub-types, by external tools
- Cross-plugin orchestration / runtime routing — handled by Claude in conversation, or by the future runtime router

---

## The 11 approaches

Each approach is a distinct *structural pattern* of execution. They are not interchangeable; each fits a specific work shape.

| Approach | When it fits |
|---|---|
| **`approach:pipeline`** | Multi-stage with fixed sequential ordering and handoffs |
| **`approach:swarm`** | N parallel-independent slices + explicit reducer |
| **`approach:critic`** | Quality-critical work needing iteration against rubric (or symmetric debate) |
| **`approach:gated`** | Irreversible / blast-radius / shared-state steps needing human authorization |
| **`approach:contract`** | Broad / ambiguous / fake-progress-prone — verifiable slices + named verifiers |
| **`approach:loop`** | Scheduled-recurrence work (cron, interval, self-paced) across sessions |
| **`approach:event`** | Reactive trigger → fresh-session handler |
| **`approach:dialogue`** | Conversation IS the deliverable — requirements gathering, sensemaking |
| **`approach:search`** | Explore-evaluate-prune over a candidate space |
| **`approach:blackboard`** | Opportunistic multi-agent over a shared workspace |
| **`approach:one-shot`** | Degenerate shape — explicit "no decomposition needed" |

Plus the meta-skill:

| Meta | What it does |
|---|---|
| **`approach:composer`** | Takes a task, extracts (or consumes) a Problem Statement, maps to candidate approaches, composes nested tree, writes root + children contracts |

The composer is the apex of the family. For multi-shape tasks, invoke composer. For tasks that obviously fit one shape, invoke that shape directly.

---

## Composition

Approaches nest. A Pipeline whose one stage is itself a Critic. A Loop whose tick spawns a Swarm. A Contract whose verification spine wraps a Pipeline-with-Gated-deploy.

Nesting depth is capped at 2 by default in composer (Pipeline-with-Critic-stage is fine; Pipeline-with-Critic-whose-producer-is-Pipeline-with-Swarm-stage is opaque). Deeper requires explicit reason.

Hidden composition is forbidden — every nested approach must be named in the parent contract's `Compositions:` field. See [PRINCIPLES.md §6](./PRINCIPLES.md#6-composition-is-explicit--no-hidden-nesting).

---

## The frame-only invariant

Every skill in this plugin produces a contract and stops. None of them execute. This is the architectural separation that keeps framing and execution composable across sessions, across tools, across worker types.

See [PRINCIPLES.md §1](./PRINCIPLES.md#1-frame-onlynever-executes) for the full rationale.

---

## Worker types

Approach contracts assign workers per stage / slice / etc. The candidate worker types:

| Worker | Strength | Use when |
|---|---|---|
| **Claude inline** | direction-setting, judgment, orchestration, small reads/writes | the work needs Claude's reasoning in the current conversation context |
| **Codex (`codex exec`)** | long-context single-task, mechanical edits, deterministic refactors, deep analysis of large inputs | the stage is well-defined and benefits from deep depth-on-one-input |
| **Claude sub-Agent (model: opus)** | isolated exploration that shouldn't pollute the main context | the work needs Claude's reasoning but in its own conversation |
| **Codex `/adversarial-review`** | adversarial critique, second-opinion review | the work is judgment-bound and needs a fresh adversarial lens |
| **Human** | taste, business decision, credentialed approval, legal/medical/financial judgment, destructive side effects | the decision requires authority Claude/Codex can't replicate |

Default lean: **Claude for direction, Codex for execution**. Worker assignment is per-stage, not defaulted globally. See [PRINCIPLES.md §2](./PRINCIPLES.md#2-sub-agent-workers-always-use-opus) for the model-opus rule on sub-Agents.

---

## Why this plugin exists

Most people use Claude only in the solution space — "Claude, do this in this way" — already having chosen the frame and the approach alone, in their head, before they ever opened the conversation. The leverage of AI moves upstream when the *approach* itself is co-shaped rather than pre-decided alone.

This plugin exists to make that upstream move explicit and operational. Each skill encodes one structural pattern (Pipeline, Swarm, Critic, …) that has emerged across years of agent-orchestration practice. Naming the pattern is the smallest unit of clarification; producing the contract is the smallest unit of commitment.

The plugin's deepest claim is structural: **once the right approach is named, the solution space is dramatically narrowed and execution becomes a smaller, more legible problem.**
