# Resilience: Failure, Stuck-Detection and Recovery

Detail for [D24](./decisions.md).

---

## 1. The problem

An agent that cannot tell it is failing will burn an entire budget repeating a
mistake. This is not an edge case — it is the *characteristic* failure of small
models on long tasks, and it is expensive in exactly the currency the user pays in:
time at 4 tok/s, and credits under P2.

Recovery is therefore a designed subsystem with its own state, signals, and
escalation policy, not error handling scattered through the agent.

---

## 2. Stuck signals

Each signal is cheap to compute and individually weak. Recovery triggers on
accumulated evidence, not on any single one.

| Signal | Detection | Why it indicates stuckness |
|---|---|---|
| **Repetition** | Near-identical tool calls recur (normalised arguments, similarity threshold) | The agent is re-asking a question it already has the answer to |
| **No state change** | N consecutive units complete with no file modified and no new facts in the working set | Work is happening; progress is not |
| **Identical verification failure** | The same test fails the same way after N repair attempts | The diagnosis is wrong, so the repairs cannot be right |
| **Edit-match failure** | Search/replace repeatedly fails to match (D18) | The model's belief about file content is stale or wrong |
| **Oscillation** | A file returns to a previously-seen content hash | Two changes are undoing each other |
| **Budget drift** | Large budget fraction consumed against small graph progress | Cost and progress have decoupled |
| **Plan thrash** | Repeated graph mutation without units completing | Planning has replaced doing |

Content hashing makes oscillation detection exact rather than heuristic, which is
worth noting: it is one of the few stuck signals with no false positives.

---

## 3. Escalating responses

Ordered by cost. Recovery starts at the cheapest response that fits the signal and
escalates only if it fails.

### 3.1 Reflect

A forced unit whose only job is to state what has been tried, what was observed,
and why the current approach is not working — with the failure history in context
and *no tools available*.

It is first because it is cheap and it works surprisingly often. Stuckness is
frequently an attention problem rather than a capability problem: the evidence that
the approach is wrong is already in context, unattended. Removing tool access is
deliberate — it forces analysis instead of another attempt.

Output: either a revised approach (graph mutation) or an admission that the
approach is wrong, which triggers §3.2.

### 3.2 Backtrack

Restore graph state, working set, and **file state** to a prior checkpoint, then
proceed down a different branch.

Choosing the target checkpoint matters: the correct one is the last point before the
decision that led into the dead end, which is usually *not* the most recent
checkpoint. The failing subtree is preserved as `superseded` so its lessons stay
available and the same branch is not re-attempted.

### 3.3 Switch strategy

Change something structural about the approach rather than retrying it:

| Failure shape | Switch |
|---|---|
| Cheap tier failing repeatedly | Escalate the tier (D22) |
| Edits not matching | Re-read the file fully; abandon assumptions about its content |
| Repair attempts failing identically | Re-diagnose from scratch in an isolated unit, discarding the current theory |
| Exploration not finding it | Change search strategy — symbol lookup instead of text, dependency traversal instead of listing |
| Task decomposition wrong | Re-plan from the original goal with the failure history as input |

### 3.4 Escalate to the user

With a real account: what was attempted, what was observed, what the current theory
is, and the specific question that would unblock it.

"I couldn't do it" is a defect. "I've tried X and Y; the test fails at Z, which
suggests the fixture is stale — is `conftest.py` meant to create that table?" is the
requirement.

### 3.5 Abort cleanly

Restore the workspace to its state at task start, or to the last checkpoint the user
approves. **A failed task must never leave a half-applied change.** Partial edits are
worse than no edits: they are silent, and they will be discovered later out of
context.

---

## 4. Shadow snapshots

Backtracking requires undoing file changes, and per C8 it cannot depend on version
control.

Solara maintains its own **shadow snapshot store**:

- copy-on-write — a file is copied once, immediately before its first modification
  within a checkpoint window;
- keyed by checkpoint, so a restore is a well-defined set of file writes;
- stored outside the workspace, so snapshots never appear in the workspace tree, in
  search results, or in a user's version control;
- bounded by size and count, with the oldest checkpoints evicted first;
- covering only files Solara itself modified, never the whole workspace.

**Where version control exists it is used as an additional safety net, never as the
mechanism.** Solara does not create commits, stashes, or branches without an explicit
request; a coding assistant that silently rewrites a user's VCS state is a hazard.

---

## 5. Budgets

Every task and every unit carries a budget across four dimensions:

| Dimension | Purpose |
|---|---|
| **Tokens** | The real cost driver, and the P2 credits unit |
| **Wall-clock** | What the user actually experiences at 3–6 tok/s |
| **Tool calls** | Bounds runaway loops that consume few tokens per call |
| **Credits** | The hosted-deployment seam (D0/P2), a no-op locally |

Budgets **degrade before they fail**. At 70% consumption the router lowers tiers and
the scheduler narrows speculative work. At 90% new speculative branches stop and
only critical-path units are admitted. At 100% the task escalates to the user with
its state intact and resumable — it is never silently killed and never silently
continued.

Sub-unit budgets are drawn from the parent's remaining allocation, so a runaway
child cannot consume a parent's entire budget (D21, D9).

---

## 6. Loop bounds

Independent of stuck detection, hard bounds prevent unbounded execution:

- maximum graph depth (nested isolated units)
- maximum total units per task
- maximum escalations per unit (D22)
- maximum recovery attempts per unit, after which recovery escalates to §3.4
- maximum consecutive graph mutations without a unit completing

Every bound is configurable and every one is emitted on the event stream when hit,
so a user can see *why* Solara stopped rather than watching it stop.

---

## 7. What is deliberately not attempted

**Automatic recovery from ambiguous requirements.** If the task is underspecified,
no amount of retrying fixes it. This escalates to the user immediately rather than
after burning a budget.

**Recovery from a wrong goal.** If the user asked for the wrong thing, Solara may
observe the mismatch and say so, but it does not substitute its own goal.

**Infinite persistence.** There is always a terminal state. An agent that never
gives up is not resilient — it is expensive.

---

## 8. Relationship to earlier decisions

| Decision | Status |
|---|---|
| D21 — execution graph | Provides checkpoints; recovery mutates the graph via supersession. |
| D18 — edit application | Match failures are a first-class stuck signal. |
| D19 — verification | Repeated identical failures drive re-diagnosis rather than repair. |
| D22 — orchestration | Tier escalation is a strategy switch; escalations are bounded. |
| D17 — API | Recovery state and escalations are emitted, and interruption remains available throughout. |
| C8 — workspaces | Snapshots work without version control; VCS is a safety net only. |
