# shape

Agent-execution shape framers for Claude Code. Each skill takes a task and produces an explicit shape contract — stages, workers, termination, failure handling — for the orchestrator to execute.

## The 11 shapes

| Shape | Skill | Status |
|---|---|---|
| Pipeline | `shape:pipeline` | ✓ live |
| Swarm | `shape:swarm` | ✓ live |
| Critic | `shape:critic` | ✓ live |
| Gated | `shape:gated` | ✓ live |
| Contract | `task-to-verifiable-loop` (external) | ✓ live |
| Loop | `/loop` (external) | ✓ live |
| One-shot | `shape:one-shot` | planned |
| Event | `shape:event` | planned |
| Dialogue | `shape:dialogue` | planned |
| Search | `shape:search` | planned (overlaps `/eval`) |
| Blackboard | `shape:blackboard` | planned |

## Design

- **Frame-only.** Skills produce a contract document; the orchestrator (Claude, the future router skill, or the user) executes by following it.
- **Codex as worker.** Where appropriate, shape contracts assign stages to Codex via the `codex` plugin rather than to Claude. Heavy single-task, deterministic, long-context stages are Codex's strength.
- **Claude sub-Agents always use Opus.** When a shape contract assigns a Claude sub-Agent as the worker for a stage or slice, the contract MUST specify `model: opus`. No exceptions — Sonnet and Haiku are not used as sub-Agent workers in any shape skill in this plugin.
- **Composition.** Shapes nest. A Pipeline whose one stage is itself a Critic, or a Loop whose tick spawns a Swarm — explicit nested-shape declarations in the contract, no hidden composition.

## Future

A router skill (`shape:router` or similar) will dispatch from a user request directly to the right shape skill(s), composing where needed. Builds incrementally — author shapes first, route after enough exist.

## License

MIT
