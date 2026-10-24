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

## [Unreleased]

### Test identities

- Added **test identities** (§8.2): synthetic principals the target establishes in its evaluation environment so a product-embedded agent can act *as someone* the product's authorization layer recognizes, without an ambient default principal (AEP-REQ-038/117) and without real accounts or credentials reaching an evaluator (AEP-REQ-097). Evaluators discover identities and bind one by `identityId` at session start; they never create them and never hold their credentials — the server establishes the acting principal under the evaluator's own authorization.
- Added Core Profile **identity discovery** endpoints `GET /aep/identities` and `GET /aep/identities/{identityId}` (§7.1), with `aep.identity.list` / `aep.identity.get` JSON-RPC equivalents, filtering (`agent`, `mode`, `kind`, `tenant`, `status`, `q`, `updatedSince`), pagination, and existence-hiding semantics. The list is legitimately empty on a server with no identity-bound agents.
- Added `contextModel.identity` to the Agent Contract (`IdentityModel`: `binding` as the agent-wide default, `none | optional | required`, default `none`; `perMode` overrides over the five live modes, inherited where unlisted, `replay` never a key, keys outside `evaluationModes` refused at startup) and mirrored it unchanged as `AgentSummary.identityBinding`; `identityId` on `POST /aep/sessions` / `aep.session.start`; an `IdentityBinding` snapshot (identity, tenancy, roles, fixture version + digest, an `authorizationRef` into the bind's audit record, `boundAt`) on `EvaluationSession.identity` and `Trace.identity`, and carried whole on the `session.started` stream event so stream-to-Trace reconstruction stays lossless (AEP-REQ-114). An identity is *applicable* to an agent when the target's declaration covers the agent and its `kind` is one the contract's `contextModel.identity.kinds` accepts. On the wire a `TestIdentity` carries `applicability`: per-caller entries of `{ agentId, authorizedModes }`, the intersection of the declaration, the agent's modes and the caller's per-agent per-mode authorization, as `AgentSummary.authorizedModes` is. `replay` is never listed, because replaying a recorded identity is governed by the Trace's mode (AEP-REQ-046); re-execution still requires the caller to be authorized for the identity. Session start checks run in a pinned order so errors never reveal hidden resources; a retired identity is `-32071` (permanent), `-32072` always carries `retryAfterMs`. Concurrency slots are released when the session leaves `active` (expiry included) and enforced per `fixtureId`; the fixture, not the tenant, is the isolation boundary; fixture writes land only in `sandboxed-live`; `resetPolicy: none` identities are never offered there. Trace-faithful playback never touches the identity.
- New schema [`test-identity.schema.json`](../schemas/test-identity.schema.json) (`TestIdentity` plus `$defs/IdentityBinding`), with fixtures under `schemas/tests/test-identity/`. Versioned, digested **fixtures** behind an identity make replay honest: re-execution rebinds the same identity and compares fixture version/digest before claiming reproduction, while trace-faithful playback (AEP-REQ-059) only reads the snapshot and never depends on the identity still existing.
- New error codes `-32070 identity_not_found` (404), `-32071 identity_binding_invalid` (400), `-32072 identity_unavailable` (503, retryable).
- Requirements AEP-REQ-141 through AEP-REQ-153. Additive for agents and evaluators (an absent `identity` declaration means `none`); servers gain one Core Profile endpoint pair that may return an empty list. Identity provisioning and fixture reset as protocol operations are deferred (§17).

### Capability extension data

- Added optional **`CapabilitiesRegistry.extensionData`** (§5.1), allowing registered and reverse-DNS vendor extensions to publish versioned structured metadata (`version`, optional `schemaRef`, object `payload`).
- Preserved `capabilities.extensions` as the authoritative extension-support declaration for negotiation, discovery filtering, `AgentSummary.supportedExtensions`, and conformance declarations.
- Defined opaque handling for unknown extensions: implementations that do not recognize an extension preserve its payload without interpreting it and do not reject otherwise valid contracts.
- Prohibited automatic retrieval of untrusted extension schemas referenced by `schemaRef`.
- Prohibited extension payloads from weakening AEP core semantics or carrying secrets, PII, credentials, or runtime tenant data.
- Added requirements AEP-REQ-134 through AEP-REQ-140. Purely additive (AEP-REQ-105, no version bump).

### Observability Correlation extension

- Added optional **Observability Correlation extension** (§11.9, extension id `observability-correlation`): an `observabilityKey` object on `AgentContract`/`AgentSummary` describing how an external telemetry system (e.g. OpenTelemetry, Application Insights) tags spans for that agent, and an `observability` correlation block on `Trace` — session-level and per-turn — carrying the external `traceId`/`spanId`. Purely additive (AEP-REQ-105, no version bump); the Trace remains self-contained and replayable without external data. Requirements AEP-REQ-131 through AEP-REQ-133.

### Security and artifact hardening

- Added normative requirements for **signed sealed traces** using canonical serialization, digesting, and signature envelopes with key IDs (`kid`).
- Added normative requirement that **session, trace, result, and evidence-target references** are opaque, non-sequential, and high-entropy.
- Added first-class **trace sensitivity labeling and redaction records** requirements.
- Added **artifact retrieval existence-hiding semantics** so unauthorized trace/result/finding fetches return not-found behavior instead of authorization disclosures.
- Strengthened **non-production exposure** requirements into concrete deployment invariants (environment refusal defaults, ingress restrictions, separation, and audience-separated credentials).
