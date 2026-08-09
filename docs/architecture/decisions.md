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
| C8 | Solara operates on **workspaces, not repositories**. A workspace is one or more roots — a single file, a loose folder, a non-versioned project, a multi-root editor workspace, or a repository. Version control is a capability that activates when present, never a prerequisite. |

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

### DECIDED — D6: Tool calls use a Solara-defined text protocol, optionally grammar-constrained

Solara defines its own tool-call syntax as delimited blocks in the model's output
stream, parsed tolerantly with repair-and-retry on malformed emissions. Where a
backend declares grammar support (see O5), the *same* protocol is additionally
constrained by a generated GBNF grammar so that malformed output becomes
structurally impossible.

*Rationale:* this is the only option that satisfies C2 unconditionally. The
protocol is a property of Solara, not of the model's chat template, so swapping
models does not change how tools are invoked. Grammar constraint is then a pure
optimisation on backends that support it, rather than a dependency.

*Rejected:* native function calling (reliability at 7B is mediocre and varies per
model, so tool behaviour would change whenever the model changes); grammar-only
(llama.cpp-specific, and would force remote backends onto an entirely separate
code path, breaking D4).

*Consequence:* Solara owns a tool-call parser and its repair strategy. Malformed
tool calls are a normal, expected event to be handled, not an error state.

### DECIDED — D7: One adaptive agent loop with a revisable plan and ephemeral sub-sessions

A single loop owns each task. Two things make it more than a plain ReAct loop:

**A revisable plan.** The task list is explicit, persistent state that the model
rewrites as it learns. Plans authored before reading the code are usually wrong;
the ability to revise is what makes planning worth doing at all. It also makes
progress legible to the user and makes a task resumable.

**Ephemeral sub-sessions.** The loop can delegate a sub-task to a context-isolated
child session which runs with a fresh context, completes, returns a compact
result, and is then evicted from the parent's context.

This is the primary defence against context exhaustion, which is the binding
constraint on small models — not intelligence. Exploration is what destroys
context: answering "where is authentication handled?" may read thousands of tokens
to produce one line. Done in a sub-session, only the line survives in the parent.

Three rules govern sub-sessions:

1. **Evicted from context, retained in the session store.** A sub-session is
   removed from the parent's prompt but persisted — auditable, resumable, and
   addressable by ID. Summarisation is lossy; destroying the source means work
   must be redone when the summary proves insufficient.
2. **Delegation is selective, never reflexive.** Each sub-session re-pays a startup
   cost of system prompt, identity, tool schemas and briefing. On the current
   hardware envelope, prompt processing for a 7B Q4 runs on the order of 20–40
   tok/s, so a 4k-token briefing costs roughly 100–200 seconds before the first
   generated token. Delegation therefore pays off only for work that **reads a lot
   and returns a little**. Delegating a task that returns a large diff is a net
   loss.
3. **Invisible is not unsupervised.** Approval requests raised inside a sub-session
   must escape to the real user (see D9). A hidden conversation must never become
   an implicit auto-approval.

*Rejected:* plain ReAct (small models drift over long horizons; no resumability, no
progress visibility, hard to debug); plan-then-execute with a fixed plan (cannot
recover when the plan turns out wrong, which is the common case); a standing
multi-agent orchestrator/worker hierarchy (multiplies model calls unconditionally,
which the hardware envelope cannot absorb — D7 gets the context-isolation benefit
on demand instead of by structure).

*Product consequence:* sub-sessions consume tokens the user never sees. Under the
credits system (P2), invisible work is still billable work, so accounting must
attribute sub-session cost to the parent task and surface it. Silent billing for
hidden work is a support problem waiting to happen.

### DECIDED — D8: A small core toolset, capability providers, and scoped exposure

Three layers:

**Core tools — always present, deliberately few.** `read_file` (line-ranged),
`edit_file` (targeted), `write_file`, `list_directory`, `search`, `run_command`,
and `delegate` (spawn a sub-session per D7), plus plan revision. This set is fixed
and small because at 7B every additional tool schema is both token cost and an
opportunity to choose wrong.

**Capability providers — registered, not hardcoded.** Git, test runners, package
managers, build systems, and later external MCP servers register their tools with
the registry. Adding a capability never means editing the agent.

**Scoped exposure — only a relevant subset reaches the prompt.** Tool schemas are
paid for on every turn. Exposure is scoped by active providers and current task
phase. Deliberately *not* an ML relevance model — that is over-engineering a
problem that activation rules solve.

*Rationale:* satisfies expandability without the prompt growing without bound, and
keeps the same design working from a 7B local model up to much larger ones.

*Rejected:* a flat always-exposed registry (thirty tools is thousands of tokens of
schema every turn and measurably worse tool selection on small models);
primitives-only, composing everything through the shell (tiny prompt and genuinely
flexible, but it collapses security granularity — `git status` and
`git push --force` become indistinguishable opaque strings to the policy engine,
which is incompatible with C6 and D9).

### DECIDED — D9: A single policy mediation gate in the core

Every tool invocation — from any interface, at any sub-session nesting depth —
passes through one policy engine before execution. The engine resolves each call to
**allow**, **deny**, or **ask the user**, evaluated against:

- **workspace roots** — filesystem access is jailed to declared roots; path
  traversal and symlink escapes are resolved before the check, not after;
- **command risk classification** — commands are classified rather than
  pattern-matched on raw strings;
- **the acting principal** — the local implicit owner today (D0), an authenticated
  Rubby Account under a hosted deployment.

It lives in the core, below the daemon API, so all three interfaces inherit it
identically and no client can bypass it by constructing its own request.

Two rules bind this to D7:

- **Sub-sessions inherit a subset of parent permissions, never a superset.**
  Delegation must not be a privilege-escalation ladder.
- **Approval requests propagate to the attached human**, escaping sub-session
  boundaries. If no interface is attached to answer, the default is deny, not
  allow.

OS-level sandboxing (containers, seccomp, bubblewrap) is designed as a **pluggable
enforcement backend** behind this same gate rather than an alternative to it. It
becomes necessary for hosted multi-tenant deployment (D0/P3) and remains optional
locally.

*Rationale:* one auditable choke point is the only structure that satisfies C6 as
a property rather than a practice.

*Rejected:* per-tool permission checks (inconsistent, trivially forgotten when
adding a tool, no central audit or policy view); OS sandbox alone (strong
isolation, but binary — it provides no basis for "ask the user first", which is the
interaction the product actually needs, and it is awkward on Windows).

### DECIDED — D10: Backends declare capabilities; strategies are selected, never assumed

Every `ModelBackend` exposes a declarative capability descriptor: usable context
length, maximum output length, grammar-constrained decoding support, native
tool-calling support, streaming, prefix/KV-cache reuse, and how tokens are counted
for that model.

Solara reads the descriptor and selects strategies accordingly — most importantly,
whether D6's tool protocol is additionally grammar-constrained, and what token
budget D11 assembles against. Every capability has a working fallback, because D6's
text protocol functions on any backend.

*Rationale:* this is what lets D3 and D4 coexist without collapsing to
lowest-common-denominator behaviour. A llama.cpp backend gets grammar constraints;
a remote backend does not, and neither needs a separate code path.

*Rejected:* a runtime negotiation handshake (complexity with no benefit — backend
capabilities are static facts, not negotiated state); assuming a fixed capability
set (breaks the moment a second backend exists).

### DECIDED — D11: Context is assembled from prioritised segments around an explicit working set

The prompt is not a message log. It is **assembled every turn** from ordered
segments, each with a priority and a token budget:

| Priority | Segment | Notes |
|---|---|---|
| 1 | Identity layer | Always present; never evicted (D14). |
| 2 | Tool schemas | Scoped per D8, not the full registry. |
| 3 | Plan state | The revisable task list from D7. |
| 4 | Workspace memory | Bounded excerpt, not the whole file (D13). |
| 5 | **Working set** | Files and symbols currently relevant, line-ranged and deduplicated. |
| 6 | Conversation history | Recent turns; evicted oldest-first under pressure. |
| 7 | Current request | Always present. |

The **working set** is the load-bearing idea. File content lives there and *only*
there — conversation history references it rather than repeating it. A file read
three times occupies context once. Files enter the working set when the task needs
them and leave when it moves on, which makes "what can the model currently see" an
explicit, inspectable property of the system rather than an emergent accident of
message order.

The assembler derives its total budget from the backend's declared context length
(D10), reserving headroom for output. Eviction is **by priority, then recency** —
never recency alone.

*Rationale:* deduplication and priority-based eviction are what make a 32k window
behave like a much larger one. This is also what D7's sub-sessions plug into: a
sub-session returns a compact result that enters the parent's working set, while
everything it read never does.

*Rejected:* chronological history alone (the same file occupies context once per
read; important early facts are evicted merely for being old); summarise-and-compact
on overflow (lossy exactly when it matters, and a compaction pass on a 7B at ~4
tok/s costs minutes of visible stall — retained only as a last-resort fallback, not
the primary strategy).

### DECIDED — D12: Workspace intelligence is structural first, tiered by size, and VCS-agnostic

Per C8 the unit is a **workspace**, not a repository. Understanding is built from
**structural analysis** — tree-sitter parsing into a symbol index (definitions,
references, imports, file structure) plus fast literal and regex search — rather
than semantic embeddings.

Scaling is **tiered**, so cost matches the workspace:

| Tier | Workspace | Strategy |
|---|---|---|
| 0 | Single file or a handful | No index. Direct read. |
| 1 | Small project | On-demand structural scan, held in memory, not persisted. |
| 2 | Large workspace | Persistent incremental index, built lazily in the background, invalidated by content hash. |

Git is a **capability provider** (D8) that activates when version control is
detected. It is never required, and nothing in indexing, memory, or the agent loop
may assume it exists. Non-code files — Markdown, configuration, data — are indexed
as text; languages without a grammar degrade to plain-text search rather than
failing.

*Rationale:* code lookup is overwhelmingly *exact* — a symbol, a function name, a
string literal — which is precisely where structural search beats vector similarity.
It is also incremental, debuggable, offline, and costs no additional RAM, which
matters when a 7B model already occupies roughly a third of a 16 GB machine.

*Deferred, with a seam:* retrieval sits behind a `Retriever` interface so semantic
embeddings can be added later as an additional implementation for vague conceptual
queries, without disturbing the structural path.

*Rejected:* embeddings-only (worse than search for the exact-match lookups that
dominate coding work, costly to keep fresh, and hard to debug when it retrieves the
wrong chunk); structural + embeddings now (a second resident model on constrained
hardware, for a capability that is not yet the bottleneck); no index at all
(rediscovers the same structure repeatedly — though this is exactly what Tier 0
does, correctly, for small workspaces).

### DECIDED — D13: Memory is curated, human-readable, and bounded

Three distinct stores, deliberately not one system:

**Workspace memory** — a human-readable Markdown file holding conventions,
architecture notes, decisions, and workspace-specific preferences. It lives at
`.solara/memory.md` inside the workspace when writable, so it is diffable and — if
the workspace is version-controlled — committable at the user's discretion. When
the workspace is read-only or unsuitable, it is mirrored in Solara's own store keyed
by workspace identity. Per C8, this must not depend on version control.

**User memory** — global preferences that follow the user across workspaces.

**Session store** — durable sessions and sub-sessions from D7. History, not
knowledge.

Solara writes memory through the ordinary edit path, so changes are visible,
reviewable, and reversible rather than silent. Memory is **budgeted**: it competes
for context under D11 and cannot grow without bound. Growth beyond budget is a
prompt to curate, not a reason to build retrieval.

*Rationale:* the failure mode of automated memory is a wrong fact persisting
invisibly and corrupting every later session. A file the user can open, read, and
correct makes that failure visible and fixable. It also requires no retrieval
infrastructure and no second model.

*Rejected:* an opaque structured fact database (scales further, but wrong facts
persist where the user cannot see them); vector memory over past sessions (another
resident model, unpredictable recall, stale recollections with no user recourse);
deferring memory entirely (Solara would relearn the same workspace facts every
session — the cost is real and the file-based form is cheap enough to justify now).

### DECIDED — D14: Identity is a structured artifact, enforced at three levels

Solara's identity is a versioned, structured artifact — **not** a prompt string:

- **core identity** — what Solara is;
- **coding philosophy** — how it approaches software;
- **hard rules** — non-negotiable behaviours;
- **output conventions** — how it presents code, diffs, and explanations;
- **tool-use discipline** — how it works, not just what it says.

It is enforced at three levels, which is what makes C4 real rather than aspirational:

1. **Composition.** The artifact is rendered into the prompt per model profile
   (D5). Each profile may adapt the rendering to its model's template, but the
   source of truth is one artifact.
2. **Runtime.** The tool protocol (D6) and the policy gate (D9) constrain what
   Solara can actually *do*, independent of what any model says. Behaviour that
   matters is enforced by the system, not requested of the model.
3. **Evaluation.** An eval suite asserts that Solara behaves as Solara on any
   backend. Swapping models becomes a measurable change rather than a silent
   behavioural drift.

**Uncensored Mode (P4) is a model profile, not a hole in the system.** It selects
different weights and a relaxed content rule set. It does **not** relax the policy
gate, the workspace jail, or the approval requirements of D9 — those are structural
properties of Solara, not model behaviour, and no profile can switch them off.

*Long-term:* fine-tuned Solara-specific models are where identity ultimately lives.
The eval suite from level 3 doubles as the validation harness for that work, and
real sessions accumulate the training data. This decision makes that a continuation
of the architecture rather than a replacement of it.

*Rejected:* a system prompt alone (precisely the "name above a model" outcome C4
rules out; drifts silently on model change with nothing to catch it); prompt-now
fine-tune-later with nothing in between (honest about the destination, but leaves
identity unenforced for the entire period in which the project is actually built).

### DECIDED — D15: A typed outbound event stream, but no internal message bus

The original vision named an "event system." It resolves into two separate things,
and only one of them is warranted.

**Adopted — an outbound activity stream.** The daemon emits a typed, versioned
stream of everything happening inside a task: token deltas, tool call started and
finished, approval requested and resolved, plan revised, sub-session opened and
closed, errors, and usage accounting. This is not optional — under D1 the
interfaces are thin, and a thin client can only render what it is told. The stream
is part of the API contract (D17).

**Rejected — an internal publish/subscribe bus.** Decoupling core components
through an internal bus makes control flow untraceable and failures hard to
attribute, in exchange for flexibility the system does not yet need. Components
call each other directly.

**The seam that satisfies the original intent:** observability consumers — logging,
tracing, metrics, and later credit accounting (P2) — subscribe to the *same* typed
stream that goes outbound. Different parts of Solara can therefore react to what
happens inside it without inverting control flow. If an internal bus later proves
necessary, this stream is the natural place it grows from.

### DECIDED — D16: Workspaces, sessions, and sub-sessions are distinct persisted entities

**Workspace** — identified by its normalised absolute root paths, plus version
control remote identity when one exists (per C8, it often will not). Owns the
roots, the memory file (D13), the structural index (D12), the permission policy
(D9), and the set of active capability providers (D8). Long-lived and shared:
several sessions may attach to one workspace and reuse its index and memory.

**Session** — a conversation and task thread bound to exactly one workspace. Owns
the plan (D7), the working set (D11), conversation history, and its granted
permissions. Persisted and resumable, so a task interrupted at 4 tok/s is not lost.

**Sub-session** — a child of a session per D7. Owns its own context, inherits a
subset of the parent's permissions, is persisted independently, and is evicted from
the parent's context on completion while remaining addressable by ID.

Every session has an owning **principal** — the single implicit local owner today,
an authenticated account under a hosted deployment (D0).

### DECIDED — D17: HTTP for operations, WebSocket for the event stream, versioned from day one

Request/response operations over HTTP; the D15 activity stream over WebSocket.
Bound to loopback only, authenticated by a per-session bearer token stored with
restricted file permissions — the D1 security consequence, restated because it is
easy to lose during implementation.

The surface covers session lifecycle (create, resume, list, delete), sending a
request, **interruption**, approval responses, workspace management, and model
profile selection (D5).

Two properties are load-bearing rather than incidental:

- **Interruption is first-class.** At 3–6 tok/s a user will stop a running task
  constantly. Interruption that only takes effect between model calls is not good
  enough; it must cancel mid-generation and leave the session in a resumable state.
- **Approvals round-trip over the stream** with a correlation ID, originating at
  any sub-session depth (D9). No attached interface, or an expired request, means
  **deny** — never allow.

The API is versioned from the first commit, because three interfaces will depend
on it and they will not upgrade in lockstep.

### DECIDED — D18: Edits are applied as validated search/replace blocks

The model emits the exact existing text and its replacement. Solara applies the
change only on an exact match, and validates the result before committing it.

*Rationale — this is the decision most constrained by the hardware envelope.* Full
file rewrites are the most reliable format for small models, but the output cost is
disqualifying: a 500-line file is roughly 6,000 output tokens, which at 3–6 tok/s is
**15–30 minutes for one edit**. Unified diffs are compact but require line
arithmetic and hunk headers that small models get wrong frequently. Search/replace
blocks are compact (only the changed region is emitted), require no line numbers,
and are *verifiable before application* — an inexact match fails loudly instead of
corrupting the file. They also fit D6's text protocol directly and can be
grammar-constrained under D10.

**Validation uses the D12 parser.** Because tree-sitter is already present for
indexing, every edit is parse-checked after application; an edit that breaks the
file's syntax is rejected and reported rather than written. This is a genuine
benefit of choosing structural indexing in D12.

**Failure handling is defined, not incidental.** A failed exact match triggers a
re-read of the region and one retry against fresh content. Repeated failure
escalates to rewriting only the enclosing function or block — never the whole file.
Edits are staged and applied atomically per tool call, with the original retained
for undo.

### DECIDED — D19: Verification is a phase of the loop, tiered cheapest-first

A coding agent is distinguished from a code generator by whether it checks its own
work. Verification is therefore an explicit phase of D7's loop, escalating only as
far as needed:

| Tier | Check | Cost |
|---|---|---|
| 1 | Syntax/parse via tree-sitter | Instant, already available (D12) |
| 2 | Type check or lint, if the workspace configures one | Seconds |
| 3 | Targeted tests — only those plausibly affected | Moderate |
| 4 | Full suite, build, or running the program | Expensive; on request or at task completion |

**Failure output is triaged, never dumped.** Build and test logs are enormous and
pasting one raw would destroy the context D11 works to protect. Large failures are
triaged in a sub-session (D7) — the ideal case for delegation, since it reads a lot
and returns a little — which returns the essential diagnosis to the parent's working
set.

The runner is a **capability provider** (D8) that detects the workspace's toolchain
rather than hardcoding one, consistent with C8: many workspaces have no test runner
at all, and that must be an ordinary condition rather than an error.

---

## Open decisions

Recorded here so the design cannot silently skip them. Roughly in dependency order.

| # | Decision | Why it matters |
|---|---|---|
| ~~O1~~ | Tool-call protocol | **Closed by D6.** |
| ~~O2~~ | Agent loop shape | **Closed by D7.** |
| ~~O3~~ | Tool system organisation | **Closed by D8.** |
| ~~O4~~ | Security enforcement point | **Closed by D9.** |
| ~~O5~~ | Backend capability negotiation | **Closed by D10.** |
| ~~O6~~ | Context assembly | **Closed by D11.** |
| ~~O7~~ | Repository intelligence — *reframed as workspace intelligence per C8* | **Closed by D12.** |
| ~~O8~~ | Memory | **Closed by D13.** |
| ~~O9~~ | Event system | **Closed by D15.** |
| ~~O10~~ | Identity expression | **Closed by D14.** |
| ~~O11~~ | Session and workspace model | **Closed by D16.** |
| ~~O12~~ | Daemon API protocol | **Closed by D17.** |
| ~~O13~~ | Edit application strategy | **Closed by D18.** |
| ~~O14~~ | Verification loop | **Closed by D19.** |
| ~~O15~~ | First implementation milestone | **Closed by D20.** |

All decisions identified so far are closed. The formal architecture derived from
them is in [`architecture.md`](./architecture.md).

New open decisions are expected to appear during implementation and should be
added here rather than settled silently in code.

### DECIDED — D20: The first milestone is one narrow vertical slice, end to end

The first implementation is **not** a layer-by-layer build. It is the thinnest path
that touches every layer: a terminal client → daemon → session → agent loop → tool
protocol → policy gate → core tools → llama.cpp backend, running a real task against
a real workspace on the current hardware.

Concretely: *read a file, make one search/replace edit, verify it parses, report
back* — with a 1.5B or 3B model first, because iteration speed matters more than
capability while the plumbing is being proven.

*Rationale:* the architecture's risky assumptions are all at the seams — whether a
small model can drive the D6 protocol, whether D18 edits apply cleanly, whether D11
assembly stays inside budget, whether the loop is bearable at local speeds. A
vertical slice tests all of them in days. A horizontal build tests none of them for
months, and every one of those assumptions is cheaper to correct early.

*Deliberately excluded from the first slice:* sub-sessions, the persistent index,
memory, capability providers beyond the core tools, and every interface except the
terminal client. Each has a defined seam and is added once the spine is proven.

---

## Change history

| Date | Change |
|---|---|
| 2026-08-09 | Log created. D0–D5 recorded. Repository cleared; prior Terms of Service draft removed, with its architectural implications extracted into "Product-shape constraints" above. |
| 2026-08-09 | D6–D9 recorded, closing O1–O4. Tool-call protocol, agent loop with ephemeral sub-sessions, tool organisation, and the policy mediation gate are settled. |
| 2026-08-09 | **C8 added** — Solara targets workspaces, not repositories; version control is a capability, not a prerequisite. O7 reframed accordingly. D10–D14 recorded, closing O5–O8 and O10. O13–O15 added: edit application, verification, and the first milestone were missing from the log. |
| 2026-08-09 | D15–D20 recorded, closing O9 and O11–O15. The event system resolves into an outbound stream with no internal bus; sessions, the API, edit application, verification, and the first milestone are settled. Formal architecture written to `architecture.md`. |
