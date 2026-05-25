# approach

Approach-space framers for Claude Code. Each skill takes a framed problem and produces an explicit **approach contract** — stages, workers, termination, failure handling — for the orchestrator to execute.

The plugin operates in the **approach** layer of the `problem → frame → approach → solution` cascade. See [ARCHITECTURE.md](./ARCHITECTURE.md) for the conceptual model and [PRINCIPLES.md](./PRINCIPLES.md) for the rules shared across all skills.

## The 11 approaches

| Approach | Skill | When it fits |
|---|---|---|
| Pipeline | `approach:pipeline` | Multi-stage with fixed sequential ordering and handoffs |
| Swarm | `approach:swarm` | N parallel-independent slices + explicit reducer |
| Critic | `approach:critic` | Quality-critical work needing iteration against a rubric (or symmetric debate) |
| Gated | `approach:gated` | Irreversible / blast-radius / shared-state steps needing human authorization |
| Contract | `approach:contract` | Broad / ambiguous / fake-progress-prone — verifiable slices + named verifiers |
| Loop | `approach:loop` | Scheduled-recurrence work (cron, interval, self-paced) across sessions |
| Event | `approach:event` | Reactive trigger → fresh-session handler |
| Dialogue | `approach:dialogue` | Conversation IS the deliverable — requirements gathering, sensemaking |
| Search | `approach:search` | Explore-evaluate-prune over a candidate space |
| Blackboard | `approach:blackboard` | Opportunistic multi-agent over a shared workspace |
| One-shot | `approach:one-shot` | Degenerate shape — explicit "no decomposition needed" |

All work for both code-shaped tasks (refactor, build, deploy, test) and knowledge-work-shaped tasks (research, synthesis, document analysis, multi-source claim verification, consulting prep, strategy comparison).

## The composer

| Skill | Role |
|---|---|
| `approach:composer` | Composition planner. Extracts (or consumes) a Problem Statement, maps to candidate approaches, composes a nested tree, writes root + per-approach child contracts. Frame-only — produces documents, never executes. |

The composer is the entry point for multi-approach work. For tasks that obviously fit one approach, invoke that approach directly.

## Architectural location

```
Problem        Frame                   Approach                Solution
   │             │                        │                       │
   │             │                        │                       │
   ▼             ▼                        ▼                       ▼
"situation in   "articulation,         "11 named patterns,      "code, docs,
the world"     filter, perception"     pick or compose;          decisions,
                                       produce contract"         sent messages"
                                       (this plugin)             (orchestrator
                                                                  executes)
```

This plugin operates in the **approach** layer. Frame work (problem articulation, perception probes, lens-application) is currently absorbed inline by composer's Phase 1 and Dialogue mode — long-term, it migrates to a separate `/frame` plugin.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full model.

## Shared principles

Every skill in the plugin follows the rules in [PRINCIPLES.md](./PRINCIPLES.md). The most important:

- **Frame-only.** Skills produce contract documents; the orchestrator executes by following them. No skill spawns Agents, invokes `codex exec`, edits `settings.json`, calls `/loop` or `/schedule`, or fires gates directly.
- **Sub-Agent workers always use Opus.** When a contract assigns a Claude sub-Agent, the contract MUST specify `model: opus`. Never Sonnet, never Haiku.
- **Surface uncertainty, don't hide it.** Every contract has an `Assumptions:` field; composer surfaces `extraction-confidence`. Low-confidence work is provisional and revisits on first failure.
- **Direct-Use Rule.** Exercise the real intended surface as early as possible — installed MCPs through real clients, web apps through real browsers, CLIs through real commands, APIs through real HTTP. Lower-level tests are fine for progress but do not replace the real surface when it's available.
- **Composition is explicit.** Hidden nesting is forbidden — every nested approach must be named in `Compositions:`.
- **Knowledge work is co-equal with coding work.** Approaches work for both; examples in each SKILL.md are balanced.
- **Recommend, never force-fit.** If the task doesn't fit the active skill's shape, recommend the right sibling and stop.

## Worker types

Contracts assign workers per stage:

| Worker | Strength |
|---|---|
| Claude inline | direction-setting, judgment, orchestration |
| Codex (`codex exec`) | long-context single-task, mechanical edits, deterministic refactors |
| Claude sub-Agent (model: opus) | isolated exploration in its own conversation |
| Codex `/adversarial-review` | adversarial critique, second-opinion review |
| Human | taste, business decision, credentialed approval, destructive side effects |

Default lean: **Claude for direction, Codex for execution.** Assignment is per-stage, not defaulted globally.

## Composition

Approaches nest. A Pipeline whose one stage is a Critic. A Loop whose tick spawns a Swarm. A Contract whose verification spine wraps a Pipeline-with-Gated-deploy.

Nesting depth is capped at 2 by default in composer; deeper requires explicit reason. Hidden composition is forbidden — every nested approach must be named in the parent contract's `Compositions:` field.

## Future

- **`/frame` plugin** — sibling plugin for the upstream frame-space work (problem articulation, perception probes, lens-application, JTBD extraction, reframing). When it ships, composer's Phase 1 delegates to it.
- **Runtime router** — a future routing skill that dispatches from a user request directly to the right approach skill(s), composing where needed. Builds incrementally; for now the user (or Claude in conversation) routes.

## License

MIT
