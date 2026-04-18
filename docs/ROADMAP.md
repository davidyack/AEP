# Roadmap

Working document. Priorities reflect current understanding; order is a signal, not a commitment.

## v0.1 (shipped, April 2026)

- Pattern language (twelve patterns)
- Six-mode evaluation taxonomy
- Wire protocol with capability negotiation (JSON-RPC + REST discovery)
- Required session lifecycle (created → active → evaluated → reported → closed)
- Canonical schemas: Agent Contract, Capabilities Registry, Context Bundle, Policy Profile, Tool Capability, Scenario, Evaluation Session, Evaluation Result, Trace, Finding
- Conformance checklist (AEP-Agent, AEP-Server, AEP-Evaluator)
- Required context injection mechanism
- Required traceability contract
- Normative determinism and replay requirements
- Red-teaming interface
- 19 standardised error codes
- Threat model, security model, scoring framing, replay framing, error handling

## v0.2 (next)

1. **Companion reference implementation repository.** A minimal compliant server and client in a separate `aep-reference` repo. Kept separate so the spec can evolve without being coupled to a specific implementation's test harness. **This is the highest-priority v0.2 deliverable** — the spec is currently impressive-on-paper but the most common adopter question ("show me the smallest working pair") doesn't yet have an answer.

2. **Large-trace transport framing.** v0.1 now defines canonical serialization + signing semantics for Trace integrity; v0.2 focuses on efficient binary framing/chunking for very large traces.

3. **Streaming extensions.** v0.1 specifies SSE streaming (§11.5) with event registry, heartbeat, resume, and trace equivalence. v0.2 adds early-abort signalling, bidirectional streaming for interactive evaluation, and cross-session streams for dashboard use cases. See spec §11.8.

4. **Cost attribution.** Scenarios have real cost (model calls, tool calls, time). The protocol should carry cost metadata so evaluators can set ceilings and account against budgets.

5. **Sandbox fidelity contracts.** Tighten what `sandbox.fidelity` declarations guarantee, with verification hooks servers can expose.

## v0.3 (probably)

- **Composite Session Evaluation (CSE) extension.** The v0.1 core includes the hooks (graph in sessions, nodeId on trace turns, compositeCapable on contracts, agentId-or-graph on session start). The extension spec itself — graph schema, handoff semantics, node/edge/path/graph evaluation targets, CSE-specific scoring dimensions — is a companion specification pending in v0.2.
- **Cross-agent-version replay semantics.** Replay a v1 trace against v2 and surface divergence. Open questions about seed compatibility and context migration.
- **Conformance test suite.** A formal suite that certifies an implementation. Comes after the reference implementation, because the suite needs something to test against.
- **Non-determinism budgeting.** A configurable tolerance for behavioral-equivalence claims under replay, for stochastic agents.

## v1.0 (not before the evidence justifies it)

A v1.0 release is a statement of stability, not completeness. Criteria we want to see before declaring it:

- At least three independent implementations interoperating
- At least one implementation running in production CI against real agents for six months
- Conformance test suite covering the required MUSTs
- A satisfactory scoring answer that has survived real pipelines
- A pattern of clean minor-version upgrades without schema breaks

Until those are met, v1.0 is premature.

## Explicitly deferred

- **A central registry of AEP-compliant agents.** Useful, but not the protocol's job.
- **A built-in scenario sharing format.** Scenarios encode what you think your agent shouldn't do — the most culturally sensitive artifact. Cross-org sharing is a product concern, not a protocol concern.
- **Protocol-level RAI benchmark definitions.** Benchmarks churn faster than protocols. They should be expressible in the scenario schema without requiring protocol changes.
- **Certification programs.** Too heavy for a v0 effort, likely premature even at v1.

## How to influence the roadmap

Open an issue or PR. Evidence beats opinion — if you have an implementation hitting a limitation, that matters more than a thought about what might matter. See [`../CONTRIBUTING.md`](../CONTRIBUTING.md).
