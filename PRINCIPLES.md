# PRINCIPLES

Shared rules across every skill in the `approach` plugin. Each SKILL.md references this file rather than restating the rules. If a rule needs to change, change it here.

---

## 1. Frame-only — never executes

Every approach skill produces a **contract document**. It never executes the work the contract describes. Execution is the orchestrator's job — the user, Claude in a follow-up turn, the future runtime router, or whichever sub-agent the contract assigns.

The architectural reason: framing and execution have different time horizons, different review surfaces, and different failure modes. Conflating them couples the cognitive work of structuring to the runtime decisions of running, which makes contracts harder to audit, harder to compose, and harder to hand off cleanly between sessions.

The practical reason: a contract written and stopped is a checkpoint. A contract written and executed in the same turn is a fait accompli. The user should always have a moment to read the framing before the work starts.

**Operational consequence:** no skill in this plugin spawns Agents, invokes `codex exec`, edits `settings.json`, calls `/loop` or `/schedule`, fires gates, or otherwise takes runtime action. Skills produce markdown contracts and stop.

---

## 2. Sub-Agent workers always use Opus

When a contract assigns a Claude sub-Agent (via the `Agent` tool or equivalent) as the worker for a stage, slice, producer, critic, peer, judge, evaluator, specialist, executor, or handler, the contract **MUST** specify `model: opus`.

Never Sonnet. Never Haiku. No exceptions — including for parallel breadth-research fan-out, code review agents, light scoring tasks, or "this is a small slice" arguments.

The reasoning: model capability dominates token cost on judgment-bound work, which is the entire surface this plugin exists for. A Sonnet sub-Agent saving a few cents per slice that produces a worse contract output costs more downstream than the savings.

**Operational consequence:** every contract template's worker assignment block includes `(model: opus)` next to sub-agent workers. If you find yourself writing `subagent:<type>` without that suffix, you've missed the rule.

---

## 3. Assumptions + extraction-confidence — surface uncertainty, don't hide it

Every contract template has an `Assumptions:` field, even when the field's content is empty or just `none`. When the framer infers anything not stated explicitly by the user — about scope, constraints, success criteria, expected inputs, worker capability, downstream consumers — the inference goes in `Assumptions:`, not in the body.

Composer additionally surfaces an `extraction-confidence: low | medium | high` marker on the Problem Statement file. Other skills should adopt the same marker on their contracts when intake was abbreviated:

- **high** — every required field was stated or directly inferable
- **medium** — some inference, no blocking unknowns
- **low** — significant inference, contract is provisional, revisit on first execution failure

**Operational consequence:** when contracts later fail in execution, low-confidence extractions are the first place to revisit. The marker is the audit trail.

---

## 4. Direct-Use Rule — exercise the real surface as early as possible

If the thing being built can be used through its real intended surface, make that surface part of the loop **as early as possible**. Do not wait until the end to install, connect, run, deploy, open, import, or exercise it if doing so can reveal real blockers sooner.

Real intended surfaces include:
- An MCP server installed and called through the client that will use it
- A web app opened in the browser and driven through its UI
- A CLI installed or run by its documented command
- An API called over HTTP against local, staging, or production endpoints
- A data pipeline run against representative fixtures or approved real data
- A plugin / automation / extension loaded into the host that will consume it

When a direct-use path exists, the contract should include an early stage / slice / handler step that exercises it. If it blocks on credentials, approval, paid access, sensitive data, 2FA, or another human-only gate, record the blocker explicitly and continue with local/deterministic work while waiting. Return to the direct-use surface as soon as the gate clears.

Lower-level tests, mocks, static checks, and local fixtures are fine for progress, but they do not replace the real surface when the real surface is available and safe to exercise.

**Originally articulated in `approach:contract`**, the rule applies to every shape — Pipeline stages, Swarm slices, Critic iterations, Event handlers, Loop ticks, Search candidate evaluations, Blackboard specialist contributions. Each skill references this principle rather than re-stating it.

---

## 5. Contracts-root resolution

Contract files land at `<contracts-root>/<shape>-<slug>.md`, with `<contracts-root>` resolved in this order (first match wins):

1. **User-specified path** — explicit override at invocation
2. **Task-implied project folder** — e.g., a task referencing `~/work/myapp` writes contracts there
3. **Current working directory if it is a project** — has `.git/` or `CLAUDE.md` → `<cwd>/.claude/contracts/`
4. **Fallback** — `/tmp/approach-contracts/` (only when no project context exists)

Contracts in `.claude/contracts/` can be committed or git-ignored per project preference. The `/tmp/` fallback exists so the skill never blocks on missing context; if it lands there often, it's a signal to either provide explicit paths or land the work in a real project.

**Slug** is kebab-case derived from the task name (`refactor-auth-module`, `synthesize-research-corpus`).

**Composer's contract paths** are slightly different — the composition root + per-shape children — see composer's SKILL.md.

---

## 6. Composition is explicit — no hidden nesting

When a contract's stage / slice / producer / handler is itself an approach (a Pipeline stage that's a Critic, a Swarm where each slice is a Pipeline, an Event handler that ends in a Gated step), the contract **MUST** name the nested approach explicitly in its `Compositions:` field.

The reasoning: hidden composition makes contracts lie about the work. A Pipeline contract that says "stage 3: review" but where stage 3 is actually a Critic-loop-with-3-iterations hides where the actual time and risk live. Reviewers (human or LLM) can only audit what's visible.

**Operational consequence:** every contract template has a `Compositions:` field. If the field reads `none`, that's a claim — defend it. If a stage involves any iteration, parallelism, gating, or hand-off that isn't trivial, name the nested approach.

Composer's recursion goes further: every level of nesting belongs in the composition root's `Nested shapes` table. Deeply-nested children must surface their own nesting in the `Compositions:` field of their child contract.

---

## 7. Knowledge work is co-equal with coding work

The shapes in this plugin work for both code-shaped tasks (refactor, build, deploy, test, audit) and knowledge-work-shaped tasks (research, synthesis, document analysis, multi-source claim verification, framing exploration, consulting prep).

Examples in each SKILL.md are deliberately balanced — knowledge-work examples are not appendices to coding examples; they are the primary use case for several shapes (notably Dialogue, Search, Blackboard, Contract). The framing language ("stages," "slices," "iterations," "candidates," "specialists") is task-agnostic by design.

**Operational consequence:** when proposing a shape, consider both lenses. A consulting deliverable can be Pipeline-shaped; a research corpus review can be Swarm-shaped; a strategy comparison can be Search-shaped. Don't default to thinking the user means code unless they've said so.

---

## 8. Recommend sibling — never force-fit

If the user's task doesn't fit the shape the active skill represents, the skill **recommends the right sibling shape and stops**. It never force-fits.

When recommending a sibling, check `<available_skills>` first to see if the sibling skill is installed in the current session. If yes, recommend by skill name (`approach:swarm`). If no, describe the shape inline so the user gets the structural advice even without the skill installed.

**Operational consequence:** every SKILL.md's "When NOT to Use" section names the alternative shape. Each Process section has an early "Confirm shape fits" step that can exit cleanly if the shape doesn't apply.

---

## 9. Architectural location — the approach layer

The `approach` plugin operates in the **approach space** of the problem → frame → approach → solution cascade (see [ARCHITECTURE.md](./ARCHITECTURE.md) for the full model). It takes a *framed problem* (Problem Statement) as input and produces an *approach contract* (structured execution pattern) as output. The contract is then handed to the orchestrator for solution-space execution.

Composer's Phase 1 currently does *frame* work (problem extraction) as a transitional inlining. When the future `/frame` plugin ships, Phase 1 will delegate to `/frame:composer` or equivalent, and `approach:composer` will become a pure approach-space tool that consumes a Problem Statement.

**Operational consequence:** every skill in this plugin can assume the problem has been framed (or the framer is willing to do quick intake inline). No skill in this plugin attempts to solve "the user doesn't know what they're trying to do" — that's a frame-space problem, handled by Dialogue mode upstream or (in the future) by /frame.
