# On Replay

*Design framing for deterministic replay of AEP evaluations.*

---

Replay is one of the six evaluation modes (see [`../patterns/TEST-MODES.md`](../patterns/TEST-MODES.md)). This document describes what replay is for, what it requires of agents and servers, and the design decisions AEP has made and deferred.

Replay is named as a first-class mode in v0.1 because most observability systems treat it as an afterthought, and an afterthought replay is one that doesn't actually reproduce. Naming the mode up front is a commitment to take reproduction seriously rather than discover the trace format is inadequate after it has shipped.

## What replay is for

Four concrete uses drive the design:

1. **Post-hoc analysis.** A finding from last week warrants another look. Re-running the scenario is expensive and, for stochastic agents, unreliable. Replay lets the evaluator re-examine the same run with different lenses.

2. **Bisection across agent versions.** A regression appears. Which version introduced it? Replay the same scenario against each candidate version without re-authoring the scenario or re-paying model costs unnecessarily.

3. **Re-scoring.** A new rubric exists. How does it score the historical corpus? Run the rubric against captured traces without touching the agent.

4. **Cheap regression detection.** Before an agent change lands, replay the prior version's traces against the new version and flag behavioral divergence. Cheaper than re-running the full scenario suite from scratch.

The common thread: replay lets evidence be re-examined without re-execution. This only works if the trace captures enough.

## What "deterministic replay" actually means

Three distinct claims get conflated:

1. **Byte-identical output** — the agent produces the exact same bytes given the same input + context + seed. This is rare in practice; even deterministic models can produce different outputs due to floating-point non-associativity across hardware.

2. **Behaviorally equivalent output** — the agent produces semantically equivalent output that would receive the same finding under the same rubric. This is usually what's wanted.

3. **Trace-faithful replay** — the agent is not re-executed at all; the captured trace is replayed as a recording. Whatever the original run did, the replay shows.

AEP's Determinism declaration (in the Agent Contract) supports all three:

- `deterministic` agents claim (1) — same input + context produces byte-identical output
- `seeded` agents claim (1) conditional on a seed being supplied
- `stochastic` agents claim only (2) at best; AEP's replay mode for stochastic agents falls back to (3)

Replay mode as implemented is always (3) — trace-faithful replay. The distinction matters because (3) answers "what happened" but not "what would happen now."

## Required capture

The v0.1 spec (§10, §11) requires the following to be in every Trace:

- Full session `injectedContext` snapshot
- `seed` value if the agent is `seeded`
- Per-turn input and output with digests
- Per-turn timestamps and latencies
- For `toolbox+`: `toolCalls` with arguments and results
- For `policybox+`: `policyEvents` with rule ID, outcome, severity
- For `toolbox+`: `reasoning` at the declared level
- Per-turn `stateSnapshot` for multi-turn agents
- Per-model-call: model identifier, parameters (temperature, top_p, seed), prompt digest or full prompt per `traceabilityContract.modelCalls`

This is the minimum. Servers **MAY** capture more; clients **MUST** tolerate extra fields.

## Design decisions made

### Capture injectedContext in full, not by reference

An early draft pointed to an external context store. This failed because replay from a different account, in a different tenant, or after context had been archived became unreliable. The trace captures the snapshot, not the pointer. Traces become larger; replay becomes dependable.

### Seal traces at session end

Sealed traces are immutable. Mutation attempts fail with error `-32006`. This is non-negotiable because replay without immutability is not replay — if the trace can be rewritten, the "replay" is an imagined run.

### Replay MUST NOT fall through to live tools

If a replay attempts to invoke a tool call not in the captured trace, the server **MUST** raise `-32050 replay_integrity_violation` rather than silently call the live tool. The alternative — "replay mostly the recording, but fall through on misses" — produces results that are neither replay nor live, and is impossible to reason about.

### Mode-aware replay authorization

A trace captured in `toolbox` mode **MUST NOT** be replayable by a caller authorized only for `blackbox`. This prevents information that was visible in the original run from leaking to parties who shouldn't see it through the replay mechanism.

## Design decisions deferred

### The canonical trace serialisation format

The spec defines what a trace **contains** (via `trace.schema.json`) but not a canonical wire format for sharing traces across AEP implementations. v0.1 uses JSON; v0.2 may define a more efficient format for large traces.

### Cross-agent-version replay semantics

What happens when you replay a trace captured against agent v1 using agent v2? v0.1 supports this only as trace-faithful replay (you see what v1 did). True re-execution across versions is a v0.2 concern, with open questions about seed compatibility, context migration, and how to surface divergence.

### Partial replay

Replaying only turns 3-5 of a 10-turn trace is not specified. It raises questions about state initialization (you're starting from the captured turn-2 state, which may embed assumptions) that the v0.1 spec doesn't address.

### Non-determinism budgeting

A stochastic agent may produce different outputs on re-execution that are "close enough." What counts as close? v0.1 punts: stochastic agents can only do trace-faithful replay. Future versions may introduce a configurable tolerance for behavioral-equivalence claims.

### Replay of external tool effects

If a tool call in the trace had side effects (e.g. wrote to a database), what does "replay" mean? The trace captures the observed result, not the external state change. A replay re-observes the recorded result but does not re-effect the change. This is correct for replay's purpose (re-examination) but insufficient for any notion of "re-running the side effects" — which is a separate concern the protocol does not attempt.

## Failure modes

Known ways replay goes wrong, each with its mitigation:

| Failure | Mitigation |
|---------|-----------|
| Trace incomplete | Server refuses to produce an EvaluationResult claiming `traceCompleteness: "complete"` unless every required field is present |
| Trace drifted from agent version | Trace carries `agentVersion`; replay warns on version mismatch |
| Replay divergence due to non-determinism | `DeterminismDeclaration` makes the expected behavior explicit |
| Replay against a deprecated agent | Servers **MAY** refuse replay against agents with `lifecycle.stage: "deprecated"` after a retention window |
| Tool-call mismatch at replay time | Error `-32050` rather than fall-through |

## What replay is not

- Replay is not a time machine. It reproduces what was captured, not what the system currently is.
- Replay is not verification. A replayed run passing a rubric means the captured behavior passes; it does not certify current behavior.
- Replay is not a substitute for live testing. Live tests catch what capture missed; replay catches what capture got right.

The design is modest on purpose. Modest replay that works beats ambitious replay that sometimes does.
