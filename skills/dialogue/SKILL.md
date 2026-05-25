---
name: dialogue
description: Dialogue approach framer for work where the conversation IS the deliverable — requirements gathering, design exploration, Socratic teaching, thinking-out-loud, ambiguity resolution. ALWAYS invoke this skill when the user asks to think-through, explore, talk-through, walk-through, scope, brainstorm, surface assumptions, gather requirements, clarify intent, or expresses uncertainty about what they want and wants a thinking partner rather than an answer. Produces a Dialogue contract — purpose, opening question, branching strategy, termination, captured artifact (what gets written down at the end). Recommends /teach-me when the dialogue is Socratic teaching specifically. Does not execute. Do not jump straight to an answer when value is in the question structure — use this skill first to frame dialogue mode explicitly. Skip for tasks with a clear deliverable (use the matching approach) or simple factual questions (use approach:one-shot).
# description-style: directive + negative constraint (Seleznov)
# rationale: Dialogue framing competes with default Claude behavior (would otherwise jump to answer-mode); directive style forces explicit recognition that the conversation IS the deliverable — value lies in question structure, not in resolution.
---

<!--
ACTIVATION TESTS (for /skill:validate --bench):
1. SHOULD activate (knowledge work / strategy): "Help me think through whether to migrate to monorepo or stay split"
2. SHOULD activate (consulting prep): "Walk me through what I'm actually trying to do with this client engagement"
3. SHOULD NOT activate: "What's the capital of France" (single factual answer — approach:one-shot)
4. BOUNDARY: "Teach me CRDTs" (could be approach:dialogue for free-form exploration, or /teach-me for structured Socratic teaching — skill must clarify and recommend /teach-me if structured)
-->

# dialogue

Frames work where the conversation IS the deliverable as a Dialogue contract — purpose, opening question, branching strategy, termination, captured artifact. The approach's value is *recognizing* that the user wants to think out loud rather than receive an answer. One of the 11 named approaches in the `approach` plugin; frame-only, never executes.

> See [PRINCIPLES.md](../../PRINCIPLES.md) for shared rules (frame-only, sub-Agent Opus, Assumptions, Direct-Use Rule, contracts-root resolution, composition explicitness, recommend-never-force-fit).
>
> See [ARCHITECTURE.md](../../ARCHITECTURE.md) for the `problem → frame → approach → solution` model.
>
> **Architectural note.** Dialogue is the most frame-adjacent of the 11 approaches. When the dialogue's *value* is articulation / perception / what the user sees, the work is genuinely frame-space and will eventually be served by the `/frame` plugin. Until then, `approach:dialogue` carries this load — both as a standalone approach and as the engine inside `approach:composer`'s Phase 1.

## When to Use

- The user is thinking out loud and wants a thinking partner, not an oracle.
- Requirements / scope / intent is ambiguous and needs collaborative surfacing.
- Multiple valid paths exist and the user needs to feel into the right one.
- Teaching / learning where Socratic question-driven exploration beats lecture.
- Composing with other approaches: Dialogue → Pipeline (talk first, then frame the resolved plan); Dialogue → Search (talk to set criteria, then evaluate options); recurring Dialogue (re-engage same topic over multiple sessions).

**Examples — knowledge work + strategy:**
- Architecture decisions, design exploration, requirements gathering, scope clarification
- Decision-making under ambiguity, ideation, exploring feelings about an option
- Thinking out loud about a research direction, surfacing what's known about a topic
- Exploring a concept before committing to a framing
- Consulting prep (what am I actually trying to land with this client?)
- Pre-conversation prep before a hard conversation

**Examples — code / ops:**
- Should we migrate to monorepo? Stay split? What's actually at stake?
- Should this be a new service or an extension of the existing one?
- Which abstraction level should the API live at?

## When NOT to Use

- Task has a clear deliverable — use the matching approach for the deliverable (Pipeline / Swarm / Critic / etc.).
- Simple factual question — use `approach:one-shot`.
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
- Frame-clarification — "what is the actual problem I'm trying to solve?" (genuinely frame-space; long-term routes to /frame)

Purpose anchors the dialogue — without it, the conversation wanders.

### 3. Define opening question

The first move sets the dialogue's shape. Not "what do you want?" but a specific, useful entry that surfaces what the user already knows or feels.

Examples:
- Requirements: "What's the smallest thing that, if it worked, would unblock you?"
- Decision: "If you had to commit to A or B today and explain to your team why, which would you pick — and what would you say?"
- Sensemaking: "Walk me through what you've tried, and where it stopped working."
- Frame-clarification: "What does done look like that doesn't look like that now?"

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
- **Captured artifact ready** — the dialogue produces the thing it was meant to (requirements doc, decision rationale, design sketch, Problem Statement).

Dialogues that don't terminate well sprawl. Define the stop.

### 6. Define captured artifact

What gets written down at the end? Even Dialogue-as-deliverable usually has *something* persisted:
- A summary of the decision + rationale.
- A requirements list extracted from the conversation.
- A wiki note capturing the concept.
- A plan that the dialogue resolved into.
- A Problem Statement (when invoked from composer Phase 1).
- Or explicitly nothing — "the value was in the talking, no artifact needed."

State what gets captured + where.

### 7. Output the contract

Write to `<contracts-root>/dialogue-<slug>.md` (`<contracts-root>` resolution per [PRINCIPLES.md §5](../../PRINCIPLES.md#5-contracts-root-resolution)):

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
- <none | dialogue resolves into approach:pipeline | dialogue sets criteria for approach:search | dialogue is composer Phase 1 producing Problem Statement | …>

## Assumptions
<inferences during framing — user's mode, willingness to be probed — or "none">

## Extraction-confidence
<low | medium | high>
```

### 8. Stop

The skill is framing-only. Do not start the dialogue. The orchestrator (typically Claude in conversation) follows.

## Anti-Patterns

- **Jumping to answer when user wants to think.** Symptom of skipping the shape recognition. If user says "help me think through X," dialogue mode, not answer mode.
- **Forcing dialogue when user is in execution mode.** Equally bad — talking when user wants action is friction. Read the room.
- **No opening question.** Just "what do you want to talk about?" — the user is here because they don't know yet. The skill writer's job is the entry move.
- **No termination.** Dialogues that don't end well sprawl. Define when to stop.
- **No captured artifact when one was needed.** Dialogue that resolves a decision but no one writes down the decision — value lost. Capture even if minimal.
- **Recommending approach:dialogue when /teach-me applies.** /teach-me is the specialized Socratic-teaching skill with multi-session state. For "teach me X" requests, recommend /teach-me first; only use approach:dialogue if the user explicitly wants free-form exploration instead.

## Rules

- The skill always outputs a single Markdown contract file; it never starts the dialogue.
- Every Dialogue contract must declare: purpose, opening question, branching strategy, termination, captured artifact, Assumptions, Extraction-confidence.
- For Socratic teaching with defined topic + multi-session state, recommend `/teach-me` over approach:dialogue.
- When the dialogue resolves into another approach, the resolution must be declared in Compositions.
- If the task doesn't fit Dialogue, recommend the right approach and stop — don't force-fit. Check `<available_skills>` before suggesting by name; if not installed, describe inline.
- Contract path: `<contracts-root>/dialogue-<slug>.md` per PRINCIPLES.md §5.

## Key Files

- Output: `<contracts-root>/dialogue-<slug>.md` — the contract document.
- Sibling approach skills (all under the `approach` plugin namespace, all live): `approach:composer`, `approach:pipeline`, `approach:swarm`, `approach:critic`, `approach:gated`, `approach:contract`, `approach:loop`, `approach:one-shot`, `approach:event`, `approach:search`, `approach:blackboard`. Related external skills: `/loop` (Loop scheduling executor), `/teach-me` (specialized Socratic-teaching dialogue with multi-session state — recommend over approach:dialogue when topic is defined and learning is the goal), `/orient` (specialized self-orientation dialogue).
- Future sibling plugin: `/frame` — when it ships, frame-clarification dialogues route there (likely `/frame:dialogue` or equivalent), and `approach:dialogue` tightens to focus on requirements / decision / sensemaking dialogues that resolve into an approach contract.
