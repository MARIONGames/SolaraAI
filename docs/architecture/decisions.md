# Solara AI — Architecture Decision Log

A living record of the decisions that define Solara's architecture, why they were
made, and what was rejected. This file is the source of truth for the design.
It is written down because the previous design conversation was lost.

Status values: **DECIDED** (settled, change requires a new entry) ·
**OPEN** (identified, not yet decided) · **DEFERRED** (decided to postpone).

---

## Foundational premise

Solara AI is a **coding AI first**. Software development is the purpose of the
system, not one feature among many. Every architectural decision is evaluated
against whether it makes Solara better at understanding, writing, changing,
running, and debugging real code in real repositories.

The model is a **component of** Solara, not Solara itself. Identity, coding
behavior, tools, context, memory, and security belong to the system and must
survive swapping the model underneath it.

---

## Hard constraints

These are treated as non-negotiable inputs to every decision below.

| # | Constraint |
|---|---|
| C1 | Coding is the primary purpose, not a feature. |
| C2 | The model is replaceable. |
| C3 | The inference engine is replaceable. |
| C4 | Identity, behavior, tools, context and memory live in Solara, not in the model. |
| C5 | One core intelligence, many interfaces (VS Code, desktop, web). |
| C6 | Security is structural, not bolted on afterwards. |
| C7 | The system must scale from a CPU laptop to much larger hardware. |

### Hardware envelope

The architecture must not be tuned to a single machine.

| Stage | Hardware | Practical model envelope |
|---|---|---|
| Now | i7-1065G7, 16 GB RAM, Iris Plus (shared memory) | 7B Q4 at ~3–6 tok/s on CPU; 1.5B–3B at ~15–25 tok/s. The iGPU shares system memory and does not meaningfully beat CPU here. |
| Next | RTX 5070, 8 GB VRAM, 32 GB RAM | 7B Q4 fully offloaded at ~40–60 tok/s, 16–32k context; 14B with partial offload. |
| Later | Larger GPUs / multi-GPU | Not designed against specifically; must not be excluded. |

**Consequence:** model capability is a *variable the system routes against*, not a
constant. An agent loop that issues 15 model calls per task is minutes-per-step on
current hardware. This is a first-class design pressure, not an optimisation detail.

---

## Product-shape constraints

Recovered from the Solara AI Terms of Service draft (dated 2026-04-19), which was
the only artifact remaining in this repository. The document has been removed from
the tree as requested; these are the architecturally relevant facts extracted from
it before deletion.

| # | Fact | Architectural implication |
|---|---|---|
| P1 | **Rubby Account** — authenticated users with email, username, password, DOB. | Solara has a notion of an authenticated principal. Local deployment has exactly one implicit principal; hosted has many. |
| P2 | **Credits** — metered, billable usage with grants, promotions, correction and anti-exploit rules. | Usage accounting must sit on a boundary the client cannot forge, i.e. server-side, wrapping model invocation. |
| P3 | **`solara.rubby-studios.com`** — a hosted web service. | The web interface is not only a local UI. Multi-tenant deployment must remain reachable from the design. |
| P4 | **Uncensored Mode** — a distinct, more permissive behavior mode. | Implemented as a **different model**, not a different system prompt. Mode selection is therefore a routing concern. |
| P5 | Greek jurisdiction, EU users. | Conversation and log retention are regulated data. Storage design must allow deletion and export. |

### DECIDED — D0: Design the hosted seams, build only the local path

Accounts, credits, and multi-tenancy are **designed for but not implemented**.
They exist in the architecture as explicit, named boundaries with local no-op
implementations:

- an **identity** boundary whose local implementation is a single implicit owner;
- a **quota/accounting** boundary whose local implementation always answers "allowed";
- a **workspace isolation** boundary whose local implementation is a directory root.

*Rationale:* retrofitting a trust boundary into a system that never had one is a
rewrite. Defining the boundary now and stubbing it costs very little.

*Rejected:* ignoring hosted concerns entirely (retrofit cost); building hosted
infrastructure first (no working product for a long time, and inference would move
off the local hardware the project is meant to use).

---

## Decisions

### DECIDED — D1: Solara is a core library plus a local daemon

A core engine package, plus a long-running local process exposing it over an
HTTP/WebSocket API. The VS Code extension, desktop app, and web app are **thin
clients** speaking one protocol; they contain interface logic only, never agent
logic.

*Rationale:* satisfies C5 directly — one implementation of the brain. Keeps the
model resident across sessions, which matters enormously given multi-second load
times and the hardware envelope. Lets interfaces be written in whatever language
suits them. Makes the web app possible at all.

*Rejected:* embedding the core in each interface (forces the core's language into
the VS Code extension host, reloads the model per interface, makes a web app
impossible without later inventing this daemon anyway); a CLI wrapped by other
interfaces (stdout becomes an accidental API, streaming and interactivity suffer).

**Security consequence (non-negotiable):** the daemon loads a model and executes
shell commands. It MUST bind to loopback only and MUST require a per-session
token supplied out-of-band. Without both, any web page the user visits, and any
process on the machine, obtains arbitrary code execution. This is a property of
the transport design, not a later hardening task.

### DECIDED — D2: Python core, TypeScript interfaces

The core engine and daemon are Python. The VS Code extension, desktop app, and web
app are TypeScript, talking to the daemon over its API.

*Rationale:* the core's work is tokenizers, GGUF handling, embeddings, evaluation
harnesses, and eventually fine-tuning data pipelines — all of which are strongest
in Python. Agent logic iterates fastest there. VS Code extensions must be
TypeScript regardless, so a second language was unavoidable; this puts the seam in
the one place it was always going to be.

*Known costs, accepted:* packaging and distributing a Python daemon to end users is
genuinely awkward and will need real work (frozen binary or bundled runtime). The
GIL constrains in-process concurrency, which is mitigated by the engine running as
a separate process (see D3).

*Rejected:* TypeScript everywhere (no serious local-embedding or fine-tuning path;
would end up shelling out to Python anyway); Rust core (best long-term distribution
and performance story, but slowest iteration on agent logic and weakest ML
ecosystem — revisit only if daemon performance or distribution becomes the binding
constraint); Python everywhere (not achievable — VS Code requires TypeScript).

### DECIDED — D3: Inference sits behind a `ModelBackend` adapter over a managed engine process

Solara defines its own `ModelBackend` interface. The first implementation manages a
`llama-server` child process and speaks to it over HTTP, using **both** its
OpenAI-compatible endpoint and its **raw completion endpoint**.

Retaining raw access is the load-bearing part of this decision. It preserves:

- **full prompt-template control**, so Solara's identity and tool protocol are not
  at the mercy of a model's baked-in chat template;
- **grammar-constrained decoding (GBNF)**, which is the main lever for making a 7B
  model reliably emit well-formed tool calls;
- explicit control over sampling, stop sequences, and KV-cache reuse.

*Rationale:* satisfies C3 — a new engine is one new adapter. Process separation
means an engine crash or OOM does not take down Solara, and sidesteps the GIL cost
from D2.

*Rejected:* in-process bindings (an engine segfault kills the daemon; engine
upgrades become build problems; couples the core to one engine's API shape);
OpenAI-compatible HTTP only (maximum swappability, near-zero engine code — but
surrenders grammar constraints and raw prompt control, which is precisely the
leverage small models need).

### DECIDED — D4: Remote backends are first-class; local is the default

Local and remote models implement the same `ModelBackend` interface. Local
inference is the default and the target of optimisation; remote endpoints are a
supported backend, not a special case.

*Rationale:* agent logic cannot be developed or evaluated at 4 tok/s. A strong
remote model also provides the reference behaviour to measure small local models
against, and is the practical source of fine-tuning data for D5's long-term goal.

*Accepted risk:* designing for capabilities that small local models do not have.
Mitigated by treating backend capability as declared and negotiated (see D-open-5)
and by evaluating on the local target model, not the remote one.

*Rejected:* local-only by principle (development gated on CPU throughput, no
baseline); remote-for-development-only (the distinction is policy, not
architecture — the code path exists either way).

### DECIDED — D5: Mode and model selection are a routing concern

Following P4, a behavior mode such as Uncensored Mode is **not** a system-prompt
variant. A mode selects a **model profile**: weights, prompt template, sampling
parameters, identity layer, and tool policy, together.

This generalises beyond modes. The same mechanism carries the hardware-driven need
to route different tasks to different model sizes — a small fast model for
classification, summarisation and mechanical edits; a larger one for planning and
reasoning — rather than assuming one model serves every call.

*Consequence:* a profile/router layer sits between the agent and the backends.
Its exact shape is open.

---

## Open decisions

Recorded here so the design cannot silently skip them. Roughly in dependency order.

| # | Decision | Why it matters |
|---|---|---|
| O1 | **Tool-call protocol** — native function calling, grammar-constrained output, or a parsed text protocol. | Determines whether a 7B model can drive tools reliably, and whether the mechanism survives changing models (C2). |
| O2 | **Agent loop shape** — single loop, plan-then-execute, adaptive loop with revisable plan, or a hierarchy. | Determines how multi-step work is structured and how many model calls a task costs. |
| O3 | **Tool system organisation** — flat registry, capability providers, or primitives plus composed skills. Includes how tools are *exposed*: every tool schema in the prompt is a token cost and a chance to choose wrong. | Tool count is a real cost on small models with limited context. |
| O4 | **Security enforcement point** — where policy is evaluated relative to the tool layer. | C6. Must be a single mediation point in the core so every interface inherits it and no client can bypass it. |
| O5 | **Backend capability negotiation** — how a backend declares support for grammars, native tool calls, context length, etc. | Lets D3/D4 coexist without lowest-common-denominator behaviour. |
| O6 | **Context assembly** — what decides which files, symbols, results and history enter the prompt. | Highest-value subsystem in a coding AI and the easiest to underbuild. |
| O7 | **Repository intelligence** — indexing, symbol extraction, retrieval strategy. | Required for large codebases; depends heavily on O6. |
| O8 | **Memory** — what persists, at what scope, and how it re-enters context. | Distinct from context; must not become an unbounded prompt tax. |
| O9 | **Event system** — whether one is warranted, and what it is genuinely for. | Named in the original vision, but must justify itself rather than be assumed. |
| O10 | **Identity expression** — how Solara's identity is enforced across swappable models. | C4. Must be stronger than a name in a system prompt. |
| O11 | **Session and workspace model** — what a session is, what it owns, how it persists. | Underpins memory, permissions, and the daemon API surface. |
| O12 | **Daemon API protocol** — request/response shape, streaming, interruption, approval round-trips. | The contract all three interfaces depend on; expensive to change later. |

---

## Change history

| Date | Change |
|---|---|
| 2026-08-09 | Log created. D0–D5 recorded. Repository cleared; prior Terms of Service draft removed, with its architectural implications extracted into "Product-shape constraints" above. |
