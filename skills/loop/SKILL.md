---
name: loop
description: Loop approach framer for scheduled-recurrence work — same prompt fires on a cadence, state accumulates between fires. ALWAYS invoke this skill when the user asks to schedule, recur, repeat, poll, monitor, watch, cron, periodically check, daily/weekly run, every-N-min run, or describe work that should fire on a temporal cadence rather than once. Distinct from approach:contract (Contract's loop is internal verify-execute within one task; Loop's loop is temporal across sessions). Produces a Loop contract — trigger cadence (fixed interval, cron, self-paced), per-fire prompt or skill invocation, accumulated state across fires, termination, dead-loop detection. Recommends /loop for in-session self-paced and fixed-interval recurrence; recommends /schedule for remote-cron scheduled agents. Does not execute or wire. Skip for one-time work (use approach:one-shot), event-triggered work (use approach:event for when-X-happens), or in-task verify-execute iteration (use approach:contract).
# description-style: directive + negative constraint (Seleznov)
# rationale: Loop framing competes with default Claude behavior (would otherwise hand-wave "set up a recurring check"); directive style forces explicit cadence + state + termination decisions. Many user workflows are scheduled-recurrence-shaped (daily summary, weekly review, hourly status check) — making the approach explicit prevents drift between intent and what gets wired.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (knowledge work): "Daily, check my reading list and surface anything new I should read first"
2. SHOULD activate (coding): "Every Monday at 9am, summarize last week's commits"
3. SHOULD NOT activate: "When a new SEF invoice arrives, list it" (event-triggered, not scheduled — approach:event)
4. BOUNDARY: "Poll the deploy status until done" (could be Loop with short interval + termination, or in-session /loop self-pace — skill should clarify based on whether external observability matters)
-->

# loop

Frames scheduled-recurrence work as a Loop contract — cadence, per-fire action, accumulated state, termination, dead-loop detection. Distinct from Contract (Contract's loop is internal verify-execute within one task; Loop's loop is temporal across sessions). One of the 11 named approaches in the `approach` plugin; frame-only, never wires or schedules.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- Work fires on a temporal cadence — every N minutes, daily at HH:MM, on cron, or self-paced by the agent.
- Same action repeats across fires, with state potentially accumulating.
- Composing with other approaches: each tick is itself a Pipeline (check → analyze → notify), or a Critic (review accumulated state and decide if action needed), or fires into a Gated handler (auto-detect candidates, gate the action).

**Examples — knowledge work:**
- Daily inbox summary, weekly research-feed digest, hourly Slack/email triage
- Monthly portfolio review, periodic wiki-note revisit ("what did I learn this week")
- Quarterly strategy reassessment with structured intake

**Examples — life ops:**
- Daily voltage check, weekly orient session, monthly tax review

**Examples — code / ops:**
- Nightly CI status check, hourly deploy poll, daily test-coverage report
- Weekly dependency-update scan

## When NOT to Use

- One-time work — use `approach:one-shot`.
- Triggered by an event rather than a clock — use `approach:event` ("when X happens, do Y" is Event, not Loop).
- Within-task verify-execute iteration — use `approach:contract` (Loop Contract).
- In-session retry loops where the agent decides next step based on prior result — that's `approach:pipeline` with branching, not Loop.

## Process

The skill outputs a **Loop contract** — a structured document, not wired automation. Wiring is delegated to `/loop` (in-session, self-paced or fixed interval) or `/schedule` (remote cron).

### 1. Define cadence

Pick the cadence shape:

- **Fixed interval** — every N minutes / hours / days. Simple recurrence.
- **Cron** — at HH:MM on specific days. Calendar-aligned.
- **Self-paced** — agent decides next wake-up based on current state (e.g., poll faster when activity is high, slower when quiet). Uses `/loop` without an interval + `ScheduleWakeup`.

State which applies + the parameter (interval, cron expression, or pacing rule).

### 2. Define per-fire action

Each fire runs the same action. Spec:

- **Action** — what runs (prompt, slash command, sub-Agent invocation, Codex run).
- **Worker** — usually a fresh Claude session (the harness fires); within the action, can delegate to Codex/sub-Agents.
- **Input at fire time** — what the per-fire action knows (timestamp, prior state, accumulated metrics).
- **Output per fire** — what each fire produces (file, notification, state update).

### 3. Define accumulated state

Loops often need state across fires:

- **State location** — where it lives (file, named artifact, shared workspace).
- **Write policy** — append-only / overwrite / merged.
- **Read at fire start** — what each fire reads before acting.
- **Compaction** — how state stays bounded over many fires (e.g., keep last 30 days, summarize older entries, archive).

If the loop is truly stateless (each fire independent, no carryover), declare so explicitly.

### 4. Define termination

- **Until condition** — stop when state reaches X (e.g., "stop once deploy succeeded").
- **Time budget** — stop after T total wall-clock time.
- **Fire count cap** — stop after N fires.
- **External signal** — user/event marks "done."
- **Never** — open-ended (rare; usually has a maintenance review cadence instead).

State the rule. "Never" should be a deliberate choice, not an oversight.

### 5. Define dead-loop detection

**Load-bearing.** Loops can run forever doing nothing useful (e.g., polling something that never changes). Without detection, you waste tokens/cycles.

Detection strategies:
- **No-change counter** — N consecutive fires with no state delta → stop or alert.
- **Cost cap** — total tokens/cycles per fire × fire count > budget → stop.
- **External heartbeat** — fire writes timestamp; external check confirms loop is doing meaningful work.

State which applies.

### 6. Apply the Direct-Use Rule

If the loop polls a real system, the polling should exercise that system through its real surface (real API call, real query) rather than against a cached or mocked version. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible).

### 7. Output the contract

Write to `<contracts-root>/loop-<slug>.md` (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)):

```markdown
# Loop contract: <task-name>

## Cadence
- Shape: <fixed-interval | cron | self-paced>
- Parameter: <interval | cron expr | pacing rule>

## Per-fire action
- Action: <prompt / slash command / sub-Agent / codex exec>
- Worker: fresh Claude session per fire (handler may delegate)
- Input at fire: <timestamp + prior state + …>
- Output per fire: <artifact / notification / state update>
- Internal composition: <none | each fire is approach:pipeline | …>

## Accumulated state
- Location: <path / artifact / "stateless">
- Write policy: <append-only | overwrite | merge>
- Read at fire start: <what each fire reads>
- Compaction: <how state stays bounded>

## Termination
- Rule: <until-condition | time-budget | fire-cap | external-signal | never>
- Parameter: <X | T | N | signal source>

## Dead-loop detection
- Strategy: <no-change counter | cost cap | external heartbeat>
- Action on detection: <stop | alert | switch to slower cadence>

## Wiring (delegated)
- Path: </loop in-session | /schedule remote cron | manual launchctl/cron>
- Command or config: <the literal invocation or config>

## Assumptions
<inferences during framing — state-size bounds, expected fire frequency — or "none">

## Extraction-confidence
<low | medium | high>
```

Then surface the path to the orchestrator.

### 8. Stop

The skill is framing-only. Do not invoke `/loop`. Do not wire `/schedule`. Do not start the loop. The orchestrator (or user) wires.

## Anti-Patterns

- **No termination.** Loop runs forever doing nothing. Always define the stop.
- **No dead-loop detection.** Polling something that never changes burns tokens silently. Always declare the detection strategy.
- **State growing unbounded.** Appending to a log every fire with no compaction → state file becomes unreadable / hits limits. Compaction strategy is non-optional for long-running loops.
- **Confusing Loop with Event.** "When X happens, do Y" is Event (trigger-based, irregular). "Every N min, check if Y" is Loop (clock-based, regular). The distinction matters because wiring is different (`/loop` and `/schedule` for Loop; hooks for Event).
- **Confusing Loop with Contract.** Contract's "loop" is internal verify-execute toward a goal within one task. Loop's "loop" is temporal recurrence across sessions. Different beasts.
- **Wiring inside the skill.** The skill is frame-only per [PRINCIPLES.md §1](../../PRINCIPLES.md#1-frame-onlynever-executes). Wiring delegates to `/loop` or `/schedule`.

## Rules

- The skill always outputs a single Markdown contract file; it never wires or runs the loop.
- Every Loop contract must declare: cadence, per-fire action, accumulated state, termination, dead-loop detection, wiring path, Assumptions, Extraction-confidence.
- Termination is non-optional. "Never" must be a deliberate choice with a maintenance review cadence.
- Dead-loop detection is non-optional.
- Wiring is delegated to `/loop` (in-session, fixed or self-paced) or `/schedule` (remote cron).
- When the per-fire action internally spawns a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- If the per-fire action does anything irreversible, the contract MUST declare composition with `approach:gated`.
- If the task doesn't fit Loop, recommend the right approach and stop — don't force-fit. Check `<available_skills>` before suggesting by name; if not installed, describe inline.
- Contract path: `<contracts-root>/loop-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/loop-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:contract`, `approach:one-shot`, `approach:event`, `approach:dialogue`, `approach:search`, `approach:blackboard`. Related external skills: `/loop` (in-session loop executor, self-paced or fixed-interval), `/schedule` (remote-cron scheduled agents), `/gut-check` (canonical user-invoked check — useful when dead-loop detection is uncertain or the user wants honest assessment of whether the loop is doing meaningful work).
