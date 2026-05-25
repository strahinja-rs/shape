---
name: gated
description: Gated approach framer for flows with irreversible, blast-radius, or shared-state steps that need human authorization. ALWAYS invoke this skill when the user asks to plan, structure, or scope work involving deploys, sends, payments, transfers, public posts, schema migrations, dropping data, force-pushing, transmitting invoices, posting legal/financial documents, or any action that cannot be cleanly undone. Produces a Gated contract — the gated action, preview shown to human, gate question, granularity (per-instance vs batch), on-approve and on-reject behavior, idempotency/cancellability after gate. Does not execute; the orchestrator follows. Do not auto-fire irreversible actions even when confident — use this skill first to make the gate explicit. Skip for fully-reversible local work, trivial confirmations already covered by tool permissions, or actions where the human can't meaningfully evaluate the gate (consider approach:critic upstream first).
# description-style: directive + negative constraint (Seleznov)
# rationale: Gated framing competes with default Claude behavior (would otherwise auto-fire when confident); directive style forces the explicit gate at irreversible steps. The Cato Networks-style single-consent attack vector is real — autonomy decisions on side-effectful work need to be slow and explicit.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (coding): "Plan the production deploy: build, test, push to prod"
2. SHOULD activate (knowledge work / life ops): "Send these 5 client emails after I review them — none of them auto-send"
3. SHOULD NOT activate: "Refactor this function and run the tests" (reversible local work, no irreversible step)
4. BOUNDARY: "Send these 5 reminder emails to clients" (each email is irreversible — per-instance gate? batch gate? skill must surface the granularity decision)
-->

# gated

Frames a flow with one or more irreversible / blast-radius / shared-state steps as a Gated contract — the gated action, the preview shown to the human, the gate question, granularity, approve/reject behavior, cancellability. One of the 11 named approaches in the `approach` plugin; frame-only, never executes.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- Action is irreversible — deploy to prod, send email, transmit invoice on SEF, transfer money, post to public Slack, publish content, force-push, drop table, commit to main.
- Blast radius extends beyond local machine — shared infra, external API with side effects, third-party consumers downstream.
- Mistake cost is high — legal documents, financial transactions, anything bearing the user's name.
- "Once done, can't undo" — or only via expensive recovery (storno > cancel > undo).
- Composing with other approaches: a Pipeline stage that is itself a Gated gate, a Swarm where each slice is Gated, a Critic followed by a Gated post-Critic action.

**Examples — knowledge work / life ops:**
- Transmitting a legal invoice on a national e-invoicing system
- Sending a client proposal after review
- Posting a public manifesto or thought-leadership piece
- Mailing a physical document
- Submitting a regulatory filing

**Examples — code / ops:**
- Production deploys, schema migrations, dropping tables
- Force-pushing to main, committing to a protected branch
- API calls with billing side effects
- Cron-firing of recurring side-effectful jobs

## When NOT to Use

- Reversible local work (file edits, feature-branch commits, scratch work) — a gate is friction without value.
- Trivial confirmations already covered by tool-level permission prompts (Claude Code already prompts for risky Bash). Don't double-gate.
- Actions where the human can't meaningfully evaluate the gate — e.g., approving a 10K-line diff at 2 AM. Better to add upstream `approach:critic` (or `approach:contract`) so the human is approving a reviewed artifact, not raw output.
- One-shot work with no irreversible step — just do it.

## Process

The skill outputs a **Gated contract** — a structured document, not executed work. Frame, write, hand back.

### 1. Identify the gated action(s)

List every step in the flow that is:
- Irreversible (action has side effects outside the local machine state, or local state changes are expensive to undo).
- Blast-radius (effect reaches shared systems, other people, external services).
- Hard to evaluate after-the-fact (you can't easily tell if it went wrong by inspecting state).

These are the gate points. A Gated flow can have one gate (single irreversible step) or multiple gates (multi-step flow with several risky points). Map them all.

### 2. Define the preview per gate

**Load-bearing.** The gate is only as good as the preview. Bad preview = false confidence — user clicks yes without actually understanding.

Each preview must:
- Show the **exact** action that will execute, not a summary. The bytes hitting the wire, the SQL that runs, the PDF body, the email recipient + subject + body.
- Surface the dangerous parts prominently (recipient, amount, target environment, scope of destruction).
- Avoid truncation that hides risk — never `…` on the dangerous bit.

If the action has many parameters, list them all. If the action triggers a chain (one call → downstream effects), name the chain so the user knows what they're approving.

**Direct-Use Rule applies:** the preview should be derived from the *real* action that will fire, not from a summary or template. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible).

### 3. Define the gate question

- **Binary** — yes/no. Default. Cleanest.
- **Three-state** — approve / reject / edit. Use when the user might want to tweak the action before firing (common for drafts).
- **Annotated** — yes/no with required reason. Use for high-stakes actions where post-hoc audit matters.

State the question literally as the human will see it. No ambiguity.

### 4. Define granularity

Per-instance vs batch:

- **Per-instance** — each item gets its own gate. Safer. Default for legal/financial. Current `/sef-inbox` pattern.
- **Batch** — one approval covers N items. Saves time. Only acceptable when the user genuinely can absorb the full batch contents in the preview.

Default to per-instance for irreversible legal/financial actions. Batch is opt-in and requires the preview to show all N items, not a summary.

**Never assume "once-confirmed = always-confirmed" across a batch or session** — explicit partnership-level rule. Re-gate per session, per batch boundary.

### 5. Define approve/reject behavior

- **On approve**: execute the previewed action exactly. No deviation. If the action needs to differ from the preview, re-gate.
- **On reject**: halt + report what was rejected; do not auto-retry; surface alternatives only if asked.
- **On edit (if three-state)**: capture the edit, re-preview, re-gate. The edited version is what runs.

### 6. Define cancellability after gate

Once approved, can the user still abort mid-execution?

- **Fire-and-forget** — HTTP request that completes synchronously; no abort once started. Document this explicitly.
- **Cancellable window** — small delay between approve and execute (e.g., 5-second "click to cancel"); useful for catastrophic-mistake recovery.
- **Storno/undo path** — execution completes but a reversal action exists (e.g., SEF storno for an issued invoice). Document the reversal cost.

State which model applies. Users deserve to know if "approve" means "fired immediately" vs "5-second window" vs "fired, but reversible via X."

### 7. Output the contract

Write to `<contracts-root>/gated-<slug>.md` (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)):

```markdown
# Gated contract: <task-name>

## Gates
N: <count>

### Gate 1: <action name>
- **Action**: <exact action, e.g., "POST https://api.example.com/send with body X">
- **Preview shown to human**:
  ```
  <literal preview content — the bytes, the body, the diff, the recipients>
  ```
- **Gate question**: <"Approve and send? (yes/no)" — literal question>
- **Granularity**: <per-instance | batch of N>
- **On approve**: <execute exactly as previewed>
- **On reject**: <halt + report | branch to alternative | escalate>
- **Cancellability**: <fire-and-forget | N-second window | reversible via storno/undo>

[repeat per gate]

## Producer (pre-gate)
- Worker: <claude | codex | subagent:<type> (model: opus)>
- Output: <the artifact that the preview shows>

## Executor (post-gate)
- Worker: <usually the same as producer; sometimes a different worker if the action requires specific tools>
- Constraint: executor must run the exact action shown in the preview — no deviation

## Compositions
- <none | pre-gate stage is approach:pipeline | pre-gate is approach:critic | …>

## Assumptions
<inferences during framing — recipient-set completeness, reversibility model — or "none">

## Extraction-confidence
<low | medium | high>
```

Then surface the path to the orchestrator.

### 8. Stop

The skill is framing-only. Do not start producing. Do not preview. Do not fire. The orchestrator decides when and whether to run.

## Anti-Patterns

- **Gate on reversible local work.** Friction without value. If undo is cheap, no gate needed.
- **Batch gate on irreversible actions.** Single yes covering N invoices, N deploys, N sends. The user can't really evaluate N items at once. Default per-instance for legal/financial.
- **Preview that hides the dangerous bit.** Truncation, summarization, omitting the recipient or amount or environment. Preview must show the bytes hitting the wire.
- **"Once-confirmed = always-confirmed."** Approving one item implicitly approving all subsequent items in a session or batch. Explicit partnership rule against this — re-gate per boundary.
- **Post-hoc gate.** Action fires, then asks "was that OK?" — theater. The gate must be BEFORE the irreversible step.
- **No abort path documented.** User clicks approve without knowing if there's a cancellation window or storno path. State it.
- **Producer is also executor with no separation.** The same worker that drafted the action also fires it without re-checking against the approved preview. Risk: drift between previewed and executed. Constraint: executor must run the exact action shown in the preview.
- **Auto-pass when "trusted."** Some workflows decay gates after N successful approvals ("you've approved 10 in a row, skip the gate"). Defeats the purpose; the 11th can be the bad one.

## Rules

- The skill always outputs a single Markdown contract file; it never executes the gated action.
- Every Gated contract must declare per gate: action, preview, gate question, granularity, on-approve, on-reject, cancellability. Plus Assumptions and Extraction-confidence.
- Preview must be literal — exact action, no truncation of the dangerous bits.
- Default granularity is per-instance for irreversible legal/financial actions. Batch is opt-in.
- Executor must run exactly the previewed action — no deviation. If deviation is needed, re-gate.
- When a producer or executor worker is a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- "Once-confirmed = always-confirmed" is never assumed; re-gate per session boundary.
- If the task doesn't fit Gated, recommend the right approach and stop — don't force-fit. When recommending a sibling, check `<available_skills>` first; if not installed, describe inline.
- Contract path: `<contracts-root>/gated-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/gated-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:event`, `approach:dialogue`, `approach:search`, `approach:blackboard`. Related external skills: `/loop` (Loop scheduling executor), `/sef-inbox` + `/invoice` + `/dm-invoice` + `/transfer` + `/tax` (concrete gated flows in the user's library).
