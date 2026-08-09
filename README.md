# Solara AI

A coding AI platform.

Solara is built around software development — writing, understanding, changing,
running, and debugging real code in real projects. It is a platform rather than a
chat interface over a model: the model is a replaceable component inside Solara,
and identity, tools, context, memory, and security belong to the system and survive
swapping the model underneath.

## Status

**Architecture phase.** No implementation yet. The design is complete and recorded.

- [Architecture](docs/architecture/architecture.md) — the formal design.
- [Decision log](docs/architecture/decisions.md) — every decision, its rationale,
  and what was rejected.

## Shape

A core engine and a local daemon, with thin clients over a versioned API — VS Code,
desktop, web, and terminal all use the same intelligence rather than
reimplementing it.

Models run locally through llama.cpp by default, behind an interface that also
accepts remote backends. Coding-focused models such as Qwen2.5-Coder and
Qwen3-Coder are the initial targets, with Solara-specific fine-tuned models as a
longer-term goal.

The architecture assumes constrained hardware. Context and output tokens are
treated as scarce resources, and designs that only work with a large model are
rejected.

## Principles

- The model is a component, not the product.
- Context is the scarce resource.
- Security is a single choke point, not a practice.
- Structure before semantics — parsers and search before embeddings.
- Visible over clever — memory, plans, and permissions are inspectable.
- Seams before features — boundaries defined now, implementations later.
