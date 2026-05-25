---
name: critic
description: Critic approach framer for quality-critical work that needs adversarial review. ALWAYS invoke this skill when the user asks to peer-review, critique, adversarially-review, red-team, second-opinion, sanity-check, harden, or get a fresh take on the output of a task where quality is hard to verify deterministically — code review, writing review, security audit, architectural review, design critique, important draft, contract draft, strategic option evaluation, research synthesis review. Produces a Critic contract — producer worker, critic worker (typically Codex /adversarial-review or sub-Agent opus), critique rubric, max iterations, pass criteria, escalation when max hit. Supports asymmetric mode (worker + critic) and symmetric debate mode (peers + judge). Does not execute; the orchestrator follows. Do not just produce-and-ship work where quality matters — use this skill first to make the critique loop explicit. Skip for deterministically-verifiable work (use approach:contract), trivial single-shot work, or open-ended exploration (use approach:search).
# description-style: directive + negative constraint (Seleznov)
# rationale: Critic framing competes with default Claude behavior (would otherwise just produce and ship without explicit critique pass); directive style forces the load-bearing rubric + max-iteration constraints.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (knowledge work): "Review this strategy doc with adversarial intent — find the assumptions that won't survive client pushback"
2. SHOULD activate (coding): "Review this migration script with adversarial intent — find what I missed"
3. SHOULD NOT activate: "Run the tests and tell me if they pass" (deterministic check, not Critic)
4. BOUNDARY: "Make sure this auth change is safe" (Critic if "safe" is judgment-bound; Contract if "safe" reduces to concrete tests — skill must disambiguate via rubric test)
-->

# critic

Frames quality-critical work as a Critic contract — producer, critic, rubric, iteration policy, escalation. Two modes: asymmetric (worker + critic) and symmetric debate (peers + judge). One of the 11 named approaches in the `approach` plugin; frame-only, never executes.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- Output quality matters more than speed.
- Quality is hard to verify deterministically — no green-test signal.
- Common LLM failure mode for this task is over-confidence (claims done before it actually works).
- Composing with other approaches: a Pipeline stage that is itself a Critic, or a Swarm where the reducer is a Critic.

**Examples — knowledge work + content:**
- Writing critique, essay / argument review, research paper critique
- Thesis defense pre-mortem, pitch deck review, contract drafts
- Important Slack / email drafts where one phrase changes the reception
- Strategic option evaluation, consulting deliverable review
- Synthesis review (does the synthesis defensibly follow from the corpus?)

**Examples — code / ops:**
- Code review, security audit, architectural review, threat modeling
- Migration scripts, deployment plans
- API design review

## When NOT to Use

- Quality is deterministically checkable (tests pass / lint clean / type-check) — use `approach:contract` with the test as verifier.
- Quality doesn't really matter for this artifact (throwaway, scratch work) — just do it once.
- No way to specify what "good" looks like — not Critic, probably `approach:search` (explore options first, then critique).
- The critic would just be a yes-machine (rubber stamp, no real signal) — wasted overhead.
- Single trivial task — just do it.

## Process

The skill outputs a **Critic contract** — a structured document, not executed work. Frame, write, hand back.

### 1. Confirm the shape fits and pick mode

Confirm Critic is right:
- Can you state a rubric of specific criteria the output must satisfy? If not → not Critic, frame the rubric first.
- Is the rubric LLM-judgeable (vs deterministic)? If deterministic → Contract.
- Is there a real failure mode (not rubber-stamping)? If not → skip the apparatus.

Pick mode:
- **Asymmetric (worker + critic)** — one produces, one critiques, iterate. Default. Best for code review, single-author critique, hardening passes, single-output review.
- **Symmetric (peers + judge)** — N peers produce competing outputs or arguments, judge picks. Best for genuinely contested decisions (option A vs option B), red-team vs blue-team, debate-shaped problems, strategic alternatives.

### 2. Pick workers

**Producer** (asymmetric) or **peers** (symmetric):
- Claude inline (you), or
- Codex (`codex exec`) for long deterministic production, or
- sub-Agent **(model: opus)** for isolated exploration.

**Critic** (asymmetric) or **judge** (symmetric):
- **Codex `/adversarial-review`** is the canonical critic — different model + adversarial framing built in. Strong default.
- sub-Agent **(model: opus)** with red-team framing in the prompt.
- Human — when critique requires taste or judgment Claude/Codex can't replicate.

Never use the same model + same prompt for producer and critic — produces fake critique. Distinguish by model, by prompt framing, or both.

### 3. Define the critique rubric

**Load-bearing.** A Critic without a concrete rubric becomes a vibes-check — the critic says "looks good" and the loop never bites.

Each rubric criterion must be:
- **Specific** — "the function handles N=0 correctly" not "the code is correct"; "the recommendation cites specific client-facing language" not "the strategy is clear."
- **LLM-judgeable** — the critic can produce a clear pass/fail per criterion from the output alone.
- **Substance over style** — prioritize correctness/safety/coverage over wording/formatting.

For asymmetric: 3-7 criteria. Fewer → too coarse. More → critic loses focus.
For symmetric: list the criteria the judge weighs when comparing peer outputs.

### 4. Define iteration policy (asymmetric) or rounds (symmetric)

**Asymmetric**:
- **Max iterations** — usually 3. Beyond that, marginal gain is low and cost climbs.
- **Pass criteria** — all rubric criteria pass, OR critic says "ship it" with no high-severity findings.
- **On max-without-pass** — escalate to human with a summary of unresolved critic findings. Never silently ship.

**Symmetric**:
- **Rounds** — usually 1 (simultaneous + judge picks) or 2-3 (peers see each other's args and respond).
- **Termination** — judge picks one OR human picks if judge can't decide.

### 5. Apply the Direct-Use Rule

If the artifact under review can be exercised in its real intended surface (deployed, opened, run), the critic should review it *after* a real-surface check, not before. A migration script reviewed without dry-run output is critique against text; reviewed after dry-run is critique against actual behavior. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible).

### 6. Output the contract

Write to `<contracts-root>/critic-<slug>.md` (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)).

**Asymmetric template:**

```markdown
# Critic contract: <task-name>

## Mode
asymmetric (worker + critic)

## Producer
- Worker: <claude | codex | subagent:<type> (model: opus)>
- Input: <input>
- Initial output: <artifact / path>

## Critic
- Worker: <codex /adversarial-review | subagent:<critic-type> (model: opus)>
- Rubric:
  1. <specific criterion>
  2. <specific criterion>
  3. <specific criterion>
- Pass criteria: <all rubric pass | no high-severity findings | …>

## Iteration
- Max: <usually 3>
- On pass: ship latest version
- On max-without-pass: <escalation — human review | ship-with-caveats | halt + report>

## Compositions
- <none | producer is approach:pipeline | …>

## Assumptions
<inferences during framing — rubric completeness, severity definitions — or "none">

## Extraction-confidence
<low | medium | high>
```

**Symmetric template:**

```markdown
# Critic contract: <task-name> (debate mode)

## Mode
symmetric (peers + judge)

## Peers
- N: <usually 2-3>
- Worker: subagent:<peer-type> (model: opus)
- Per-peer framing:
  - Peer 1: <stance / angle>
  - Peer 2: <stance / angle>

## Judge
- Worker: <subagent:<judge-type> (model: opus) | human>
- Decision criteria:
  1. <criterion>
  2. <criterion>

## Rounds
- N: <1 = simultaneous + judge picks | 2-3 = peers respond>
- Termination: judge picks | human picks if judge stalemates

## Compositions
- <none | each peer is approach:pipeline | …>

## Assumptions
<inferences during framing — peer-stance coverage, judgability — or "none">

## Extraction-confidence
<low | medium | high>
```

Then surface the path to the orchestrator.

### 7. Stop

The skill is framing-only. Do not start the producer. Do not invoke `/adversarial-review`. The orchestrator decides when and whether to run.

## Anti-Patterns

- **Rubber-stamp critic.** The critic says "looks good" every time and the loop never bites. Symptom of a vibes-only rubric. Fix: concrete LLM-judgeable criteria per step 3.
- **Same model + same prompt for producer and critic.** Produces fake critique — the critic sees no flaws because it would have made the same choices. Fix: different model (Codex critiques Claude) OR explicit adversarial/red-team framing in the critic prompt.
- **No max iteration.** Infinite revise-recritique loops burn tokens. Always cap (default 3).
- **Critic critiques style, not substance.** Wording nitpicks instead of correctness/safety. Fix: rubric explicitly prioritizes substance; style criteria deprioritized or excluded.
- **No escalation when max hits without pass.** Silently shipping after N iterations is worse than not having Critic at all — it manufactures false confidence. Always define the escalation.
- **Critic for deterministically-checkable work.** Tests pass / lint clean is Contract, not Critic. Critic is for judgment-bound quality.
- **Symmetric debate with peers given identical prompts.** Produces N copies of the same output, judge picks one at random. Symmetric mode requires peers to be positioned differently (stance, angle, persona).
- **Critic without real-surface input.** Critiquing a migration script's text instead of the dry-run output, critiquing a deck's slides instead of the rendered PDF — missed Direct-Use Rule.

## Rules

- The skill always outputs a single Markdown contract file; it never executes the producer or critic.
- Every Critic contract must declare: mode, producer/peers, critic/judge, rubric, iteration policy, escalation, Assumptions, Extraction-confidence.
- The rubric is non-optional and must be substance-prioritized (correctness/safety over style/formatting).
- Producer and critic must differ — different model, different prompt framing, or both. Same model + same prompt is rejected.
- When a producer, critic, peer, or judge worker is a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- Codex `/adversarial-review` is the default critic when no specific reason to prefer another.
- Iteration cap defaults to 3 for asymmetric; rounds default to 1 for symmetric.
- If the task doesn't fit Critic, recommend the right approach and stop — don't force-fit. When recommending a sibling, check `<available_skills>` first; if not installed, describe inline.
- Contract path: `<contracts-root>/critic-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/critic-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:gated`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:event`, `approach:dialogue`, `approach:search`, `approach:blackboard`. Related external skills: `/loop` (Loop scheduling executor), `codex:adversarial-review` (canonical asymmetric critic worker).
