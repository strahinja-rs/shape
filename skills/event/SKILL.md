---
name: event
description: Event approach framer for reactive flows that fire when a trigger condition occurs. ALWAYS invoke this skill when the user asks to plan, wire, or scope work that should run automatically on a trigger — file change, cron, webhook, inbox arrival, new email, new SEF invoice, new GitHub PR, CI failure, Claude Code lifecycle hook, system state change, or any "when X happens, do Y" pattern. Produces an Event contract — trigger detection, handler (bounded, fresh-session), rate limit, idempotency, failure handling, wiring path (delegated to /update-config for hooks, /schedule for cron). Does not execute or wire; the orchestrator follows. Do not just describe an action as "should fire automatically" — use this skill first to make trigger, handler, and wiring explicit. Skip for synchronous user-initiated work, fully-reversible local triggers with no real handler logic, or work needing inter-fire coordination (use approach:blackboard).
# description-style: directive + negative constraint (Seleznov)
# rationale: Event framing competes with default Claude behavior (would otherwise hand-wave "we could automate this"); directive style forces explicit trigger + handler + wiring + idempotency. Especially important because handler runs in a fresh session — statelessness is a hard constraint that hand-wavy descriptions miss.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (knowledge work / life ops): "When a new SEF invoice arrives in my inbox, surface it for review"
2. SHOULD activate (coding): "When CI fails on main, post to Slack"
3. SHOULD NOT activate: "Send this email now" (user-initiated, no trigger)
4. BOUNDARY: "Check email every 15 minutes" (Event with cron trigger via /schedule, or session-local /loop self-pace — skill must clarify the difference and pick)
-->

# event

Frames a reactive flow as an Event contract — trigger, handler, rate limit, idempotency, wiring. The handler runs in a **fresh session** on each fire (no in-session memory), which is the single most important design constraint. One of the 11 named approaches in the `approach` plugin; frame-only, never executes or wires.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.

## When to Use

- Work should fire automatically when condition X is met, without the user issuing the command.
- Trigger is well-defined (deterministic detection — file event, cron tick, webhook POST, inbox arrival, system state change, conversation lifecycle hook).
- Handler is bounded — terminates per fire, doesn't run forever.
- Composing with other approaches: Event → Gated (handler produces a preview, human gates the action); Event → Pipeline (handler does multi-stage work); Event firing into a Blackboard.

**Examples — knowledge work + life ops:**
- "When a new SEF invoice arrives, list it for review"
- "When a new paper drops in my feed, surface for triage"
- "When a new wiki note is added, suggest related links"
- "Daily 18:00, summarize today's captured items in the inbox"
- "Before any commit, run the linter" (lifecycle hook on Claude Code)

**Examples — code / ops:**
- "When CI fails on main, post to Slack"
- "When this directory changes, rebuild docs"
- "Every Monday 09:00, summarize last week's commits" (treated as Event with cron trigger, or as `approach:loop` — Event for one-off; Loop for recurring scheduled state-accumulating work)

## When NOT to Use

- Synchronous user-initiated work — just do it; no trigger needed.
- Trigger is ambiguous or flaky — noisy detection produces noisy fires; tighten the trigger first or use a different approach.
- Handler needs in-session conversation context that a fresh-session fire won't have — the trigger fires a new Claude with no memory of prior turns.
- "On every X" volume is too high (many fires per minute) — rate-limit or change the trigger granularity.
- Work needing inter-fire coordination between concurrent handlers — use `approach:blackboard` or restructure.

## Process

The skill outputs an **Event contract** — a structured document, not executed work. Frame, write, hand back. Wiring is delegated.

### 1. Define the trigger precisely

Each trigger needs:
- **Source** — file event / cron / webhook / inbox / Claude Code hook (UserPromptSubmit, Stop, etc.) / system state / external service.
- **Condition** — the specific event in the source (e.g., not just "file change" but "new file matching `~/inbox/*.pdf`").
- **Frequency cap** — expected fire rate. Helps catch noisy triggers early.
- **Detection mechanism** — what watches for the event. Hook in `settings.json`? Cron via `/schedule`? Webhook URL? External service?

Imprecise triggers produce noise. Tighten before continuing.

### 2. Define the handler

**Load-bearing.** The handler runs in a **fresh Claude session** on each fire — no memory of prior fires, no carry-over conversation state, no in-flight tool results. This is the single biggest design trap.

Handler spec:
- **Worker** — the harness fires a new Claude. Within the handler, the new Claude can delegate (call Codex, spawn sub-Agents, invoke other skills).
- **Input available at fire time** — what does the trigger pass in? (filename, payload, timestamp, exit code, etc.) Be explicit.
- **State the handler needs that isn't in the fresh session** — must be read from disk, queried from an API, or otherwise re-fetched.
- **Bounded scope** — what the handler does and explicitly doesn't do. Should terminate cleanly per fire.
- **Output** — what the handler produces (file written, message sent, state updated, surface to user, etc.).

### 3. Define rate limit and dedupe

Without these, a flaky trigger fires the handler in a loop.

- **Throttle** — minimum interval between fires (e.g., "max one fire per 5 min"). Suppresses bursts.
- **Debounce** — wait for quiet (e.g., "wait 2 seconds of no events before firing"). Useful for file-change storms.
- **Concurrency cap** — max simultaneous handler instances (usually 1; the handler fires while a prior one is still running is rarely desired).

State which apply.

### 4. Define idempotency

**Load-bearing.** Because the handler is stateless per fire, it can re-process the same input if the trigger re-fires. Without an idempotency check, you get duplicate work (duplicate emails sent, duplicate invoices processed, duplicate Slack messages posted).

Idempotency strategies:
- **Marker file** — handler writes `~/.cache/event-<id>/processed/<input-hash>` and checks before processing.
- **Inbox state** — handler queries the source ("is this invoice already marked read?") before acting.
- **Natural idempotency** — action is itself idempotent (e.g., re-running a build, re-syncing state). Document why.

State which applies. "No idempotency needed" is rarely true; if claimed, justify.

### 5. Define failure handling

- **Handler errors** — what if the handler itself throws? Log + halt (default), retry with backoff, escalate to user via notification.
- **Side-effect failures** — if the handler's irreversible step fails partway, what state is left behind? Define the cleanup or rollback.
- **Trigger lost** — if the trigger source dies (cron daemon stops, webhook endpoint goes down), how is the user notified? Document the dead-trigger-detection if possible.

### 6. Apply the Direct-Use Rule

If the handler can exercise the real downstream surface (real API call to the real endpoint that will be hit in production), prefer that during initial development. See [PRINCIPLES.md §4](../../PRINCIPLES.md#4-direct-use-rule--exercise-the-real-surface-as-early-as-possible).

### 7. Define wiring (delegated)

The skill produces the contract; it does NOT wire the trigger. Wiring is delegated to:

- **Claude Code hooks** (UserPromptSubmit, Stop, pre/post tool, etc.) → `/update-config` (writes to `settings.json`).
- **Cron-style remote schedule** → `/schedule` (Claude Code's scheduled-agent system).
- **Local file watch** → user wires via `launchctl` (macOS) or systemd (Linux); document the plist/unit shape in the contract.
- **Webhook endpoint** → user wires the external service; contract specifies the URL pattern + payload schema expected.
- **Inbox polling** (no native push) → cron + handler that diffs current vs prior state.

State which wiring path applies and link to the appropriate sibling skill or document the manual wiring steps.

### 8. Output the contract

Write to `<contracts-root>/event-<slug>.md` (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)):

```markdown
# Event contract: <task-name>

## Trigger
- Source: <file | cron | webhook | inbox | Claude Code hook | system state>
- Condition: <specific event predicate>
- Expected frequency: <fires/min, fires/day>
- Detection: <hook config / cron expression / webhook URL / polling interval>

## Handler
- Worker: fresh Claude session (per fire)
- Input at fire time: <what the trigger passes in>
- Required state from disk/API: <what the handler must re-fetch>
- Bounded scope: <what it does, what it doesn't>
- Output: <what it produces>
- Internal composition: <none | handler is approach:pipeline | handler ends in approach:gated | …>

## Rate limit & dedupe
- Throttle: <interval or none>
- Debounce: <delay or none>
- Concurrency cap: <usually 1>

## Idempotency
- Strategy: <marker file at path X | inbox state check | natural idempotency because Y>
- Without this: <what duplicates if a re-fire happens>

## Failure handling
- Handler error: <log + halt | retry N times | escalate via notification to …>
- Partial side effect: <cleanup action | rollback path | "acceptable to leak — document why">
- Dead trigger: <detection mechanism | accept as unmonitored>

## Wiring (delegated)
- Path: <claude-code-hook via /update-config | cron via /schedule | manual launchctl plist | webhook URL via external service>
- Steps: <list the wiring commands or skill invocations>

## Assumptions
<inferences during framing — trigger source reliability, expected payload shape — or "none">

## Extraction-confidence
<low | medium | high>
```

Then surface the path to the orchestrator.

### 9. Stop

The skill is framing-only. Do not edit `settings.json`. Do not create cron jobs. Do not start watchers. Do not invoke the handler. The orchestrator (or the user) wires.

## Anti-Patterns

- **Trigger too noisy.** Handler fires 50×/minute, burns tokens, drowns out real signal. Tighten the predicate or add throttle/debounce. If trigger can't be tightened, this probably isn't the right approach.
- **Handler depends on in-session context.** The fresh session has no memory of prior conversation, prior handler fires, or prior tool calls. State must come from disk or API.
- **Auto-firing irreversible actions.** Event handler that auto-sends/posts/deploys without a gate. Compose with `approach:gated`: handler produces preview, gate waits for human, executor fires. Cato Networks-class attack vector.
- **No idempotency check.** Re-fire on same input produces duplicate work. Always state the dedup strategy or justify why none is needed.
- **Handler unbounded.** Runs forever, blocks the harness, prevents next fire. Define termination.
- **Trigger detection broken silently.** Handler never fires, user assumes it's working. Add a heartbeat or a "this fired last at" indicator the user can inspect.
- **Hidden side effects in handler.** User wired the event for X, handler also does Y and Z that user didn't realize. Bounded scope must be explicit and respected.
- **Wiring inside the skill.** The skill is frame-only per [PRINCIPLES.md §1](../../PRINCIPLES.md#1-frame-onlynever-executes). Wiring (editing settings.json, scheduling cron, creating launchctl plists) is delegated. Don't conflate.

## Rules

- The skill always outputs a single Markdown contract file; it never wires the trigger or runs the handler.
- Every Event contract must declare: trigger source/condition/frequency/detection, handler (worker, input, required state, scope, output), rate limit + dedupe, idempotency strategy, failure handling, wiring path, Assumptions, Extraction-confidence.
- The handler is always a fresh Claude session per fire — no in-session-memory assumptions.
- Idempotency is non-optional. "No idempotency needed" must be justified.
- Wiring is delegated to `/update-config` (Claude Code hooks), `/schedule` (cron), or manual user steps for OS-level watchers and external webhooks.
- When the handler internally spawns a Claude sub-Agent, the contract MUST specify `model: opus` per PRINCIPLES.md §2.
- If the handler does anything irreversible, the contract MUST declare composition with `approach:gated` (handler produces preview, gate awaits human, executor fires the exact approved action).
- If the task doesn't fit Event, recommend the right approach and stop — don't force-fit. When recommending a sibling, check `<available_skills>` first; if not installed, describe inline.
- Contract path: `<contracts-root>/event-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/event-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:dialogue`, `approach:search`, `approach:blackboard`. Related external skills: `/loop` (in-session Loop with self-pacing), `/schedule` (remote-cron scheduling), `/update-config` (Claude Code hooks wiring), `/sef-inbox` (concrete Event-into-Gated handler pattern).
