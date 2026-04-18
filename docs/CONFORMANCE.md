# Conformance Requirement Index

*A flat index of every AEP-REQ-NNN requirement, grouped by concern and cross-referenced to the spec.*

---

This document is an index, not a standalone specification. The authoritative text for every requirement is in [`../spec/AEP-0.1.md`](../spec/AEP-0.1.md); this file exists so a future test suite and compliance tracker can refer to requirements by stable ID.

Each requirement ID is permanent across 0.x versions. Deprecated requirements will be retained as tombstones (`DEPRECATED` status) rather than reused.

## How to use this document

- Building an AEP implementation? Use this as a checklist. Each row links to the spec section that defines the requirement.
- Writing a conformance test? Reference the AEP-REQ-NNN ID; it will not change.
- Declaring partial compliance? List AEP-REQ-NNN IDs in your `/.well-known/aep-compliance.json` exceptions array.

## Requirements by role

### AEP-Agent (authoring)
An agent is AEP-compliant when every requirement in this section is satisfied.

| ID | Summary | Spec |
|----|---------|------|
| AEP-REQ-002 | Publish an Agent Contract | §5 |
| AEP-REQ-003 | Contract includes all required fields | §5 |
| AEP-REQ-004 | Input/output schemas are valid JSON Schema 2020-12 | §5 |
| AEP-REQ-006 | Evaluator validates inputs against inputSchema before submission | §5 |
| AEP-REQ-019 | Maintain session state across turns for multi-turn agents | §6.4 |
| AEP-REQ-034 | Declare contextModel.injectionPoints with schemas | §8 |
| AEP-REQ-038 | Consume context only from injected sources | §8 |
| AEP-REQ-039 | Support at least blackbox and graybox modes | §9 |
| AEP-REQ-060 | Deterministic-replay agents declare determinism level | §11.4 |
| AEP-REQ-061 | Seeded agents capture seed + model parameters in traces | §11.4 |

### AEP-Server
A server is AEP-compliant when every requirement in this section is satisfied.

| ID | Summary | Spec |
|----|---------|------|
| AEP-REQ-001 | No evaluator-initiated tool invocation | §3 |
| AEP-REQ-005 | Validate own Agent Contract at startup | §5 |
| AEP-REQ-007 | Validate agent outputs against outputSchema | §5 |
| AEP-REQ-008 | Implement five-state lifecycle | §6.1 |
| AEP-REQ-009 | Record transitions; append-only | §6.1 |
| AEP-REQ-010 | Reject invalid-state methods with -32030 | §6.1 |
| AEP-REQ-011 | Enforce session expiry | §6.1 |
| AEP-REQ-012 | Define scenario completion semantics | §6.3 |
| AEP-REQ-013 | Record completion cause in finalOutcome | §6.3 |
| AEP-REQ-014 | No evaluate while turns in-flight | §6.3 |
| AEP-REQ-015 | active→evaluated blocks on scoring | §6.3 |
| AEP-REQ-016 | Declare sync vs async scoring | §6.3 |
| AEP-REQ-017 | Async scoring surfaces status: pending/ready | §6.3 |
| AEP-REQ-018 | Scoring failure still reaches evaluated state | §6.3 |
| AEP-REQ-020 | Session state isolation | §6.4 |
| AEP-REQ-021 | Implement both REST and JSON-RPC | §7 |
| AEP-REQ-022 | REST and JSON-RPC return equivalent data | §7 |
| AEP-REQ-023 | Discovery returns summaries, not full contracts; omits unauthorised agents | §7.1 |
| AEP-REQ-029 | Turns idempotent with idempotencyKey | §7.3 |
| AEP-REQ-030 | Expose aep.initialize for capability negotiation | §7.6 |
| AEP-REQ-031 | Require authentication on all endpoints | §7.7 |
| AEP-REQ-032 | Support mTLS, OAuth 2.0, or workload identity | §7.7 |
| AEP-REQ-033 | Short-lived tokens (≤1h recommended) | §7.7 |
| AEP-REQ-024 | Discovery supports cursor + limit pagination | §7.1 |
| AEP-REQ-025 | Discovery supports filter query parameters; rejects unknown ones | §7.1 |
| AEP-REQ-026 | Discovery summaries include authorizedModes for caller | §7.1 |
| AEP-REQ-027 | Discovery supports updatedSince for incremental sync; summaries carry lastUpdatedAt | §7.1 |
| AEP-REQ-028 | Unauthorised GET /agents/{id} returns -32001, not -32010 | §7.1 |
| AEP-REQ-035 | Supply injectedContext at session start | §8 |
| AEP-REQ-036 | Validate injected context; -32040 on mismatch | §8 |
| AEP-REQ-037 | Include injectedContext in Trace | §8 |
| AEP-REQ-040 | Reject unsupported modes with -32003 | §9 |
| AEP-REQ-041 | Enforce mode visibility boundaries | §9 |
| AEP-REQ-042 | Traces respect mode visibility | §9 |
| AEP-REQ-043 | Produce Trace for every session | §10 |
| AEP-REQ-044 | Seal traces at session end; -32006 on mutation | §10 |
| AEP-REQ-045 | Traces carry minimum required fields | §10 |
| AEP-REQ-046 | Mode-aware replay authorisation | §10 |
| AEP-REQ-047 | Retain sealed traces ≥7 days | §10 |
| AEP-REQ-059 | Support trace-faithful replay (required baseline) | §11.4 |
| AEP-REQ-062 | Deterministic replay blocks live tool calls; -32050 | §11.4 |
| AEP-REQ-063 | Replay Results declare traceCompleteness | §11.4 |
| AEP-REQ-079 | Use AEP error object shape | §12 |
| AEP-REQ-080 | Populate retryAfterMs for retryable errors | §12 |
| AEP-REQ-081 | Never leak internal details in errors | §12 |
| AEP-REQ-085 | Record errors in Trace | §12.3 |
| AEP-REQ-086 | No public production exposure | §13.1 |
| AEP-REQ-088 | Per-agent, per-mode authorisation independent | §13.2 |
| AEP-REQ-089 | Preserve tenant boundaries on artifact retrieval | §13.2 |
| AEP-REQ-090 | Log authorisation decisions | §13.2 |
| AEP-REQ-091 | Enforce rate limits per evaluator and per agent | §13.3 |
| AEP-REQ-092 | Rate-limited requests return -32020 | §13.3 |
| AEP-REQ-094 | Validate scenarios against schema | §13.4 |
| AEP-REQ-095 | Validate injected context | §13.4 |
| AEP-REQ-096 | Validate turn inputs against inputSchema | §13.4 |
| AEP-REQ-077 | Never surface secrets | §13.5 |
| AEP-REQ-097 | Reject real credentials as injected context | §13.5 |
| AEP-REQ-098 | Log auth attempts, auth decisions, lifecycle, errors | §13.6 |
| AEP-REQ-099 | Tamper-evident audit logs, ≥90 days | §13.6 |
| AEP-REQ-100 | TLS 1.3+ | §13.7 |
| AEP-REQ-101 | SHA-256+ for integrity hashes | §13.7 |
| AEP-REQ-102 | Interop by capability intersection, not version | §14 |
| AEP-REQ-103 | Advertise only fully-implemented capabilities | §14 |
| AEP-REQ-104 | Breaking changes require minor/major bump | §14 |
| AEP-REQ-105 | Additive capabilities don't bump version | §14 |
| AEP-REQ-106 | Vendor extensions use reverse-DNS prefixes | §14 |
| AEP-REQ-113 | All session-scoped info derivable from Trace | §10 |
| AEP-REQ-114 | Stream reconstruction produces full Trace without loss | §11.5 |
| AEP-REQ-115 | Stream events carry monotonic traceSeq | §11.5 |
| AEP-REQ-116 | Session reconstructable solely from Trace and inputs | §6.5 |
| AEP-REQ-117 | No hidden server state affects outputs without Trace representation | §6.5 |
| AEP-REQ-118 | AEP MUST NOT be used to expose arbitrary tools | §3 |
| AEP-REQ-119 | Implementations declaring AEP conformance satisfy Core Profile | §4.1 |
| AEP-REQ-120 | Declare supported profiles and extensions in conformance declaration | §4.1 |
| AEP-REQ-121 | Servers SHOULD support ephemeral evaluator tokens | §13.8 |
| AEP-REQ-122 | Servers SHOULD support scoped evaluation sessions | §13.8 |
| AEP-REQ-123 | Servers SHOULD support time-bound access | §13.8 |

### AEP-Evaluator (client)

| ID | Summary | Spec |
|----|---------|------|
| AEP-REQ-006 | Validate inputs against agent's inputSchema | §5 |
| AEP-REQ-029 | Use idempotency keys correctly when retrying | §7.3 |
| AEP-REQ-082 | Respect retryAfterMs | §12.3 |
| AEP-REQ-083 | Don't retry non-retryable errors | §12.3 |

## Optional-extension requirements

Only required for implementations claiming the extension.

### Policy Profile (§11.1)

| ID | Summary |
|----|---------|
| AEP-REQ-048 | Populate policyProfileRef in Agent Contract |
| AEP-REQ-049 | MAY gate profile retrieval on elevated mode |
| AEP-REQ-050 | Policy events reference published ruleIds |

### Tool Capability (§11.2)

| ID | Summary |
|----|---------|
| AEP-REQ-051 | Populate tools[] in Context Bundle |
| AEP-REQ-052 | Include reversibility, observability, blastRadius |
| AEP-REQ-053 | Forbid irreversible external tools in sandboxed-live without real-isolated sandbox |

### Red-teaming (§11.3)

| ID | Summary |
|----|---------|
| AEP-REQ-054 | Advertise red-teaming in capabilities.extensions |
| AEP-REQ-055 | Sessions carry metadata.redteam: true |
| AEP-REQ-056 | testMode must be policybox or higher |
| AEP-REQ-057 | Refusals/redactions/escalations surface as policy events |
| AEP-REQ-058 | Retain red-team traces ≥30 days |

### Streaming (§11.5)

| ID | Summary |
|----|---------|
| AEP-REQ-064 | Advertise streaming in capabilities.extensions; implement SSE endpoint |
| AEP-REQ-065 | Emit only registered event types; vendor events use reverse-DNS prefixes |
| AEP-REQ-066 | Heartbeat ≥every 30s with sequence id |
| AEP-REQ-067 | Honour Last-Event-ID for resume; 409 if outside retention window |
| AEP-REQ-068 | Retain replay buffer of ≥1000 events or 5 minutes per active session |
| AEP-REQ-069 | Streamed events applied in id order equal sealed Trace content |
| AEP-REQ-070 | Sealed Trace always retrievable via polling even if stream fails |
| AEP-REQ-071 | Scoring methods declare partial-evaluation support |

### Dataset (§11.6)

| ID | Summary |
|----|---------|
| AEP-REQ-072 | Advertise "dataset" in capabilities.extensions |
| AEP-REQ-073 | Datasets conform to dataset.schema.json |
| AEP-REQ-074 | Dataset inputs validate against agent's inputSchema when agentId declared |
| AEP-REQ-075 | Agents MAY publish datasetRefs in Agent Contract; discoverability only |
| AEP-REQ-076 | Datasets MUST NOT contain secrets, PII, or tenant-isolated data |
| AEP-REQ-078 | Converters from foreign formats MUST populate sourceRef |

### Composite Session Evaluation hooks (§11.7)

These are the core hooks that enable the CSE extension. The full extension is defined in a companion specification; v0.1 core requires only that these hooks be honoured.

| ID | Summary |
|----|---------|
| AEP-REQ-109 | Servers supporting CSE advertise "cse"; reject `graph` without support |
| AEP-REQ-110 | Exactly one of agentId or graph per session |
| AEP-REQ-111 | Composite sessions' trace turns carry nodeId |
| AEP-REQ-112 | compositeCapable: false agents MUST NOT be placed in composite sessions |

## Declaring conformance

Publish a declaration at `/.well-known/aep-compliance.json` (AEP-REQ-107):

```json
{
  "aepVersion": "0.1",
  "roles": ["AEP-Server", "AEP-Agent"],
  "extensions": ["policy-profile", "red-teaming"],
  "declaredAt": "2026-04-17T00:00:00Z",
  "exceptions": [
    { "id": "AEP-REQ-093", "reason": "Rate-limit headers not yet implemented" }
  ]
}
```

Unlisted exceptions are non-compliance (AEP-REQ-108).

## Not in this index

- SHOULD-level recommendations (non-binding; see spec)
- MAY-level options (discretionary; see spec)
- Non-normative guidance (see `RATIONALE.md`, `THREAT-MODEL.md`, etc.)

When the v0.2 conformance test suite ships, tests will cite these IDs directly.
