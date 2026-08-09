# Solara AI

A coding AI platform.

Solara is built around software development — writing, understanding, changing,
running, and debugging real code in real projects. It is a platform rather than a
chat interface over a model: the model is a replaceable component inside Solara, and
identity, tools, context, memory, and security belong to the system and survive
swapping the model underneath.

## Status

**Architecture phase.** No implementation yet. The design is complete and recorded.

| Document | Covers |
|---|---|
| [Architecture](docs/architecture/architecture.md) | The formal design — start here |
| [Decision log](docs/architecture/decisions.md) | Every decision, its rationale, and what was rejected |
| [Execution graph](docs/architecture/execution-graph.md) | Work units, scheduler, concurrency, resource model |
| [Model orchestration](docs/architecture/model-orchestration.md) | Speculative decoding, cascade, critic, routing, profiles |
| [Code intelligence](docs/architecture/code-intelligence.md) | Symbol, dependency and call graphs; change-impact analysis |
| [Resilience](docs/architecture/resilience.md) | Stuck detection, recovery, snapshots, budgets |
| [Learning](docs/architecture/learning.md) | Trajectories, eval ratchet, distillation |
| [Distribution](docs/architecture/distribution.md) | Worker pools, placement, locality constraints |

## Shape

A core engine and a local daemon, with thin clients over a versioned API — VS Code,
desktop, web, and terminal all use the same intelligence rather than reimplementing
it.

Tasks execute as a graph of work units under a resource-aware scheduler, not a fixed
loop. Models run locally through llama.cpp by default, behind an interface that also
accepts remote backends and remote inference workers. Coding-focused models such as
Qwen2.5-Coder and Qwen3-Coder are the initial targets, with Solara-specific
fine-tuned models as a longer-term goal reached by distillation.

## Principles

- The model is a component, not the product.
- Context is the scarce resource.
- Security is a single choke point, not a practice.
- Structure before semantics — parsers and graphs before embeddings.
- Honest uncertainty — approximate analysis is labelled, not hidden.
- Visible over clever — memory, plans, routing and permissions are inspectable.
- Degrade, don't fail.
- **Capability in the architecture, modest defaults in configuration.** Hardware
  determines settings, never structure.
