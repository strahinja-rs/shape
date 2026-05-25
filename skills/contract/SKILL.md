---
name: contract
description: Contract approach framer (Loop Contract authoring) for broad, ambiguous, high-risk, iterative, or fake-progress-prone tasks — produces verifiable slices, named verifiers, stop conditions, integrity rules. ALWAYS invoke this skill when the user asks to plan, scope, structure, or de-risk a multi-step task; when work is open-ended, architecture-heavy, research-heavy, security-sensitive, hard to test, or easy to fake-pass; when phrasing like "make sure it actually works", "verify each step", "don't fake it", or "let's break this down" appears; or when the user asks for a Loop Contract. Skip for small, well-scoped one-shot tasks (use approach:one-shot) unless explicitly requested. Do not just start executing a broad task or hand back a vague checklist. Use this skill first to split verifiable slices from human-only decisions and define how each will be verified.
# description-style: directive + negative constraint (Seleznov)
# rationale: Contract framing competes with default Claude behavior (would otherwise just start executing a broad task or write a flat plan). Directive style forces the structured-contract route — verifiable slices, named verifiers, integrity rules. Works equally for coding and knowledge-work tasks (research synthesis, document analysis, multi-source claim verification).
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (coding): "Refactor the auth system to support multiple providers, make sure each piece is actually verified"
2. SHOULD activate (knowledge work): "Synthesize the literature on X across these 12 papers — make sure no claim is unsourced"
3. SHOULD NOT activate: "Fix the typo in line 42 of foo.py"
4. BOUNDARY: "Help me build an MCP server end-to-end" (broad + risk of fake-progress → should activate, then clarify scope)
-->

# contract

Turn a broad, ambiguous, or high-risk task into an evidence-driven Loop Contract before doing substantive work. Extract what can be made verifiable, isolate what needs human judgment, define the smallest input that unblocks the next iteration, then execute against the contract. One of the 11 named approaches in the `approach` plugin.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, **Direct-Use Rule originally articulated here, now elevated to a shared principle**, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- The task is broad, ambiguous, multi-step, or open-ended.
- Risk of regression, security, privacy, or destructive side effects.
- Easy to claim success without evidence (research, refactors, integrations, ingest pipelines, UI quality work, synthesis claims).
- Iterative work where each pass needs its own pass/fail signal.
- The user asks for "a plan", "let's structure this", "make sure it actually works", "verify each step", "don't fake it", "we keep going in circles".
- The user explicitly invokes the skill (`/approach:contract` or "make a Loop Contract").
- Mixed work (code + research + ops + content) where a flat checklist would mask weak verification.

## When NOT to Use

- Trivial one-shot edits (rename, single-line bugfix, format pass) unless the user explicitly asks.
- Read-only questions and lookups (use the right tool, return the answer).
- Tasks already covered by a more specialized skill (`/skill:create`, `/eval`, `/research`). Defer to those; if they need a verification spine, they can call this skill themselves.
- When the user has already produced a contract and just wants execution. Use the contract as the operating plan; do not re-derive it.

## Process

### 1. Decide if a Loop Contract is warranted

Before writing the contract, do a 30-second gate:

- Is the task large enough that flat execution would lose track of evidence? If no, skip the contract and do the work.
- Is there a real risk of fake-progress (silent passes, hallucinated verifiers, weakened tests, reviewer-only sign-off)? If yes, contract is warranted.
- Will the work span multiple sessions, multiple files, or multiple constraint domains? If yes, contract is warranted.

If the user explicitly asked for a contract, skip the gate and proceed.

### 2. Apply the Direct-Use Rule (load-bearing for Contract specifically)

The Direct-Use Rule (originally named in this skill, now a [shared principle in PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible)) applies with particular force to Contract because Contract often wraps work that *could* be verified through mocks indefinitely. The rule's intent is to prevent exactly that.

If the thing being built can be used through its real intended surface, make that surface part of the loop **as early as possible**. Do not wait until the end to install, connect, run, deploy, open, import, or exercise it if doing so can reveal real blockers sooner.

Examples of real intended surfaces:

- An MCP server installed as an actual MCP/connector and called through the client that will use it.
- A web app opened in the browser and driven through its UI (via Playwright, Chrome MCP, or `mcp__Claude_Preview__*`).
- A CLI installed or run by its documented command.
- An API called over HTTP against local, staging, or production endpoints.
- A data pipeline run against representative fixtures or approved real data.
- A plugin, automation, extension, or package loaded into the host that will consume it.
- A research synthesis tested against the real corpus, not against a summary of the corpus.

When a direct-use path exists:

1. Add an early verifiable slice for installing/connecting/running the real surface.
2. Run the smallest smoke test through that surface before or alongside code changes.
3. If it blocks on credentials, approval, paid access, sensitive data, 2FA, or another human-only gate, record that blocker explicitly and continue with local/deterministic slices while waiting.
4. Return to the direct-use smoke as soon as the human gate is cleared.
5. Keep direct-use checks in the loop after changes, not just as a final validation.

Lower-level tests, mocks, static checks, and local fixtures are fine for progress, but they do not replace the real surface when the real surface is available and safe to exercise.

### 3. Clarify only blockers

Before writing the contract:

1. Extract the clear task, constraints, files, tools, acceptance signals, and implied risks from context (conversation, CLAUDE.md, recent files, Linear/PR state).
2. Separate unknowns into:
   - **Blocking unknowns** that could make the work wrong or unsafe.
   - **Non-blocking unknowns** that can be handled by a conservative assumption.
3. Ask only blocking questions, batched in one message. Keep to three when possible; if more are truly essential, list all and mark which must be answered before the first loop iteration.
4. If no blocking question is needed, proceed and record the assumptions in the contract.

### 4. Write the Loop Contract

```markdown
**Loop Contract**
Objective:
Context:
Constraints:
Assumptions:
Extraction-confidence: <low | medium | high>

Verifiable slices:
- Slice:
  Success predicate:
  Verifier:
  Evidence to capture:

Human-only decisions:
- Decision:
  Why human input is needed:
  Smallest useful answer:

Execution loop:
1. Inspect:
2. Change:
3. Verify:
4. Review:
5. Decide:

Stop conditions:
- Pass:
- Blocked:
- Risk:
- Budget:
- No-progress:

Final report:
- What changed:
- Evidence:
- Residual risk:
- Human decisions still needed:
```

For small tasks, compress the template but keep objective, verifiable slices, verifier, stop conditions, and final evidence.

**Persist the contract.** If the work will span more than one session or compaction is likely, write the contract to a file (e.g. `loop-contract.md` next to the work, or at `<contracts-root>/contract-<slug>.md` per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)) and create matching items via `TaskCreate` so the slices survive context loss. The first 5K tokens of context survive compaction; the file does not unless re-read.

### 5. Verifier selection

Prefer verifiers in this order (ranked by signal quality):

1. **Deterministic commands** — tests, typecheck, lint, build, schema validation, migrations, benchmarks, format checks.
2. **Behavioral checks** — focused smoke tests, CLI/API calls, browser interactions, screenshots, generated fixture comparisons.
3. **Structural checks** — file presence, diff inspection, dependency graph checks, link checks, required sections, JSON/YAML/schema validity.
4. **Research/content checks** — source coverage, citation traceability, recency checks, contradiction checks, quote limits, completeness against the request.
5. **Reviewer checks** — Claude self-review, subagent review via the `Agent` tool, or external CLI review (Codex, GPT-5.4) against a rubric.
6. **Human gates** — preference, business decision, credentialed approval, legal/medical/financial judgment, destructive side effects.

Human gates dominate reviewer checks when the decision involves legal, medical, financial, credentialed, destructive, privacy-sensitive, or business-ownership judgment.

When the Direct-Use Rule applies, treat the real intended surface as the highest-value behavioral verifier after the relevant deterministic checks. If deterministic checks can pass while the real surface is unusable, the task is not complete.

Read `references/verifier-patterns.md` when the verifier is not obvious, the work is non-code, or the task mixes code, research, UI, documents, data, or operations.

### 6. Execution loop

1. Inspect the current state and choose the next smallest verifiable slice.
2. If a direct-use surface exists, install/connect/run it early and perform the smallest safe smoke test.
3. If direct use blocks on human-only input, record the gate and continue with independent slices.
4. Make the smallest change that can move the current slice toward passing.
5. Run the narrowest relevant verifier immediately.
6. Diagnose failures from evidence, not guesses.
7. Repair, then rerun the same verifier.
8. Broaden verification only after the narrow verifier passes, including returning to the direct-use surface when available.
9. Review the diff or artifact against the Loop Contract.
10. Decide pass, continue, ask, or stop based on the contract.

Keep a short progress ledger during execution. Use `TaskCreate` / `TaskUpdate` for the slices themselves, and a markdown block for the per-attempt detail:

```markdown
Attempt:
Slice:
Change:
Verifier:
Result:
Diagnosis:
Next:
```

If the same failure recurs after a repair, change strategy per the No-Progress Rule: narrow the slice, add instrumentation, read source/docs, ask the smallest blocking question, or escalate to a reviewer with the failure evidence. Stop and report when the same slice fails repeatedly without new evidence, when a verifier cannot be made available, when the next action requires credentials or destructive permission, or when the task reaches a human-only decision.

### 7. Reviewers

Use reviewers to find missed risks **after evidence exists**. A reviewer is not a substitute for running the verifier.

- **Claude self-review.** For code or artifact changes before finalizing. Review for bugs, regressions, missing tests, security/privacy issues, broken assumptions, and weak evidence. `/fresh-eyes` works well here.
- **Subagent review via the `Agent` tool.** Use `subagent_type: "general-purpose"` for independent bounded checks, `subagent_type: "Explore"` for read-only investigation. Give the subagent the task, the Loop Contract, the diff or artifact, and the verifier evidence. Do not give them your desired answer.
- **External CLI review.** When the task is high-risk, architecture-heavy, security-sensitive, ambiguous, large, or near final delivery, hand to a different model:
  - `/codex:rescue` or `/codex:review` for engineering-task adversarial review.
  - `/gpt54` for GPT-5.4 Extended Thinking on deep reasoning tasks.
  Read `references/reviewer-patterns.md` before invoking an external reviewer or writing a review prompt.
- **Plan agent for upfront contract drafting.** For multi-component tasks where the initial Loop Contract draft itself is non-trivial, delegate to `subagent_type: "Plan"` and then add the verifier columns afterward.
- **`/gut-check` for human-invoked lightweight self-assessment between iterations.** Different from formal reviewer passes — this is the user reaching for the canonical 10-question plain-language self-report (understanding, telos, progress, definition-of-done, keep-vs-change). Useful when the contract has been running for a while and the user wants to surface drift, fake progress, or a missing definition-of-done before more time is wasted. Do not auto-invoke; the user reaches for it deliberately.

### 8. Final report

Report only the highest-signal facts:

- The Loop Contract result if the user asked for rewrite-only.
- The changes made if execution occurred.
- The exact verifiers run and their outcomes.
- Any verifiers that were not run and why.
- Residual risks and the smallest human decisions still needed.

Label outcomes as **complete**, **partially verified**, **blocked**, or **needs human decision**.

## Anti-Patterns

Skill-specific failure modes for `approach:contract`:

- **Contract-then-stop.** Producing a beautiful Loop Contract and stopping there when the user asked for execution. The contract is the operating plan, not the deliverable. If the user only asked to rewrite the task into a contract, produce it and stop. If they asked to execute, the contract should be visible but the loop should run.
- **Skipping the Direct-Use Rule.** Building and "verifying" with only mocks, fixtures, or local unit tests when the real surface (deployed endpoint, installed MCP, real browser, real client) is available. The real surface is a behavioral verifier, not an optional final step.
- **Verifier weakening to make a slice pass.** Loosening assertions, deleting failing tests, hardcoding the visible example, skipping checks, or downgrading "behavioral" to "structural" so the slice goes green. Sign of the integrity rules being violated. If a verifier is too strict for the slice, change the slice, not the verifier.
- **Inventing evidence in the progress ledger.** Writing "verifier passed" without actually running it, narrating success on a tool call that failed, hallucinating output. The ledger is auditable; reviewers will catch it.
- **Reviewer-only sign-off.** Marking a slice complete because a reviewer (Claude self-review, subagent, Codex, GPT-5.4) approved it when the deterministic or behavioral verifier was not run or failed. Reviewers find missed risks; they do not replace verifiers.
- **Asking too many clarifying questions.** Over-clarifying (5+ questions for a medium task) instead of recording conservative assumptions in the contract and proceeding. Only ask blocking questions. Non-blocking unknowns become Assumptions.
- **No-progress denial.** Retrying the same failing slice with the same strategy multiple times in a row. After two consecutive failures without new evidence, change strategy (narrow the slice, add instrumentation, read docs, ask the smallest blocking question, escalate to a reviewer). Do not just keep trying.
- **Contract not persisted across compaction.** Building a multi-session Loop Contract entirely in conversation memory. When compaction fires at ~83.5% context, the contract and ledger disappear. Persist both to disk or to `TaskCreate` items.

## Rules (integrity)

- **Do not weaken verifiers to make the loop pass.** No deleting failing tests, no loosening assertions, no skipping important checks, no hardcoding visible examples unless the user explicitly asked and the contract records it.
- **Do not mark a task complete on reviewer approval alone** if deterministic or behavioral verifiers are failing or unrun.
- **Do not hide uncertainty.** Always label outcomes as complete, partially verified, blocked, or needs human decision.
- **Do not fake the ledger.** Every entry must correspond to a real tool call with real output. No invented evidence.
- **Do not collapse "full autonomy is impossible" into a stopping point.** Instead, extract the parts that can be made verifiable, isolate the parts that need human judgment, and define the smallest human input that would unblock the next loop.
- **Always run the narrow verifier first.** Broaden only after the narrow check passes.
- **Always return to the direct-use surface** when a human gate clears. Do not declare success against mocks if the real surface is now available.
- **Sub-Agent workers always use Opus.** When the contract assigns a Claude sub-Agent to a slice, verifier, or reviewer, the contract MUST specify `model: opus` per [PRINCIPLES.md §2](../../PRINCIPLES.md#2-sub-agent-workers-always-use-opus).
- **Assumptions and Extraction-confidence are non-optional fields** per [PRINCIPLES.md §3](../../PRINCIPLES.md#3-assumptions--extraction-confidence--surface-uncertainty-dont-hide-it).

## Key Files

- `references/verifier-patterns.md` — verifier patterns by task type (code, frontend, research, documents, data, ops). Read when the verifier is not obvious.
- `references/reviewer-patterns.md` — reviewer prompt shapes for Claude self-review, subagent review, and external CLI review (Codex, GPT-5.4). Read before invoking an external reviewer.
- Output: typically `loop-contract.md` next to the work, or `<contracts-root>/contract-<slug>.md` per PRINCIPLES.md §5 for project-aware paths.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:loop`, `approach:one-shot`, `approach:event`, `approach:dialogue`, `approach:search`, `approach:blackboard`.
