# Model Orchestration

Detail for [D22](./decisions.md). Extends D5, which selected a model profile but did
nothing further with it.

---

## 1. Four mechanisms, deliberately distinct

These are routinely conflated and behave completely differently. They compose, but
each has its own cost model and its own failure mode.

| Mechanism | Level | Buys | Costs |
|---|---|---|---|
| **Speculative decoding** | Inference | Raw throughput on one request | A draft model resident in memory |
| **Cascade** | Task | Fewer expensive calls | A verification step, plus latency on escalation |
| **Critic pass** | Quality | Fewer bad edits reaching disk | Roughly doubles the edit path's cost |
| **Self-consistency** | Quality | Better answers from cheap models | N× tokens; requires parallelism |

---

## 2. Speculative decoding

A small draft model proposes *k* tokens; the target model verifies them in a single
forward pass. Accepted tokens are kept, and generation continues from the first
rejection.

**Why it fits code particularly well:** acceptance depends on how predictable the
next tokens are, and code is far more predictable than prose — indentation, closing
brackets, repeated identifiers, boilerplate. Acceptance rates on code are typically
well above general text.

**Realistic gains.** On GPU, decoding is memory-bandwidth-bound and verification is
nearly free, so gains are large — commonly around 2–3× with a well-matched draft.
On CPU, the verification pass is itself compute-bound, so gains are real but
smaller. **This must be measured on the target machine rather than assumed**; the
eval harness (D25) is the place to measure it.

**Requirements and costs.**

- The draft model must share the target's vocabulary — in practice, the same model
  family at a smaller size (e.g. a 0.5B draft for a 7B target).
- Draft weights are resident: roughly 350–400 MB for a 0.5B at Q4.
- Draft length *k* is a tunable trade-off: too long wastes work on rejection, too
  short under-uses the mechanism.

**Architecturally** this lives entirely inside the backend (D3) and is declared as a
capability (D10). Nothing above the model layer knows it is happening. A backend
without draft support simply reports the capability as absent.

---

## 3. Cascade

Attempt work at the cheapest adequate tier; verify; escalate only on failure.

```
result = tier_1.attempt(unit)
if verify(result):  return result
result = tier_2.attempt(unit, hint=failure_of(tier_1))
if verify(result):  return result
return tier_3.attempt(unit, hint=...)
```

### 3.1 When it pays

With success probability *p* at the cheap tier:

```
E[cost] = c_cheap + c_verify + (1 − p) × c_expensive
```

Cascading beats going straight to the expensive tier when
`c_cheap + c_verify + (1−p)·c_expensive < c_expensive`, i.e. roughly when
`p > (c_cheap + c_verify) / c_expensive`.

The practical consequence: **cascade where verification is cheap and objective**,
and avoid it where verification requires as much judgement as the task. This is why
the tier assignment below is not uniform.

| Unit kind | Cheap tier viable | Verification |
|---|---|---|
| Classification, routing | Yes | Structural — valid label or not |
| Summarisation, triage | Yes | Cheap heuristics plus downstream usefulness |
| Mechanical edit (rename, signature change) | Yes | Parse check, then targeted tests |
| Test writing | Sometimes | The test runs and fails for the right reason |
| Novel implementation | Rarely | Expensive and subjective — skip the cascade |
| Architecture and design | No | Not objectively verifiable |

### 3.2 Escalation carries context

An escalating call includes the cheap tier's attempt and the reason it failed. The
expensive tier is not asked to start over — it is asked to fix a specific, diagnosed
failure, which is a smaller and better-specified problem.

---

## 4. Critic pass

A second model call reviews an edit before it is applied: does it do what the unit
intended, does it break adjacent assumptions, does it leave the file coherent.

It is **phase-scoped and configurable**, because it roughly doubles the cost of the
edit path. Defaults:

| Situation | Critic |
|---|---|
| Mechanical edit, parse-checked, covered by tests | Off — verification is stronger and cheaper |
| Edit in a file with no test coverage | On |
| Edit flagged high-impact by change-impact analysis (D23) | On |
| Autonomous run with no user reviewing | On |
| User actively watching each diff | Off — the user is the critic |

The critic may run at a *different* tier than the author. A small model is often a
capable reviewer of a specific change even when it could not have written it, since
recognising a mistake is easier than avoiding one.

---

## 5. Self-consistency

Sample *N* solutions for one unit and select by verification rather than by vote.
Voting on code is weak; **running the code is not**.

Requires parallel capacity (D21), so it is unavailable at concurrency 1 and becomes
attractive exactly where cheap tiers plus real verification are available. This is
one of the clearest arguments for the execution graph: three 1.5B attempts filtered
by a test run can beat one 7B attempt, at lower total cost — but only if they can
run at once.

---

## 6. The router

The router maps a work unit to a model profile. It is an ordinary, inspectable
component, not a learned black box.

```
route(unit, context) -> ModelProfile

inputs:
    unit.kind                 explore | plan | edit | verify | triage | review
    unit.resource_class       hint from the planner
    estimated_complexity      input size, symbol count, impact radius (D23)
    remaining_budget          tokens, time, credits
    hardware_profile          loaded models, free KV cache, compute class
    mode                      the user-selected behaviour mode (D5, P4)
    history                   escalations already attempted for this unit
```

**Rules.**

1. **Mode selection is absolute.** A user-selected mode (including Uncensored Mode)
   pins the profile family. The router chooses tiers *within* it, never across it.
2. **Every decision is explained.** Routing choices are emitted on the event stream
   with their reason. A user must be able to see why a request went to a small
   model.
3. **Users can pin.** An explicit profile choice disables routing for that unit.
4. **Budget pressure lowers tiers before it truncates context.** Degrading model
   quality is more recoverable than degrading the information the model receives.
5. **Escalation is bounded.** A unit has a maximum tier and a maximum number of
   escalations, after which it fails to recovery (D24) rather than climbing forever.

---

## 7. Profiles

A profile bundles everything that makes a model behave as a specific Solara
configuration:

```
ModelProfile
  id                       stable name, e.g. "coder-7b-default"
  backend                  which ModelBackend serves it
  weights                  model reference
  draft_model              optional, for speculative decoding
  prompt_template          how the assembled context is rendered
  sampling                 temperature, top-p, repetition handling
  identity_rendering       how the identity artifact is expressed (D14)
  tool_policy              which tools this profile may use
  content_rules            the rule set for this mode
  tier                     1 (cheap) … 3 (capable)
  context_length           usable window
  capabilities             inherited from the backend (D10)
```

Profiles are configuration, not code. Adding a model, or a mode, is a profile.

**Uncensored Mode is a profile with different weights and different `content_rules`.
It does not alter `tool_policy` beyond what the policy gate permits, and it cannot
relax the gate, the workspace jail, or approval requirements (D9, D14).** Those are
properties of Solara, not of a model.

---

## 8. Example configurations

**Current laptop — i7-1065G7, 16 GB, CPU**

| Tier | Model | Use |
|---|---|---|
| 1 | Qwen2.5-Coder-1.5B Q4 | Classification, triage, mechanical edits |
| 2 | Qwen2.5-Coder-7B Q4 | Everything substantive |
| 3 | Remote backend (D4) | Escalation only |

Concurrency 1. Speculative decoding measured before adoption — a 0.5B draft
alongside a 7B target competes for both RAM and the same four cores.

**RTX 5070 — 8 GB VRAM, 32 GB RAM**

| Tier | Model | Use |
|---|---|---|
| 1 | Qwen2.5-Coder-1.5B Q4, resident | Cheap tier, critic, triage |
| 2 | Qwen2.5-Coder-7B Q4 + 0.5B draft | Primary |
| 3 | Remote, or 14B with partial offload | Escalation |

Concurrency 3–4, bounded by KV cache. Speculative decoding on by default.
Self-consistency becomes viable for cheap-tier units.

---

## 9. Relationship to earlier decisions

| Decision | Status |
|---|---|
| D3 — backend adapter | Unchanged. Speculative decoding lives inside the backend. |
| D4 — remote backends | Unchanged. Remote is a legitimate escalation tier. |
| D5 — profiles as routing | Extended: profiles remain the unit of selection; cascade, critic and self-consistency are added on top. |
| D10 — capability declaration | Extended with draft-model support and batching capacity. |
| D14 — identity | Unchanged and reinforced: identity renders per profile; modes cannot bypass structural enforcement. |
| D21 — execution graph | Self-consistency and parallel escalation depend on it. |
