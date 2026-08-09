# Code Intelligence

Detail for [D23](./decisions.md). Extends D12 — the structural, VCS-agnostic, tiered
approach is unchanged; the depth built on top of it is new.

---

## 1. The layers

Each layer is derived from the one below it, and each is independently useful.

```
 5  Architecture map        modules, layers, entry points, hot spots
 4  Change-impact analysis  what breaks if this changes
 3  Call graph              who calls whom
 2  Dependency graph        what imports what
 1  Symbol index            definitions, references, exports        ← D12
 0  Files and text          content-addressed, searchable
```

Layers 0–1 are D12 and sufficient for the first milestone. Layers 2–5 are this
document. A workspace may sit at any depth: a folder of loose scripts may never
justify a call graph, and that is a normal state, not a degraded one (C8).

---

## 2. Layer 2 — dependency graph

Nodes are files and modules; edges are imports, includes, and requires, extracted
from the tree-sitter parse rather than by regex.

Resolution is the difficult part and is explicitly per-language: Python's package
semantics, JavaScript's resolution algorithm plus bundler aliases, Go's modules,
Rust's crates. Each language provides a resolver; unresolvable imports are recorded
as **unresolved edges** rather than dropped, because an unresolved import is itself
information the agent can use.

Derived cheaply from this layer:

- **Reverse dependencies** — who depends on this file
- **Cycles** — import cycles, which are usually design problems worth surfacing
- **Layering violations** — once an architecture map (§5) declares layers
- **Orphans** — files nothing imports, often dead code or entry points

---

## 3. Layer 3 — call graph

Nodes are functions, methods and classes; edges are calls, instantiations and
references.

**Precision is language-dependent and must be represented honestly.** Building a
precise call graph for a dynamic language is undecidable in general: a Python call
through `getattr`, a JavaScript call through a runtime-selected property, or any
form of dynamic dispatch cannot be resolved statically.

Every edge therefore carries a confidence label:

| Confidence | Meaning | Typical source |
|---|---|---|
| `certain` | Statically resolvable | Direct call to an unambiguous, statically-typed target |
| `probable` | Resolvable by name with a single plausible match | A uniquely-named method in a dynamic language |
| `heuristic` | Name matches, target ambiguous | Overloaded or duck-typed dispatch |
| `unresolved` | A call site exists, the target is unknown | Reflection, dynamic dispatch, indirection |

The agent is given the labels. "There are three certain callers and two heuristic
ones" is a materially different fact from "there are five callers", and hiding the
difference produces confident wrong answers — the failure mode this architecture is
most concerned with.

---

## 4. Layer 4 — change-impact analysis

The payoff layer. Given a proposed change to a symbol, compute what it can affect.

```
impact(symbol, depth) -> {
    direct_callers      certain-confidence callers
    transitive          reverse-reachable set, depth-bounded, confidence-weighted
    uncertain           heuristic and unresolved edges needing human judgement
    covering_tests      tests reaching the symbol
    public_surface      whether the symbol is exported beyond its module
    blast_radius        a coarse score: contained | module | cross-module | public API
}
```

Three concrete uses:

**It makes D19's targeted testing actually targeted.** Tier 3 verification runs the
tests that *cover the changed symbols*, derived from reverse reachability rather
than guessed from file names. On a large workspace this is the difference between
a 4-second check and a 6-minute one — which at local generation speeds is the
difference between verifying every edit and verifying almost none.

**It drives critic-pass policy (D22).** A `public API` blast radius turns the critic
on; a `contained` one does not.

**It is what "understanding a large codebase" actually means.** Not retrieving a
relevant file — knowing what a change touches. This is the capability that
distinguishes editing code from editing text that happens to be code.

**Depth bounding matters.** Transitive closure over a large graph explodes; results
are depth-bounded with the truncation reported, never silently.

---

## 5. Layer 5 — architecture map

A generated, cached summary of the workspace: modules and their responsibilities,
inferred layers, entry points, public surfaces, test topology, and hot spots.

It is **generated, verified, and cached**, not written by hand:

- structural facts (entry points, module boundaries, exports) come from layers 2–3;
- prose descriptions are model-generated, one module at a time, from real structure;
- the map is regenerated on **drift** — when accumulated changes to a module exceed
  a threshold — not on every edit.

It is what a new session loads to orient itself in a large workspace instead of
re-exploring from scratch, and it is a bounded excerpt in the context assembler
(D11) rather than an unbounded prompt tax.

---

## 6. History intelligence — only when version control exists

Per C8 this layer is **conditional**. It activates when a VCS is detected and is
absent otherwise, which must be an ordinary state.

| Signal | Derived from | Use |
|---|---|---|
| **Churn hotspots** | Change frequency per file | High-churn plus high-impact is where bugs live |
| **Co-change coupling** | Files that historically change together | Catches coupling the import graph cannot see — the strongest signal here |
| **Ownership** | Authorship concentration | Who to attribute conventions to |
| **Recency** | Last modified | Recently-changed code is likelier to be relevant to the current task |
| **Change rationale** | Commit messages touching a symbol | Why the code is the way it is |

Co-change coupling is the most valuable of these because it is invisible to static
analysis: two files with no import relationship that always change together are
coupled by something real, and Solara would otherwise never know.

---

## 7. Incrementality

Nothing above survives contact with a real workspace unless it updates cheaply.

**Content addressing.** Every file is keyed by content hash. An unchanged hash means
no re-parse.

**Partial invalidation.** A changed file invalidates its own symbols, edges incident
to it, impact results that traverse it, and the architecture map for its module —
not the whole index.

**Background rebuild.** Indexing runs off the critical path. A query against a
stale region either waits briefly or answers with a staleness marker; it never
blocks a task indefinitely.

**Schema versioning.** The persistent index carries a version. A version change
triggers a rebuild rather than a subtly-wrong result from a stale schema.

**Storage.** SQLite: one file, transactional, no server, fast enough at this scale,
and trivially deletable — which matters for P5.

---

## 8. Tiering, revisited

D12's tiers extend to the deeper layers, so cost continues to match the workspace.

| Tier | Workspace | Layers built | When |
|---|---|---|---|
| 0 | A file, or a handful | 0 | Never indexed; read directly |
| 1 | Small project | 0–1 | On demand, in memory, discarded |
| 2 | Substantial project | 0–3 | Persisted, incremental, background |
| 3 | Large codebase | 0–5 | Persisted, plus generated architecture map |

Tier is chosen from file count, total size, and language mix, and can be overridden.
Building layer 5 for a 200-line project is waste; not building it for a
200,000-line one is negligence.

---

## 9. The seam that stays open

Retrieval remains behind the `Retriever` interface from D12. Semantic embeddings are
still deliberately unbuilt — but the reason is now sharper: with layers 2–4
available, most queries that would have needed semantic search resolve
*structurally*, and structurally means exactly rather than approximately.

The case that survives is genuinely conceptual search over prose — documentation,
comments, commit messages — where no structural relationship exists. That is a
narrow, well-defined gap, and it is where an embedding implementation should land
if one is ever added.

---

## 10. Relationship to earlier decisions

| Decision | Status |
|---|---|
| D12 — structural, tiered, VCS-agnostic indexing | Unchanged foundation; layers 2–5 build on it. |
| D19 — verification tiers | Tier 3 becomes genuinely targeted via change-impact analysis. |
| D18 — edit application | Parse validation unchanged; impact analysis now informs which edits warrant a critic. |
| D11 — context assembly | The architecture map and impact results are bounded segments, not unbounded additions. |
| D22 — orchestration | Impact radius feeds complexity estimation and critic policy. |
| C8 — workspaces, not repositories | History intelligence is conditional; every other layer works without version control. |
