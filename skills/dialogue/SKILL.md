---
name: dialogue
description: Dialogue shape framer for work where the conversation IS the deliverable — requirements gathering, design exploration, Socratic teaching, thinking-out-loud, ambiguity resolution. ALWAYS invoke this skill when the user asks to think-through, explore, talk-through, walk-through, scope, brainstorm, surface assumptions, gather requirements, clarify intent, or expresses uncertainty about what they want and wants a thinking partner rather than an answer. Produces a Dialogue contract — purpose, opening question, branching strategy, termination, captured artifact (what gets written down at the end). Recommends /teach-me when the dialogue is Socratic teaching specifically. Does not execute. Do not jump straight to an answer when value is in the question structure — use this skill first to frame dialogue mode explicitly. Skip for tasks with a clear deliverable (use the matching shape) or simple factual questions (use shape:one-shot).
# description-style: directive + negative constraint (Seleznov)
# rationale: Dialogue framing competes with default Claude behavior (would otherwise jump to answer-mode); directive style forces explicit recognition that the conversation IS the deliverable — value lies in question structure, not in resolution.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate: "Help me think through whether to migrate to monorepo or stay split"
2. SHOULD NOT activate: "What's the capital of France" (single factual answer — shape:one-shot)
3. BOUNDARY: "Teach me CRDTs" (could be shape:dialogue for free-form exploration, or /teach-me for structured Socratic teaching — skill must clarify and recommend /teach-me if structured)
-->

# dialogue

Frames work where the conversation IS the deliverable as a Dialogue contract — purpose, opening question, branching strategy, termination, captured artifact. The shape's value is *recognizing* that the user wants to think out loud rather than receive an answer. One of an 11-shape family in the `shape` plugin; frame-only, never executes.

## When to Use

- The user is thinking out loud and wants a thinking partner, not an oracle.
- Requirements / scope / intent is ambiguous and needs collaborative surfacing.
- Multiple valid paths exist and the user needs to feel into the right one.
- Teaching / learning where Socratic question-driven exploration beats lecture.
- Examples (mixed coding + knowledge work): architecture decisions, design exploration, requirements gathering, scope clarification, decision-making under ambiguity, ideation, exploring feelings about an option, thinking out loud about a research direction, surfacing what's known about a topic, exploring a concept before committing to a framing.
- Composing with other shapes: Dialogue → Pipeline (talk first, then frame the resolved plan); Dialogue → Search (talk to set criteria, then evaluate options); recurring Dialogue (re-engage same topic over multiple sessions).

## When NOT to Use

- Task has a clear deliverable — use the matching shape for the deliverable (Pipeline / Swarm / Critic / etc.).
- Simple factual question — use `shape:one-shot`.
- Socratic teaching on a defined topic — recommend `/teach-me` (it's the specialized version with diagnostic intake, retrieval practice, container-scoped state).
- The user has stated intent clearly and just wants execution — don't talk it to death.
- The user is in execution mode and a Dialogue would be friction — read the room.

## Process

The skill outputs a **Dialogue contract** — a structured document.

### 1. Confirm shape fits and check for /teach-me

Dialogue fits if:
- The conversation produces value independent of any final artifact.
- Asking the right questions matters more than providing answers.
- Multiple sessions on the same topic are plausible (the thinking continues).

Special case: if the dialogue is Socratic teaching with diagnostic intake and retrieval practice (the user wants to *learn deeply*, not just discuss), **recommend `/teach-me` instead** — it's the specialized version with persistent multi-session lesson state.

### 2. Define purpose

State the purpose in one sentence. What does the user want from this dialogue?

- Requirements surfacing — "what do we actually need?"
- Decision exploration — "should we go A or B?"
- Ideation — "what's the space of possible approaches?"
- Sensemaking — "what's actually going on here?"
- Teaching — "help me understand X" (recommend /teach-me first)

Purpose anchors the dialogue — without it, the conversation wanders.

### 3. Define opening question

The first move sets the dialogue's shape. Not "what do you want?" but a specific, useful entry that surfaces what the user already knows or feels.

Examples:
- Requirements: "What's the smallest thing that, if it worked, would unblock you?"
- Decision: "If you had to commit to A or B today and explain to your team why, which would you pick — and what would you say?"
- Sensemaking: "Walk me through what you've tried, and where it stopped working."

### 4. Define branching strategy

Dialogues branch. Strategy options:

- **Follow the user's lead.** When the user goes somewhere, follow. Re-anchor only if it drifts far from purpose.
- **Surface-then-prune.** First pass surfaces N angles; second pass picks the live one.
- **Pre-mortem / post-mortem.** Project forward ("imagine we shipped this — what broke?") or backward ("if this were already done, what would we wish we'd asked?").
- **Socratic.** Questions only, no answers. (See `/teach-me` for the structured form.)

State which strategy applies.

### 5. Define termination

When does the dialogue stop?

- **User satisfaction** — user explicitly says "I have what I need."
- **Time** — wall-clock cap (rare; usually user-driven).
- **Depth** — N levels of branching or M minutes per branch.
- **Captured artifact ready** — the dialogue produces the thing it was meant to (requirements doc, decision rationale, design sketch).

Dialogues that don't terminate well sprawl. Define the stop.

### 6. Define captured artifact

What gets written down at the end? Even Dialogue-as-deliverable usually has *something* persisted:
- A summary of the decision + rationale.
- A requirements list extracted from the conversation.
- A wiki note capturing the concept.
- A plan that the dialogue resolved into.
- Or explicitly nothing — "the value was in the talking, no artifact needed."

State what gets captured + where.

### 7. Output the contract

Write to `/tmp/dialogue-<slug>.md`:

```markdown
# Dialogue contract: <task-name>

## Purpose
<one sentence: what does the user want from this dialogue?>

## Opening question
<literal first question / opening move>

## Branching strategy
<follow-lead | surface-then-prune | pre-mortem | post-mortem | socratic | …>

## Termination
<user satisfaction | time cap | depth cap | captured artifact ready>

## Captured artifact
<what gets written down + where, or "nothing — value is in the talking">

## Compositions
- <none | dialogue resolves into shape:pipeline | dialogue sets criteria for shape:search | …>
```

### 8. Stop

The skill is framing-only. Do not start the dialogue. The orchestrator (typically Claude in conversation) follows.

## Anti-Patterns

- **Jumping to answer when user wants to think.** Symptom of skipping the shape recognition. If user says "help me think through X," dialogue mode, not answer mode.
- **Forcing dialogue when user is in execution mode.** Equally bad — talking when user wants action is friction. Read the room.
- **No opening question.** Just "what do you want to talk about?" — the user is here because they don't know yet. The skill writer's job is the entry move.
- **No termination.** Dialogues that don't end well sprawl. Define when to stop.
- **No captured artifact when one was needed.** Dialogue that resolves a decision but no one writes down the decision — value lost. Capture even if minimal.
- **Recommending shape:dialogue when /teach-me applies.** /teach-me is the specialized Socratic-teaching skill with multi-session state. For "teach me X" requests, recommend /teach-me first; only use shape:dialogue if the user explicitly wants free-form exploration instead.

## Rules

- The skill always outputs a single Markdown contract file; it never starts the dialogue.
- Every Dialogue contract must declare: purpose, opening question, branching strategy, termination, captured artifact.
- For Socratic teaching with defined topic + multi-session state, recommend `/teach-me` over shape:dialogue.
- When the dialogue resolves into another shape, the resolution must be declared in Compositions.
- If the task doesn't fit Dialogue, recommend the right shape and stop — don't force-fit. Check `<available_skills>` before suggesting by name; if not installed, describe inline.
- Contract path always `/tmp/dialogue-<slug>.md`.

## Key Files

- Output: `/tmp/dialogue-<slug>.md` — the contract document.
- Sibling shape skills (planned, all under the `shape` plugin namespace): `shape:pipeline` (✓ live), `shape:swarm` (✓ live), `shape:critic` (✓ live), `shape:gated` (✓ live), `shape:event` (✓ live), `shape:one-shot`, `shape:search`, `shape:blackboard`, `shape:loop`. Related external skills: `shape:contract`, `/loop` (Loop scheduling), `/teach-me` (specialized Socratic-teaching dialogue with multi-session state — recommend over shape:dialogue when topic is defined and learning is the goal), `/orient` (specialized self-orientation dialogue).
