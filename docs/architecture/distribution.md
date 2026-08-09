# Distributed Execution

Detail for [D26](./decisions.md).

---

## 1. The asymmetry that defines this design

Distributed execution is usually described as though compute were uniform. It is
not, and the difference is the whole decision:

**Inference distributes easily.** A model call is a request carrying tokens and a
response carrying tokens. It holds no reference to the local filesystem. Any machine
with the weights can serve it, and the only real costs are network latency and
scheduling.

**Tool execution does not distribute.** `pytest` runs against files. `cargo build`
runs against a source tree. `git status` runs against a working copy. A tool call
is meaningless without the workspace it operates on, so it must execute where the
workspace lives.

Consequently:

| Work | Distributable | Constraint |
|---|---|---|
| Model inference | **Freely** | Weights available on the worker |
| Embedding, reranking | Freely | Same |
| Indexing and parsing | Partially | Needs file content, which can be shipped |
| Tool and command execution | **No** | Bound to the workspace filesystem |
| Edit application | **No** | Same |

The immediately valuable capability is therefore narrow and concrete: **run
inference on a stronger machine while the workspace and all tool execution stay
local.** For the stated hardware path — an i7 laptop now, an RTX 5070 machine later
— that is exactly the useful case. The laptop keeps the code and runs the commands;
the desktop runs the model.

---

## 2. Workers

A worker is a typed compute resource that declares what it can do.

```
Worker
  id            stable identifier
  kind          inference | execution | index
  endpoint      how to reach it
  capabilities  see below
  locality      which workspace roots it can access (execution workers)
  health        reachable, current load, queue depth
  trust         trusted | restricted
```

### 2.1 Inference workers

Declare the D10 capability descriptor plus scheduling facts: loaded profiles,
available profiles, KV-cache capacity, batching capacity, measured throughput, and
observed latency.

Implementations:

| Implementation | Notes |
|---|---|
| **In-process / local child** | The default; the managed `llama-server` of D3 |
| **Remote Solara worker** | Another machine running a Solara inference worker over the LAN |
| **Bare inference server** | Any reachable `llama-server`, vLLM, or compatible endpoint |
| **Cloud endpoint** | D4's remote backend, expressed as a worker |

The `ModelBackend` interface from D3/D4 already abstracts all of these. What D26
adds is a **registry** and **scheduler awareness** — which is why this is a
comparatively cheap capability rather than a rewrite.

### 2.2 Execution workers

Run tools and commands. Each declares the workspace roots it can reach, and the
scheduler will not place a unit on a worker that cannot see the unit's files.

Locally there is exactly one, implicit, with access to the workspace roots. This is
the current behaviour, unchanged.

Remote execution workers are only meaningful when the workspace is genuinely present
on them — a network mount, a synchronised copy, or a container with the repository
inside. **Solara does not solve workspace synchronisation.** It is the hosted
deployment problem from D0, and pretending otherwise is what makes distributed
architectures unbuildable.

### 2.3 Index workers

Indexing needs file content but not a live workspace, so it can be shipped. Useful
when a large first index would otherwise saturate a laptop's cores. Optional, and
the lowest-priority worker kind.

---

## 3. Scheduling across workers

D21's scheduler gains a placement step.

```
for unit in ranked_ready_units:
    candidates = workers matching unit.kind
                 ∩ workers satisfying unit.locality
                 ∩ workers with the required profile available
                 ∩ healthy workers
    if candidates empty: leave pending
    else: place on best(candidates)
```

`best` accounts for measured throughput, queue depth, round-trip latency, whether
the required profile is already **loaded** (loading a model is expensive enough to
dominate the decision for short units), and cost where a worker is metered.

Two rules keep placement honest:

- **Locality is a hard constraint, never a preference.** A unit that touches files
  runs where those files are, full stop.
- **Short units prefer local.** Network round-trip can exceed the total cost of a
  small tier-1 call. Remote workers pay off for substantial generation, not for
  classification.

---

## 4. Failure handling

Distribution introduces failure modes that a single process does not have, and each
needs a defined response.

| Failure | Response |
|---|---|
| Worker unreachable at placement | Mark unhealthy, place elsewhere; if none, run local or leave pending |
| Worker dies mid-unit | The unit fails; D24 retries, normally on a different worker |
| Worker slow / queue growing | Deprioritise in `best`; do not migrate running units |
| Network partition | Fall back to local workers; the task continues degraded rather than stopping |
| Worker returns malformed output | Treated as a protocol failure (D6), not a transport failure |

**The system must remain fully functional with zero remote workers.** Distribution is
an accelerator, never a dependency. If the desktop is off, the laptop still works —
slower.

---

## 5. Transport and trust

A remote inference worker is a machine that will execute whatever prompt it is sent
and return whatever it produces. On a home LAN that is a modest risk; it is still a
real one.

- **Authenticated.** Workers require a shared key; unauthenticated workers are not
  contacted.
- **Encrypted off-host.** TLS for anything leaving the machine. Prompts contain the
  user's source code.
- **Explicitly registered.** No discovery protocol, no automatic enrolment. A user
  adds a worker deliberately.
- **Trust-classified.** A `restricted` worker may serve inference but is never sent
  workspace memory or identity internals it does not need.
- **The policy gate stays local.** Tool authorisation is evaluated on the machine
  that owns the workspace, never delegated to a worker. A compromised or misbehaving
  inference worker can produce bad output; it cannot authorise a command.

That last point is the security-critical one: D9's mediation gate does not move.
Distribution changes where tokens are generated, not who decides what may run.

---

## 6. What this enables, concretely

**Today.** One machine. One implicit inference worker, one implicit execution
worker. Identical to the pre-D26 behaviour.

**With the RTX 5070 desktop.** The laptop holds the workspace, runs all tools, and
enforces policy. The desktop runs a Solara inference worker serving the 7B profile
with speculative decoding and 3–4 concurrent slots. The laptop keeps a 1.5B locally
for tier-1 work where a network round-trip would dominate. The user works on the
laptop and gets desktop-class inference — which is the practical payoff of this
entire decision.

**Later, hosted.** Inference workers become a managed pool; execution workers become
per-principal sandboxes with synchronised workspaces. That is the D0 seam, and the
worker abstraction is what it will attach to.

---

## 7. Relationship to earlier decisions

| Decision | Status |
|---|---|
| D3 — backend adapter | Unchanged; a remote worker is a backend behind the same interface. |
| D4 — remote backends first-class | Generalised: a cloud endpoint is one kind of inference worker. |
| D9 — policy gate | **Explicitly not distributed.** Authorisation stays with the workspace owner. |
| D10 — capabilities | Extended with throughput, latency, load, and loaded-profile state. |
| D21 — scheduler | Gains placement; locality is a hard constraint. |
| D24 — resilience | Worker failure is an ordinary unit failure with an existing recovery path. |
| D0 — hosted seam | The worker abstraction is where hosted deployment attaches. |
