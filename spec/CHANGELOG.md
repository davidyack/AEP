# AEP Specification Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning: [SemVer](https://semver.org/spec/v2.0.0.html), with interoperability governed by capability negotiation.

## [0.1.0] — 2026-04-17

Initial release.

### Core

- **Self-contained normative spec.** Every MUST needed for interoperability is in [`AEP-0.1.md`](./AEP-0.1.md). All other documents are non-normative.
- **Stable requirement IDs.** Every MUST carries an `AEP-REQ-NNN` identifier; a future conformance test suite references these directly. [`../docs/CONFORMANCE.md`](../docs/CONFORMANCE.md) is a flat index.
- **REST-first with JSON-RPC.** Both surfaces are required and return equivalent data. REST for operator ergonomics and happy-path usage; JSON-RPC for capability negotiation and batching.
- **Discovery with filtering, pagination, and incremental sync.** `GET /aep/agents` returns `AgentSummary[]` with `authorizedModes` reflecting caller's permissions; supports filters by domain, mode, extension, risk, and lifecycle; cursor-based pagination; `updatedSince` for incremental client sync.
- **Required core vs optional extensions.** Small required surface (Agent Contract, session lifecycle, context injection, trace, error handling, security). Policy Profile, Tool Capability, Red-teaming, and Deterministic Replay are optional extensions with their own requirement IDs.
- **Explicit execution model.** AEP-REQ-012 through AEP-REQ-018 specify when scenarios complete, when scoring runs, and how partial states are exposed.
- **Required five-state lifecycle:** created → active → evaluated → reported → closed.
- **Happy-path walkthrough.** [`../docs/WALKTHROUGH.md`](../docs/WALKTHROUGH.md) — one end-to-end example from discovery to result.
- **Trace-faithful replay as baseline** (AEP-REQ-059). Every session produces a replayable trace; byte-identical agent replay is an optional extension.

### Schemas

Required: `agent-contract`, `agent-summary`, `agent-descriptor`, `capabilities-registry`, `context-bundle`, `scenario`, `evaluation-session`, `evaluation-result`, `trace`, `finding`.

Optional: `policy-profile`, `tool-capability`, `stream-event`, `dataset`.

### Known gaps for v0.2

- Companion reference implementation repository (`aep-reference`)
- Conformance test suite (each test cites `AEP-REQ-NNN`)
- Canonical trace serialisation format (binary framing for large traces)
- Early-abort signalling via the streaming extension
- Cost attribution as a first-class concern
- Full Composite Session Evaluation (CSE) extension specification (v0.1 core provides the hooks; the extension itself is a companion document)
- Cross-version replay semantics
