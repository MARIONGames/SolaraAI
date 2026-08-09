# Learning Flywheel

Detail for [D25](./decisions.md).

---

## 1. What this is for

Solara's long-term goal includes models that behave specifically for Solara and for
software development — eventually Solara-specific fine-tuned models. That goal is
unreachable without infrastructure that turns usage into data, and data into
measured improvement.

The flywheel:

```
usage → trajectories → implicit labels → eval suite → measured change
                                      ↘ filtered dataset → fine-tuning → better model → usage
```

The eval suite is the important half. **Fine-tuning without evaluation is
guesswork**, and the failure mode it prevents — silent regression — is what actually
kills systems like this.

---

## 2. Trajectories

Every task produces a trajectory: a complete, replayable record.

```
Trajectory
  task_goal              the original request
  graph                  full execution graph, including superseded and cancelled units
  per unit:
      assembled_context  what the model actually saw, by segment
      profile            which model profile served it (D22)
      raw_output         verbatim, before parsing
      parsed_calls       tool calls extracted, plus any repairs applied
      tool_results       results, with large payloads referenced not inlined
      edits              search/replace blocks, applied or rejected, with reasons
      verification       tier reached and outcome
  interventions          interruptions, corrections, approvals, denials
  outcome                see §3
  cost                   tokens, wall-clock, escalations
  environment            model, quantisation, backend, hardware, Solara version
```

Two details are what make trajectories usable rather than decorative:

**Assembled context is recorded, not reconstructed.** When something goes wrong, the
question is always "what did the model actually see?" Reconstructing it later from
the session is unreliable; recording it is cheap.

**Raw output is kept alongside the parse.** Protocol failures (D6) are only
diagnosable from what the model emitted before repair.

---

## 3. Implicit labelling

Nobody will label thousands of trajectories by hand, so labels are derived from
signals the user produces anyway.

| Signal | Interpretation | Strength |
|---|---|---|
| Verification passed at tier 3–4 | Success | **Strong** — objective |
| Edit survives the session unmodified | Success | **Strong** |
| User accepts a diff without edits | Success | Strong |
| User immediately reverts an edit | Failure | **Strong** |
| User rephrases the same request | Failure | **Strong** — the clearest negative signal available |
| User interrupts mid-task | Failure or misdirection | Moderate |
| Recovery triggered (D24) | Difficulty | Moderate |
| Escalated to a higher tier (D22) | Cheap tier insufficient | Moderate — useful for routing, not for quality |
| Task abandoned | Failure | Moderate |
| No signal at all | Unknown | Excluded from training data |

**Only strong signals produce training data.** Moderate ones inform routing and
eval design. Unknown outcomes are discarded rather than guessed at — a wrongly
labelled trajectory is worse than a missing one.

The rephrase signal deserves emphasis: a user restating the same request in
different words is the most reliable indicator of failure available, and it costs
nothing to detect.

---

## 4. The eval suite as a ratchet

A growing set of frozen tasks with **objectively checkable** outcomes, run against
every change to a model, prompt, protocol, or architecture.

**Composition.**

| Category | Checks |
|---|---|
| **Protocol conformance** | Well-formed tool calls, correct arguments, repair rate (D6) |
| **Edit application** | Search/replace match rate, parse survival, unnecessary-rewrite rate (D18) |
| **Task completion** | Real tasks in real workspaces, verified by tests |
| **Context discipline** | Budget adherence, working-set hygiene, no duplication (D11) |
| **Identity conformance** | Behaves as Solara across backends (D14) |
| **Safety** | Respects the policy gate, workspace jail, approvals — including under Uncensored Mode (D9) |
| **Efficiency** | Tokens and wall-clock per completed task |
| **Resilience** | Stuck detection fires, recovery resolves, clean abort (D24) |

**Growth from failure.** Every genuine failure observed in real usage becomes a
frozen eval case. The suite is therefore a record of every mistake the system has
made — which is precisely the set of mistakes it must not make again.

**Cases are frozen.** Once added, a case is not edited to make it pass. A ratchet
that can be loosened is not a ratchet.

**Cheap tiers run continuously.** Protocol and edit conformance run on every change;
full task completion runs less often, because it is expensive at local speeds.

---

## 5. Distillation

The realistic route to a Solara-specific model. Training a coding model from scratch
is not viable on any hardware in scope; **specialising an existing one is.**

```
1  A strong remote backend (D4) executes real tasks through the full Solara stack
2  Verification filters the results — only objectively successful trajectories survive
3  Surviving trajectories are converted to supervised examples:
       input   = the assembled context, exactly as the model saw it
       output  = the emitted tokens, including Solara's tool protocol
4  Deduplicate, balance across task kinds, hold out an eval split
5  Fine-tune a small local model (LoRA or full, per hardware)
6  Measure against the frozen eval suite
7  Promote only on improvement; otherwise discard and diagnose
```

**Why this produces a genuinely Solara-specific model rather than a generic coding
model:** the training examples are Solara's *assembled context* and Solara's *tool
protocol*, not generic instruction data. The model learns Solara's prompt structure,
Solara's tool syntax, Solara's identity, and Solara's editing conventions. That is
the concrete meaning of "make models behave specifically for Solara" — and it is
achievable with a LoRA on modest hardware, unlike pretraining.

**Step 7 is not optional.** Fine-tuning frequently makes things worse in ways that
are invisible without evaluation, particularly by narrowing the model onto the
distribution of the tasks it was tuned on.

---

## 6. Privacy

Trajectories contain the user's source code, file paths, and requests. This is the
most sensitive data in the system.

- **Local by default.** Trajectories never leave the machine unless explicitly
  exported.
- **Collection is opt-in**, per workspace, never global and never a default.
- **Redaction before export**: credentials, environment variables, tokens, and files
  matching ignore rules are stripped, and export is reviewable before it leaves.
- **Deletable**: trajectories are addressable and removable individually, per
  session, or per workspace — required by P5 and correct regardless.
- **Retention is bounded** by default rather than infinite.

An architecture that quietly accumulates a user's proprietary source into a training
corpus is a breach, not a feature. The opt-in boundary is structural.

---

## 7. What this is not

**Not online learning.** Solara does not update model weights during use. Weight
changes happen in a deliberate, evaluated, promotable step.

**Not automatic prompt mutation.** The identity artifact (D14) is not
self-modifying. Changes to it are proposed, reviewed, and measured.

**Not a substitute for memory.** Memory (D13) is explicit, readable, and immediate.
Learning is statistical, slow, and offline. Conflating them produces a system where
neither is inspectable.

---

## 8. Relationship to earlier decisions

| Decision | Status |
|---|---|
| D4 — remote backends | Becomes the teacher in distillation, which is the strongest argument for keeping it first-class. |
| D6 — tool protocol | Its conformance is directly measurable and directly trainable. |
| D11 — context assembly | Assembled context becomes the training input, so assembly quality bounds achievable model quality. |
| D13 — memory | Distinct system; deliberately not merged. |
| D14 — identity | The eval suite is its enforcement level 3, and the fine-tuning target. |
| D24 — resilience | Recovery events are labelling signals and eval cases. |
| P5 — EU data rules | Satisfied by local-default, opt-in, and per-item deletion. |
