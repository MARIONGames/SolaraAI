# Execution Graph and Scheduler

Detail for [D21](./decisions.md). Supersedes the loop structure of D7; retains its
plan semantics, sub-session rules, and delegation economics.

---

## 1. Why a graph

A loop encodes one assumption: that the next thing to do cannot begin until the
previous thing has finished. That is true of *dependent* work and false of most
of the rest. Reading three files to understand a change, running a type check while
drafting an edit, or evaluating two competing approaches are all naturally
concurrent, and a loop cannot express any of them.

A graph expresses dependency explicitly and lets everything else run when resources
allow. **A loop is a graph whose width is one** — so setting concurrency to 1
reproduces the previous design exactly. That property is what makes this safe to
adopt on a machine that cannot yet exploit it.

---

## 2. Work units

A work unit is the scheduling atom.

```
WorkUnit
  id              stable identifier, referenced by dependents
  kind            explore | plan | edit | verify | triage | review | synthesize
  goal            natural-language statement of what this unit must achieve
  depends_on      [unit_id]        hard ordering constraints
  inputs          [reference]      working-set entries, prior unit results
  context_policy  inherit | isolated
  result_contract what the unit must return, and its size budget
  resource_class  tier hint for the model router (D22)
  permissions     subset of the parent's grants (D9)
  budget          tokens, wall-clock, tool calls
  status          pending | ready | running | done | failed | cancelled | superseded
```

Two fields carry most of the weight.

**`context_policy`** decides whether a unit runs inside the parent's context or in
an isolated one. `isolated` is D7's sub-session, unchanged: the unit runs with a
fresh context, returns a compact result, is evicted from the parent's prompt, and is
retained in the session store. The delegation economics from D7 still govern the
choice — isolation pays only for work that **reads a lot and returns a little**.

**`result_contract`** is what makes isolation safe. A unit declares in advance what
it will return and how large that may be — "the file and line range implementing
X, under 200 tokens." Without a contract, an isolated unit is free to return
everything it read, which defeats the purpose.

---

## 3. The graph is the plan

D7 kept a revisable plan as separate state. That created a second artifact capable of
drifting out of sync with what was actually executing.

Here they are the same object. The plan is the graph; revising the plan means
mutating the graph:

- **add** a unit, with dependencies
- **supersede** a unit — replaced by one or more others; the original is retained as
  `superseded`, never deleted, so history stays intelligible
- **cancel** a subtree that is no longer needed
- **re-open** a completed unit whose assumptions were invalidated

Mutations are made through a tool (D8), pass the policy gate (D9), and are emitted
on the event stream (D15), so the user watches the plan evolve in real time rather
than receiving a summary afterwards.

**Invariant:** the graph must stay acyclic. A mutation that would introduce a cycle
is rejected and returned to the model as a tool error.

---

## 4. The scheduler

```
loop:
    ready   = units whose dependencies are all `done`
    ranked  = rank(ready)
    for unit in ranked:
        if admits(unit):  start(unit)
    await any completion
    apply graph mutations produced by completed units
```

### 4.1 Admission control

A unit starts only if every budget it needs is available.

| Resource | Constraint |
|---|---|
| Concurrency slots | Hard ceiling from the hardware profile. |
| KV-cache memory | Each concurrent context consumes cache; see §5. |
| Model slots | Which tiers are loaded, and their batching capacity. |
| Task budget | Remaining tokens, wall-clock, tool calls, credits. |
| Execution locality | Tool-executing units must run where the workspace is (D26). |

### 4.2 Ranking

When more units are ready than can run, rank by:

1. **Critical path length** — units with the most work depending on them first.
2. **Information value per token** — exploration that unblocks planning outranks
   speculative work.
3. **Cheapness** — a tier-1 unit that may resolve the question runs before a tier-3
   unit that certainly will.
4. **User-visible progress** — ties break toward work the user can see, because
   perceived responsiveness matters at local generation speeds.

### 4.3 Serialisation of writes

Reads parallelise freely. **Writes do not.** Two units editing the same file
concurrently is a correctness hazard, not a performance question.

- Every unit that may write declares its intended write set.
- The scheduler takes exclusive locks per path.
- Conflicting units are serialised, never run and merged.
- A unit whose write set was modified by another unit since it began is invalidated
  and re-planned against the new content.

---

## 5. Resource model

The scheduler's admission control is only as good as its cost model, so the cost
model is explicit.

### 5.1 KV cache

Under llama.cpp, concurrent requests are served by continuous batching and **share
one copy of the model's weights**. An additional slot therefore costs KV cache, not
another model.

Per-token KV cost, in bytes:

```
2 (K and V) × n_layers × n_kv_heads × head_dim × bytes_per_element
```

Worked for Qwen2.5-Coder-7B (28 layers, 4 KV heads via GQA, head_dim 128, f16):

```
2 × 28 × 4 × 128 × 2  ≈  57 KB per token
                      ≈  450 MB at 8k context
                      ≈  900 MB at 16k context
```

On an 8 GB card holding a 4.7 GB Q4 model, roughly 3 GB remains — about **4–6
concurrent 8k slots**. Cache quantisation to 8-bit approximately halves this at
some quality cost, and is a configuration option rather than a default.

### 5.2 Compute, and why CPU is different

Memory is not the binding constraint on the current machine — **compute is**.

On a GPU, batched decoding is close to free until the batch saturates the device,
because decoding is memory-bandwidth-bound and batching amortises weight reads
across sequences. Concurrency there is nearly a free multiplier.

On a 4-core CPU, parallel slots contend for the same cores. Two concurrent
generations do not run at full speed each; they approximately halve. Total
throughput is roughly flat while latency per unit doubles.

**Therefore the scheduler must derive concurrency from the backend's declared
compute class (D10), not from available memory.**

| Host | Default concurrency | Reason |
|---|---|---|
| CPU-only (current i7) | **1** | Compute-bound; parallelism adds latency without throughput. |
| RTX 5070, 8 GB | **3–4** | Batching is cheap; KV cache is the ceiling. |
| Larger GPU / multi-GPU | Higher, measured | Determined by profiling, not assumption. |
| Distributed inference (D26) | Sum of worker capacities | Scheduled per worker. |

---

## 6. What parallelism buys

| Pattern | Structure | Applies when |
|---|---|---|
| **Parallel exploration** | Several isolated units answer independent questions at once. | Concurrency ≥ 2. The clearest win: exploration is read-heavy and return-light. |
| **Concurrent verification** | Tier 1–2 checks run against completed edits while later edits are drafted. | Almost always — these tiers are cheap and non-model-bound. |
| **Speculative branches** | Two approaches execute in parallel; the first to pass verification wins, the loser is cancelled. | Concurrency ≥ 2 and genuine ambiguity. Costly: pay for both, keep one. |
| **Self-consistency** | Sample N solutions for one unit, select by verification. | Cheap tiers with parallel capacity (D22). |
| **Pipelined triage** | Failure analysis for one test begins while others still run. | Any multi-failure verification. |

At concurrency 1 every one of these degrades to sequential execution. Nothing
breaks; the schedule simply narrows.

---

## 7. Checkpoints, resumption, backtracking

Each completed unit is a **checkpoint**: graph state, working set, and a shadow
snapshot of files it modified (D24).

This yields three properties:

- **Resumption** — a task interrupted at 4 tok/s resumes from the last completed
  unit rather than the beginning.
- **Backtracking** — recovery can restore both graph and file state to a prior
  checkpoint and take a different branch (D24).
- **Auditability** — the full execution history, including superseded and cancelled
  units, is inspectable after the fact.

---

## 8. Failure semantics

A unit that fails does not fail the task.

1. The failure is recorded on the unit with its diagnosis.
2. Dependents are marked blocked, not failed.
3. Recovery (D24) chooses: retry with adjusted inputs, replace via supersession,
   backtrack, or escalate.
4. Only exhausting recovery options fails the task, and it fails to a **clean
   workspace state**, never a half-applied edit.

---

## 9. Relationship to earlier decisions

| Decision | Status |
|---|---|
| D7 — loop, plan, sub-sessions | Loop superseded by the graph. Plan unified *into* the graph. Sub-sessions retained as `context_policy: isolated`, with delegation economics unchanged. |
| D9 — policy gate | Unchanged and now more important: every unit's permissions are a subset of its parent's, at any depth. |
| D11 — context assembly | Unchanged per unit. Each unit assembles its own context under its own budget. |
| D15 — event stream | Extended with unit lifecycle and graph mutation events. |
| D16 — sessions | A session owns one graph; sub-sessions are isolated units within it. |
| D17 — interruption | Interruption cancels running units and preserves the graph for resumption. |
