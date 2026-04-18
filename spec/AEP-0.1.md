# AEP 0.1 — Agent Evaluation Protocol

**Version:** 0.1.0 · **Date:** April 2026 · **Status:** Normative

This is the authoritative normative specification for AEP. It is intentionally self-contained: every requirement needed for interoperability is in this document. Non-normative companions live in [`../docs/`](../docs/) and the pattern language lives in [`../patterns/`](../patterns/); these inform the spec but do not extend it.

---

## 1. Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be interpreted as described in RFC 2119 and RFC 8174.

Every MUST-level requirement carries an identifier of the form `AEP-REQ-NNN`. These identifiers are stable across minor versions and are the canonical handles for a future conformance test suite to reference.

A compliant implementation of AEP is one that honors every requirement addressed to its role (Agent, Server, Evaluator). §15 lists requirements by role.

## 2. Scope

### 2.1 Normative in this document
- Wire protocol (REST and JSON-RPC)
- Required data contracts (Agent Contract, Session, Result, Trace, Finding)
- Required session lifecycle and execution model
- Required endpoints and their semantics
- Security requirements
- Error handling
- Required vs optional extensions

### 2.2 Non-normative companions
- [`../patterns/PATTERNS.md`](../patterns/PATTERNS.md) — pattern language
- [`../patterns/TEST-MODES.md`](../patterns/TEST-MODES.md) — evaluation mode taxonomy
- [`../docs/RATIONALE.md`](../docs/RATIONALE.md) — design decisions
- [`../docs/THREAT-MODEL.md`](../docs/THREAT-MODEL.md) — threat analysis
- [`../docs/SCORING.md`](../docs/SCORING.md) — scoring concerns
- [`../docs/REPLAY.md`](../docs/REPLAY.md) — replay design framing
- [`../docs/WALKTHROUGH.md`](../docs/WALKTHROUGH.md) — happy-path end-to-end example

### 2.3 Out of scope
- Agent implementation internals
- Scoring algorithms
- Deployment operations
- Tool execution (MCP's domain)

## 3. Relationship to MCP

**AEP defines evaluation of decision-making systems. MCP defines invocation of capabilities. AEP MUST NOT be used to expose arbitrary tools, and MCP MUST NOT be used to evaluate agent behavior.** These are the hard architectural lines.

The two protocols solve different problems:

| Concern | MCP | AEP |
|---------|-----|-----|
| Direction | Agent → tools | Evaluator → agent |
| Question | "What can this agent invoke?" | "How does this agent behave?" |
| Session | Stateless tool calls | Stateful evaluation sessions |
| Runtime | In-band with production | Out-of-band, separate surface |
| Primary artifact | Tool catalog | Trace |

AEP-compliant agents **MAY** use MCP internally for tool invocation. AEP servers **MUST NOT** expose agent tools for evaluator invocation through AEP methods; evaluators observe tool usage via Traces (§10), not drive it.

**AEP-REQ-001**: AEP servers MUST NOT allow evaluator-initiated tool invocation through AEP methods.

**AEP-REQ-118**: AEP methods MUST NOT be used to expose arbitrary tool capabilities. An AEP endpoint MUST respond to session, turn, trace, and evaluation operations — not to generic tool-call invocations. Implementations that blur this line are non-compliant regardless of the specific methods they expose.

The temptation for implementers is to say "couldn't I just expose my agent as a tool via MCP?" The answer is that this would conflate evaluation with invocation: you'd lose the Trace (MCP has no equivalent), the session lifecycle, mode enforcement, and context-injection discipline. Agent evaluation is a distinct concern that warrants a distinct protocol; MCP covers the orthogonal concern of capability invocation.

## 4. Required Core vs Optional Extensions

The protocol has a **required core** — what every compliant implementation MUST support — and **optional extensions** — capabilities implementations MAY support but are not required to. This split keeps the core small and fast to implement while allowing richer behavior where it's warranted.

### 4.1 The Core Profile

The **Core Profile** is the minimum viable AEP implementation. Any server that implements the Core Profile is AEP-compliant and can be used for evaluation; everything beyond Core is either a larger profile or an optional extension that implementers opt into.

An implementation satisfies the Core Profile when it provides:

- **Single-agent sessions** (no graph / composite sessions)
- **Non-streaming operation** (sealed Trace retrieval only; no SSE)
- **Synchronous or asynchronous scoring** (either is acceptable; the choice is declared)
- **Trace production** for every session, sealed and retrievable (§10)
- **Five-state session lifecycle** (§6.1): created → active → evaluated → reported → closed
- **Discovery, session, turn, trace, result endpoints** (§7) on both REST and JSON-RPC surfaces
- **Agent Contract** publication with the required fields (§5)
- **Context injection** discipline (§8)
- **`blackbox` and `graybox` evaluation modes** (§9); others are opt-in
- **Registered error codes** (§12)
- **Required security model**: authentication, authorization by mode, rate limits, isolation (§13)
- **Conformance declaration** at `/.well-known/aep-compliance.json` (§15)

This is deliberately a smaller surface than the entire §4.2 list below. A Core-Profile-only server is a legitimate, useful AEP server — it can evaluate agents, produce Traces, accept findings, and interoperate with Core-Profile clients.

**AEP-REQ-119**: Implementations declaring AEP conformance MUST satisfy the Core Profile. Additional profiles, modes, and extensions are additive; they do not relax Core Profile requirements.

**AEP-REQ-120**: Implementations MUST declare which additional profiles and extensions they support in the conformance declaration (AEP-REQ-107). Clients MUST NOT assume support for anything beyond the Core Profile without verification.

### 4.2 Required core (details)

The full list of requirements for Core Profile conformance, referenced by ID, is in [`../docs/CONFORMANCE.md`](../docs/CONFORMANCE.md). The Core Profile corresponds to the AEP-Agent, AEP-Server, and AEP-Evaluator role requirements in that document, minus all optional-extension blocks.

### 4.3 Optional extensions

Implementations MAY support the following; if they do, they MUST follow the relevant specifications. Each extension is independently adoptable — supporting one does not require supporting any other.

- **Policy Profile** (`policy-profile.schema.json`) — machine-readable policy rules. §11.1
- **Tool Capability** (`tool-capability.schema.json`) — structured tool descriptors. §11.2
- **Red-teaming interface** — adversarial testing. §11.3 + [`../docs/RED-TEAMING.md`](../docs/RED-TEAMING.md)
- **Deterministic replay** — byte-identical reproduction for deterministic/seeded agents. §11.4
- **Streaming** — live SSE projection of the Trace. §11.5
- **Dataset** — portable dataset format for agent evaluation. §11.6 + [`../docs/DATASET.md`](../docs/DATASET.md)
- **Composite Session Evaluation (CSE) hooks** — core hooks enabling multi-agent evaluation; full extension defined in [`../docs/CSE.md`](../docs/CSE.md). §11.7

Agents MUST declare which optional extensions they support in the Agent Contract's `capabilities` field. Evaluators MUST NOT assume extension support without verification.

## 5. Agent Contract

Every AEP-compliant agent MUST publish an Agent Contract conforming to [`../schemas/agent-contract.schema.json`](../schemas/agent-contract.schema.json).

**AEP-REQ-002**: Agents MUST publish an Agent Contract retrievable via the discovery endpoints in §7.1.

**AEP-REQ-003**: The Agent Contract MUST include: `agentId`, `version`, `descriptor`, `inputSchema`, `outputSchema`, `contextModel`, `capabilities`, `evaluationModes`, `endpoints`.

**AEP-REQ-004**: `inputSchema` and `outputSchema` MUST be valid JSON Schema 2020-12 documents.

**AEP-REQ-005**: Servers MUST validate their own Agent Contract at startup and refuse to start on schema violation.

**AEP-REQ-006**: Evaluators MUST validate inputs against `inputSchema` before submission.

**AEP-REQ-007**: Servers MUST validate outputs against `outputSchema` before returning them; malformed outputs produce error `-32013`.

## 6. Session Lifecycle and Execution Model

### 6.1 Lifecycle

Every evaluation MUST occur within a session. Sessions follow a required five-state lifecycle:

```
  created ──start──▶ active ──evaluate──▶ evaluated ──report──▶ reported ──end──▶ closed
                       │
                        └──▶ aborted  (on timeout, auth failure, explicit abort)
```

`POST /aep/sessions` is the REST form of `aep.session.start`. A successful start call allocates the session, records the initial `created` state, performs the `created → active` transition, binds the agent version, initializes injected context, and then returns the active session summary. `created` is therefore a real lifecycle state even when it is transient in successful start responses.

**AEP-REQ-008**: Servers MUST implement the five states and enforce the transition rules in §6.2. `POST /aep/sessions` / `aep.session.start` MUST create the session in `created` and complete the `created → active` transition before returning success.

**AEP-REQ-009**: Servers MUST record every transition in the session's `lifecycle.transitions` array with timestamp. The array is append-only; once written, entries MUST NOT be modified.

**AEP-REQ-010**: Servers MUST reject methods invalid for the current state with error `-32030`.

**AEP-REQ-011**: Sessions MUST expire at `expiresAt`. Expired sessions transition to `aborted` and further operations return `-32002`.

### 6.2 State transitions

| From | To | Method | Notes |
|------|-----|--------|-------|
| created | active | `aep.session.start` | Binds agent version; initializes injected context |
| active | evaluated | `aep.session.evaluate` | Triggers scoring; see §6.3 |
| evaluated | reported | `aep.session.report` | Seals the EvaluationResult |
| reported | closed | `aep.session.end` | Releases resources |
| any | aborted | `aep.session.abort` | Timeout, auth failure, or explicit abort |

### 6.3 Execution model

This section specifies when scenarios complete, when scoring runs, and what partial states are observable. Underspecified execution is the source of most interoperability bugs, so the rules here are deliberately explicit.

**AEP-REQ-012**: A scenario is **complete** when any of the following occurs:
  (a) the last scenario turn executes successfully,
  (b) a failure condition with `haltScenario: true` fires,
  (c) the turn count exceeds `limits.maxTurnsPerSession`,
  (d) the scenario timeout expires, or
  (e) an unrecoverable error occurs.

**AEP-REQ-013**: Servers MUST record the completion cause in the Trace's `finalOutcome.status` field using one of: `completed`, `failed`, `timed_out`, `aborted`.

**AEP-REQ-014**: `aep.session.evaluate` MUST NOT be accepted while any turn is in-flight. Servers MUST either reject with `-32030` or wait for in-flight turns to drain; servers MUST document which behavior they implement.

**AEP-REQ-015**: Scoring runs during the `active → evaluated` transition. The transition MUST NOT complete until scoring produces an EvaluationResult or fails explicitly.

**AEP-REQ-016**: Scoring MAY be synchronous (blocking the evaluate call) or asynchronous (returning a result reference that clients poll). Servers MUST declare which mode they support in their capabilities at `aep.initialize`.

**AEP-REQ-017**: For asynchronous scoring, clients poll via `aep.result.get`. Servers MUST return `status: "pending"` while scoring is in progress and `status: "ready"` when complete.

**AEP-REQ-018**: If scoring fails, the session MUST still transition to `evaluated`, but the EvaluationResult MUST have `outcome.status: "error"` with a populated top-level `error` field. Clients MUST be able to retrieve the Trace even when the Result is in error.

### 6.4 Multi-turn state

**AEP-REQ-019**: For multi-turn agents (those declaring `contextModel.historyModel` other than `none`), servers MUST maintain session state across turns and include it in the Trace as per-turn `stateSnapshot`.

**AEP-REQ-020**: Servers MUST NOT expose one session's state to another. Session isolation violations are critical defects.

### 6.5 Server state vs replayability

AEP sessions are stateful for performance: servers maintain live session state to avoid re-transmitting context on every turn. But the *authoritative* record of the session is always the Trace, not the server's live state. These two rules keep that distinction sharp:

**AEP-REQ-116**: Servers MAY maintain session state for performance, correctness of in-session operations, and rate-limit accounting. A session MUST nonetheless be reconstructable solely from its Trace and inputs: given the sealed Trace of a completed session and the agent's contract and version, a conforming replay MUST produce equivalent outputs (subject to the agent's declared determinism level).

**AEP-REQ-117**: No hidden server state may affect turn outputs without being represented in the Trace. Any input that influences an output — injected context, random seeds, model parameters, system prompts, retrieved documents — MUST appear in the Trace. Servers MUST NOT maintain influence-bearing state outside what the Trace captures.

Together these mean: server state is an optimization; the Trace is the specification of what happened. A reviewer asking "is this system replayable?" gets the same answer regardless of implementation — yes, from the Trace.

## 7. Required Endpoints

AEP is presented in two equivalent surfaces: a REST surface (for discovery, operator ergonomics, and simple interactions) and a JSON-RPC 2.0 surface (for capability negotiation and efficient batch operations). Both surfaces MUST be supported; they return equivalent data.

**AEP-REQ-021**: Servers MUST implement both the REST endpoints in §7.1–§7.5 and the JSON-RPC methods in §7.6.

**AEP-REQ-022**: REST and JSON-RPC responses for equivalent operations MUST contain the same logical data.

### 7.1 Discovery (REST)

Discovery is a two-step pattern: list summaries, then fetch full contracts on demand.

```
GET  /aep/agents              → { agents: AgentSummary[], nextCursor, total }
GET  /aep/agents/{agentId}    → AgentContract
```

**AEP-REQ-023**: `GET /aep/agents` MUST return an array of `AgentSummary` objects (see [`../schemas/agent-summary.schema.json`](../schemas/agent-summary.schema.json)), not full contracts. Summaries contain enough to browse and filter; full contracts are retrieved per-agent.

**AEP-REQ-024**: `GET /aep/agents` MUST support pagination via query parameters `cursor` (opaque) and `limit` (integer, 1–100, default 50). Responses MUST include `nextCursor` when more results exist; `nextCursor` MUST be absent or null when the list is exhausted. Responses MUST include `total` when the server can compute it affordably; MAY omit `total` otherwise.

**AEP-REQ-025**: `GET /aep/agents` MUST support the following filter query parameters. Multiple filters combine with logical AND. Where a parameter accepts multiple values (comma-separated), values combine with logical OR within that parameter.

| Parameter | Matches | Example |
|-----------|---------|---------|
| `domain` | capability-registry domain ID | `domain=customer-service,billing` |
| `action` | capability-registry action ID | `action=classify` |
| `mode` | supported evaluation mode | `mode=policybox` |
| `extension` | supported optional extension | `extension=red-teaming` |
| `lifecycle` | deployment stage | `lifecycle=staging,ga` |
| `risk` | risk tier level | `risk=high,critical` |
| `q` | free-text search over name + description | `q=onboarding` |

Unknown filter parameters MUST be rejected with `-32602` (invalid params). Silent ignore is forbidden — a typo in a filter MUST NOT produce a false full result.

**AEP-REQ-026**: Each `AgentSummary` MUST include an `authorizedModes` field listing evaluation modes the *calling identity* is authorized to use for that agent, derived from the intersection of the agent's declared `evaluationModes` and the caller's authorization. Unauthorized agents MUST be omitted from the list entirely (AEP-REQ-023), not surfaced with empty `authorizedModes`.

**AEP-REQ-027**: `GET /aep/agents` SHOULD support an `updatedSince` query parameter (RFC 3339 timestamp). When provided, responses include only agents whose contract was updated at or after the timestamp. Each `AgentSummary` MUST include a `lastUpdatedAt` timestamp. Clients use this for incremental sync.

**AEP-REQ-028**: The `GET /aep/agents/{agentId}` endpoint returns the full `AgentContract`. If the caller is not authorized for the agent, the server MUST return `-32001 agent_not_found` (not `-32010`) to avoid leaking the existence of unauthorized agents.

Example list response:

```json
{
  "agents": [
    {
      "agentId": "example.inquiry-assistant",
      "version": "1.2.0",
      "name": "Inquiry Assistant",
      "description": "Helps PMs sharpen vague requests.",
      "riskTier": { "level": "medium" },
      "lifecycle": { "stage": "staging" },
      "domains": ["product-management", "requirements-elicitation"],
      "modalities": ["text"],
      "supportedModes": ["blackbox", "graybox", "policybox", "toolbox", "replay"],
      "authorizedModes": ["blackbox", "graybox", "policybox"],
      "supportedExtensions": ["policy-profile", "deterministic-replay"],
      "lastUpdatedAt": "2026-04-15T00:00:00Z"
    }
  ],
  "nextCursor": "eyJvZmZzZXQiOjUwfQ==",
  "total": 147
}
```

Example filtered call:

```
GET /aep/agents?domain=customer-service&mode=policybox&extension=red-teaming&limit=25
```


### 7.2 Session (REST)

```
POST   /aep/sessions                      → { sessionId, agentId, agentVersion, testMode, state, createdAt, expiresAt }
GET    /aep/sessions/{sessionId}          → EvaluationSession
POST   /aep/sessions/{sessionId}/evaluate → { resultRef }
POST   /aep/sessions/{sessionId}/report   → { resultRef, sealed: true }
POST   /aep/sessions/{sessionId}/abort    → { aborted: true }
DELETE /aep/sessions/{sessionId}          → { closed: true }
```

Request body for `POST /aep/sessions`:
```json
{
  "agentId": "string",
  "testMode": "graybox",
  "injectedContext": { },
  "ttlSeconds": 900
}
```

For single-agent sessions, supply `agentId` and the server creates the session bound to that agent at its current version. `POST /aep/sessions` completes the `created → active` transition before returning, so successful responses return the active session summary rather than a bare allocation token. Composite sessions — where execution spans a graph of agents — are the subject of the Composite Session Evaluation (CSE) extension (§11.7). When CSE is supported, the request body MAY instead supply a `graph` field in place of `agentId`; servers MUST accept exactly one of `agentId` or `graph`.

### 7.3 Turns (REST)

```
POST /aep/sessions/{sessionId}/turns  → TurnResult
GET  /aep/sessions/{sessionId}/turns  → TurnResult[]
```

Request body:
```json
{
  "input": { },
  "metadata": { "idempotencyKey": "string" }
}
```

**AEP-REQ-029**: `POST /aep/sessions/{sessionId}/turns` MUST be idempotent when an `idempotencyKey` is supplied. Repeated requests with the same key MUST return the original response.

### 7.4 Scenarios (REST)

```
POST /aep/sessions/{sessionId}/scenarios  → ScenarioResult
```

Executes a complete scenario to completion per §6.3.

### 7.5 Artifacts (REST)

```
GET /aep/traces/{traceRef}              → Trace
GET /aep/results/{resultRef}            → EvaluationResult
GET /aep/sessions/{sessionId}/findings  → Finding[]
```

Servers supporting the Streaming extension (§11.5) additionally expose `GET /aep/sessions/{sessionId}/events` as a Server-Sent Events stream. Streaming is additive; sealed artifacts are always retrievable via the endpoints above regardless of streaming support.

### 7.6 JSON-RPC surface

All operations above are also callable via JSON-RPC 2.0 at `POST /aep/rpc`:

| Method | Equivalent REST |
|--------|-----------------|
| `aep.initialize` | (no REST equivalent; capability negotiation) |
| `aep.agent.list` | `GET /aep/agents` |
| `aep.agent.get` | `GET /aep/agents/{id}` |
| `aep.session.start` | `POST /aep/sessions` |
| `aep.session.get` | `GET /aep/sessions/{id}` |
| `aep.turn.execute` | `POST /aep/sessions/{id}/turns` |
| `aep.scenario.run` | `POST /aep/sessions/{id}/scenarios` |
| `aep.session.evaluate` | `POST /aep/sessions/{id}/evaluate` |
| `aep.session.report` | `POST /aep/sessions/{id}/report` |
| `aep.session.abort` | `POST /aep/sessions/{id}/abort` |
| `aep.session.end` | `DELETE /aep/sessions/{id}` |
| `aep.trace.get` | `GET /aep/traces/{ref}` |
| `aep.result.get` | `GET /aep/results/{ref}` |
| `aep.findings.list` | `GET /aep/sessions/{id}/findings` |

`aep.agent.list` accepts the same filter, pagination, and `updatedSince` parameters as `GET /aep/agents`, passed in the `params` object:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "aep.agent.list",
  "params": {
    "domain": ["customer-service"],
    "mode": "policybox",
    "extension": "red-teaming",
    "limit": 25,
    "cursor": null,
    "updatedSince": "2026-04-01T00:00:00Z"
  }
}
```

Response shape matches the REST endpoint (AEP-REQ-022): `{ agents, nextCursor, total }`.

**AEP-REQ-030**: Servers MUST expose `aep.initialize` for capability negotiation. Subsequent JSON-RPC methods called before successful initialization return `-32011`.

`aep.initialize` request/response shape:
```json
// Request
{
  "aepVersion": "0.1",
  "client": {"name": "string", "version": "string"},
  "capabilities": {"streaming": false, "asyncScoring": false}
}

// Response
{
  "aepVersion": "0.1",
  "server": {"name": "string", "version": "string"},
  "capabilities": {
    "testModes": ["blackbox", "graybox", "policybox"],
    "extensions": ["policy-profile", "red-teaming", "deterministic-replay"],
    "asyncScoring": false
  }
}
```

### 7.7 Authentication

**AEP-REQ-031**: All endpoints MUST require authentication. Anonymous access MUST NOT be supported.

**AEP-REQ-032**: Servers MUST support at least one of: mTLS, OAuth 2.0 bearer tokens, or workload identity.

**AEP-REQ-033**: Authentication tokens MUST be short-lived. Recommended maximum: 1 hour.

## 8. Context Injection

Agents MUST consume context from injected sources, not ambient ones. This is what makes evaluation reproducible.

**AEP-REQ-034**: Every Agent Contract MUST declare `contextModel.injectionPoints`, each with a name, type, and schema.

**AEP-REQ-035**: Evaluators MUST supply `injectedContext` at `POST /aep/sessions` (or `aep.session.start`), keyed by the declared injection-point names.

**AEP-REQ-036**: Servers MUST validate injected context against the injection point schemas and return `-32040` on mismatch.

**AEP-REQ-037**: Servers MUST include the full `injectedContext` snapshot in the session's Trace.

**AEP-REQ-038**: Agents MUST NOT read context from ambient sources during evaluation. All context MUST arrive through declared injection points.

### 8.1 Reserved injection point types

The following types are reserved for standard meanings; custom types MUST use reverse-DNS prefixes:

- `user` — simulated user profile
- `system` — system state
- `tenant` — tenant or workspace configuration
- `history` — prior conversational turns
- `environment` — locale, timezone, feature flags
- `retrieval` — simulated retrieval results

## 9. Evaluation Modes

See [`../patterns/TEST-MODES.md`](../patterns/TEST-MODES.md) for conceptual treatment. Normatively:

**AEP-REQ-039**: Agents MUST support at least `blackbox` and `graybox`.

**AEP-REQ-040**: Servers MUST reject session creation in modes not in the agent's declared `evaluationModes` with error `-32003`.

**AEP-REQ-041**: Servers MUST enforce mode visibility boundaries:
- `blackbox`: only user-visible input/output
- `graybox`: adds Context Bundle access
- `policybox`: adds policy events
- `toolbox`: adds tool call detail
- `sandboxed-live`: toolbox with real (sandboxed) side effects
- `replay`: re-examines a captured Trace

**AEP-REQ-042**: A session's Trace MUST NOT contain content beyond its declared mode's visibility. A graybox Trace MUST NOT include tool call arguments.

## 10. Trace

Every session MUST produce a Trace conforming to [`../schemas/trace.schema.json`](../schemas/trace.schema.json).

**The Trace is the canonical, replayable record of a session. All turn responses, streaming events, and artifacts MUST be derivable from the Trace. No other record may carry information that is not present in (or not derivable from) the Trace.**

This is the protocol's anchor: when a reviewer asks "what is the single source of truth I can persist and replay?", the answer is the Trace. Everything else — live turn responses, SSE events, result artifacts, findings — is a transport, a projection, or a summary.

**AEP-REQ-043**: Servers MUST produce a Trace for every session, including failed and aborted ones.

**AEP-REQ-044**: Traces MUST be sealed at session end. Once `sealed: true`, mutation attempts return `-32006`.

**AEP-REQ-045**: Every Trace MUST contain, at minimum: all turn inputs and outputs, input/output digests, timestamps, and the session's `injectedContext` snapshot.

**AEP-REQ-046**: Traces MUST carry their originating test mode. Replay of a Trace MUST NOT be authorized for a caller whose mode authorization is below the Trace's captured mode.

**AEP-REQ-047**: Sealed traces MUST be retrievable for at least 7 days. Implementations MAY retain longer.

**AEP-REQ-113**: Every piece of session-scoped information returned by any AEP surface — turn response payloads, streaming events, evaluation results, findings, artifacts — MUST be either present in the Trace or mechanically derivable from it. Servers MUST NOT return session-scoped information that has no Trace representation.

## 11. Optional Extensions

Implementations MAY support these extensions. Implementations that claim support MUST honor the requirements in the relevant subsection.

### 11.1 Policy Profile extension

Agents MAY publish a Policy Profile (`policy-profile.schema.json`) with machine-readable rules.

**AEP-REQ-048**: Agents claiming `policy-profile` support MUST populate `policyProfileRef` in their Agent Contract.

**AEP-REQ-049**: Servers MAY require elevated mode (`policybox` or higher) to return non-trivial policy profiles.

**AEP-REQ-050**: When Policy Profile is supported, policy events in traces MUST reference `ruleId` values from the published profile.

### 11.2 Tool Capability extension

Agents with tool access MAY publish Tool Capability declarations (`tool-capability.schema.json`).

**AEP-REQ-051**: Agents claiming `tool-capability` support MUST populate the `tools` array in their Context Bundle.

**AEP-REQ-052**: Tool declarations MUST include `reversibility`, `observability`, `blastRadius`, and optionally `sandbox` fidelity.

**AEP-REQ-053**: Irreversible external tools MUST NOT be permitted in `sandboxed-live` mode unless their sandbox fidelity is `real-isolated`.

### 11.3 Red-teaming extension

Servers MAY implement a red-teaming interface for adversarial testing. See [`../docs/RED-TEAMING.md`](../docs/RED-TEAMING.md).

**AEP-REQ-054**: Servers claiming `red-teaming` support MUST advertise `"red-teaming"` in `capabilities.extensions` at initialization.

**AEP-REQ-055**: Red-teaming sessions MUST carry `metadata.redteam: true`.

**AEP-REQ-056**: Red-teaming sessions MUST use `testMode` of `policybox` or higher.

**AEP-REQ-057**: All refusals, redactions, and escalations during red-teaming MUST surface as explicit policy events in the Trace.

**AEP-REQ-058**: Red-teaming traces MUST be retained for at least 30 days.

### 11.4 Deterministic Replay extension

Every session produces a Trace that MAY be replayed in trace-faithful form. Agents that claim *deterministic replay* support an additional guarantee: re-executing against the Trace produces byte-identical behavior.

**AEP-REQ-059**: Every server MUST support Trace-faithful replay — i.e. re-reading a captured Trace. This is the required baseline; it does not require agent re-execution.

**AEP-REQ-060**: Agents claiming `deterministic-replay` MUST declare `determinism.level` as either `deterministic` or `seeded` in the Agent Contract.

**AEP-REQ-061**: For `seeded` agents, the Trace MUST capture the seed value and model parameters (temperature, top_p) per model call.

**AEP-REQ-062**: Deterministic-replay sessions MUST NOT invoke live external tools. Attempts to do so return `-32050`.

**AEP-REQ-063**: EvaluationResults from deterministic-replay sessions MUST include `reproducibility.traceCompleteness` indicating whether byte-identity could be guaranteed.

See [`../docs/REPLAY.md`](../docs/REPLAY.md) for the design rationale.

### 11.5 Streaming extension

Sessions normally produce a sealed Trace retrievable via `GET /aep/traces/{ref}` after completion. Servers MAY additionally expose the session's events as a live stream while the session is active, allowing clients to observe turn-by-turn progress, inspect intermediate state, or run partial-trace scoring before the session completes.

Streaming is an observability extension, not a control extension. It does not add stream-native control semantics such as early-abort based on streamed observations, mid-stream redirection, or bidirectional steering; those are deferred to v0.2. Session abort remains available through the standard lifecycle method `aep.session.abort`.

**Endpoint**

```
GET /aep/sessions/{sessionId}/events
Accept: text/event-stream
Last-Event-ID: {integer}   (optional, for resume)
```

The response is a Server-Sent Events (SSE) stream conforming to [WHATWG SSE](https://html.spec.whatwg.org/multipage/server-sent-events.html). Each event MUST have a numeric `id:` (monotonically increasing per session), an `event:` type, and a JSON `data:` payload.

**AEP-REQ-064**: Servers claiming `streaming` support MUST advertise `"streaming"` in `capabilities.extensions` at initialization and MUST implement `GET /aep/sessions/{sessionId}/events` for every active session.

**AEP-REQ-065**: Stream events MUST be drawn from the following registered event types. Vendor extensions MUST use reverse-DNS prefixes (e.g. `com.example.custom.event`).

| Event type | Emitted when | Payload includes |
|------------|--------------|------------------|
| `session.started` | Session transitions `created → active` | `sessionId`, `agentId`, `agentVersion`, `testMode`, `startedAt` |
| `turn.started` | Turn execution begins | `turnId`, `turnIndex`, `input`, `startedAt` |
| `turn.reasoning` | Reasoning step exposed (per traceabilityContract) | `turnId`, `step`, `content` |
| `tool.invoked` | Tool call dispatched (toolbox+ mode) | `turnId`, `toolCallId`, `toolName`, `arguments` |
| `tool.completed` | Tool call returns | `turnId`, `toolCallId`, `result`, `error`, `latencyMs` |
| `policy.event` | Policy rule triggers (policybox+ mode) | `turnId`, `policyEventId`, `ruleId`, `outcome`, `severity` |
| `turn.completed` | Turn completes | `turnId`, `output`, `latencyMs` |
| `session.evaluated` | Scoring completes | `sessionId`, `resultRef` |
| `session.error` | Unrecoverable session error | `code`, `message`, `turnId` (if applicable) |
| `stream.heartbeat` | Keepalive (no session state change) | `timestamp` |
| `stream.end` | Stream is closing | `reason` (one of: `session-complete`, `session-aborted`, `client-disconnect`), `sealed` |

**AEP-REQ-066**: Servers MUST emit a `stream.heartbeat` event at least every 30 seconds during an active session. Heartbeats MUST carry a sequence `id` so they count against resume offsets. Clients detecting no heartbeat for 60+ seconds SHOULD reconnect.

**AEP-REQ-067**: Servers MUST honor the `Last-Event-ID` request header. When present, the server MUST replay all events with `id` strictly greater than the supplied value, in order, before emitting new events. If the supplied ID is older than the server's retention window, the server MUST respond with HTTP 409 and the client MUST fall back to polling the sealed Trace.

**AEP-REQ-068**: Servers MUST retain a replay buffer of at least the last 1000 events or 5 minutes of stream history (whichever is larger) for each active session, to support reconnection.

**Trace equivalence invariant**

Streaming is not an authoritative surface. It is a **progressive emission of a future Trace**. Stream events are projections of Trace content, delivered early for observability; the Trace is the authority.

**AEP-REQ-069**: Every non-`stream.heartbeat`, non-`stream.end` event MUST correspond to a Trace entry (a turn, reasoning step, tool call, policy event, or session-level state change). Each event's payload MUST carry a `traceEventRef` that identifies which Trace entry it projects, using the form `turns[n]`, `turns[n].reasoning[m]`, `turns[n].toolCalls[m]`, `turns[n].policyEvents[m]`, or a session-level pointer.

**AEP-REQ-114**: A compliant implementation MUST be able to reconstruct the final Trace from the emitted stream without loss of session-scoped information. Specifically: given the complete event stream (all events with IDs from 1 to `stream.end`), applying each event's projection to an empty Trace MUST produce content equivalent to the sealed Trace retrieved via `GET /aep/traces/{traceRef}`. This guarantees streaming is a transport for the Trace, not a separate source of truth.

**AEP-REQ-115**: Stream events MUST carry a monotonic `traceSeq` field — a deterministic ordering hook that corresponds to the order of the associated entry in the sealed Trace. Events MAY arrive out of wall-clock order (interleaved across turns) but MUST arrive in order when filtered to a single turn's `traceSeq` sequence.

This is the invariant that lets judge and scoring code paths be written once against `Trace`: a client can consume the stream or poll the sealed artifact and the resulting data structure is the same.

**Fall-through to polling**

**AEP-REQ-070**: If a client disconnects permanently or the server cannot serve the stream, the sealed Trace MUST still be retrievable via the standard `GET /aep/traces/{traceRef}` endpoint after session completion. Streaming is additive; it never replaces the pull-based artifact retrieval.

**Partial-trace scoring**

**AEP-REQ-071**: Scoring methods that operate on streamed events MUST declare whether they support partial evaluation. Methods that require full-session context (e.g. aggregate metrics) MUST NOT run against incomplete streams; servers MUST reject premature `aep.session.evaluate` calls with `-32030` if turns are still in flight (reinforcing AEP-REQ-014).

### 11.6 Dataset extension

AEP core defines per-session semantics: one agent, one session, one or more turns. Real evaluation workflows drive thousands of sessions from a curated set of inputs — a **dataset**. Without a shared dataset shape, every tool mints its own (OpenAI Evals JSONL, LangSmith examples, HuggingFace parquet, Braintrust examples, …). The friction is in interoperability, not in any one format being inadequate.

This extension defines a canonical shape so datasets port cleanly between AEP-compliant tools. It is strictly additive — agents and servers that don't implement the extension remain fully compliant with the core.

**In scope:**
- A canonical JSON schema for a dataset document ([`../schemas/dataset.schema.json`](../schemas/dataset.schema.json))
- A discoverability hook on the Agent Contract (`datasetRefs`)
- A conformance rule linking dataset example inputs to the agent's declared `inputSchema`

**Out of scope:**
- Dataset storage, versioning, or distribution protocols
- Registries or marketplaces
- Mandatory adapters for foreign formats (OpenAI Evals, HuggingFace, LangSmith). These are tooling concerns; the spec defines the shape everyone lands in, not the conversion.

**AEP-REQ-072**: Implementations claiming dataset-extension support MUST advertise `"dataset"` in `capabilities.extensions` at `aep.initialize`.

**AEP-REQ-073**: Datasets MUST conform to [`../schemas/dataset.schema.json`](../schemas/dataset.schema.json). A dataset document has the shape:

```json
{
  "id": "string",
  "name": "string",
  "version": "string",
  "agentId": "string (optional)",
  "description": "string (optional)",
  "evaluator": "string (optional)",
  "policyProfileRef": "string (optional)",
  "sourceRef": "string (optional)",
  "metadata": { },
  "examples": [
    {
      "id": "string",
      "input": { },
      "expected": { },
      "references": [ ],
      "tags": ["string"],
      "metadata": { }
    }
  ]
}
```

Field semantics:

- `id`, `name`, `version` — identity. `version` is a semver-style monotonic string so tools can diff.
- `agentId` — when present, the dataset is intended for a specific agent; example inputs MUST validate against that agent's `inputSchema` (see AEP-REQ-074).
- `evaluator` — when the dataset is calibrated for a named RAI evaluator (e.g. `harmful_content`), record it here.
- `policyProfileRef` — ties the dataset to a policy profile (§11.1). Enables "this dataset exercises this rule set" queries.
- `sourceRef` — where this came from (HuggingFace slug, OpenAI Evals path, internal ticket). Informational only.
- `examples[].expected` — OPTIONAL reference answer for reference-aware grading.
- `examples[].references` — OPTIONAL multiple acceptable references, for reference-aware judging that accepts variant correct answers.

**AEP-REQ-074**: When a dataset declares `agentId`, evaluators MUST validate each `examples[].input` against the agent's `inputSchema` before session submission. AEP-REQ-006 already requires per-turn validation at runtime; this extends the requirement to the whole dataset at import time so bad examples fail fast.

**AEP-REQ-075**: Agents MAY publish `datasetRefs` in their Agent Contract — an array of opaque references that resolve to dataset documents (URL, OCI-style digest, internal slug). This is purely discoverability; servers MUST NOT derive behavior from the content of referenced datasets.

**AEP-REQ-076**: Dataset documents MUST NOT contain secrets, PII, or tenant-isolated data. Implementations SHOULD run a redaction pass before publishing a dataset (cf. AEP-REQ-077 for secret handling in the core).

**AEP-REQ-078**: When a dataset is imported from a foreign format (OpenAI Evals JSONL, HuggingFace, LangSmith, Braintrust, etc.), the converter MUST populate `sourceRef` so provenance is preserved.

#### Interop notes (non-normative)

Tools implementing this extension SHOULD provide adapters for at least:

- **OpenAI Evals JSONL** — one JSON object per line; typically `{input, ideal}` or `{inputs, expected}` shape.
- **HuggingFace Datasets** — via the `datasets` library's JSON/CSV export; dataset-card metadata.
- **LangSmith / Braintrust** — `{inputs, outputs, metadata}` example shape.

Adapters live outside the spec. See [`../docs/DATASET.md`](../docs/DATASET.md) for mapping guidance, consumer patterns, and security notes.

### 11.7 Composite Session Evaluation (CSE) extension

Real-world agent systems are often composed of multiple specialised agents coordinated by a router, orchestrator, or supervisor. Evaluating such a system one-agent-at-a-time misses the interesting failure modes: wrong routing, lost context across handoffs, correct individual agents producing incorrect joint outcomes.

The Composite Session Evaluation extension (extension id: `aep.cse`) enables evaluation of multi-agent systems within a single session by representing execution as a directed graph of agent nodes connected by handoff edges. Handoffs become first-class evaluatable events, and evaluation targets extend to nodes, edges, and the graph as a whole.

**Status in v0.1 core:** The core provides the hooks necessary for CSE but does not define the extension itself. The five hooks are:

1. `EvaluationSession.graph` — optional graph definition at session creation. Mutually exclusive with `agentId`.
2. `Trace` top-level `agentId` and `agentVersion` — relaxed from required to optional; required only for single-agent sessions.
3. `TraceTurn.nodeId` — optional per-turn pointer to the graph node active for that turn.
4. `AgentContract.compositeCapable` — optional boolean declaring an agent safe to participate as a composite-session node.
5. `POST /aep/sessions` — accepts exactly one of `agentId` (single-agent) or `graph` (composite).

**AEP-REQ-109**: Servers supporting CSE MUST advertise `"cse"` in `capabilities.extensions` and MUST accept session creation with a `graph` field. Servers not supporting CSE MUST reject session creation requests containing `graph` with error `-32005`.

**AEP-REQ-110**: When a session carries a graph, exactly one of `agentId` or `graph` MUST be present; both or neither are invalid.

**AEP-REQ-111**: For composite sessions, every Trace turn MUST carry a `nodeId` identifying the active agent node for that turn.

**AEP-REQ-112**: Agents declaring `compositeCapable: false` (or omitting the field) MUST NOT be placed as nodes in a composite session. Servers MUST reject such placements with error `-32005`.

The full CSE extension — graph schema, edge/handoff semantics, node/edge/path/graph evaluation targets, CSE-specific scoring dimensions (routing quality, handoff fidelity, end-to-end outcome), CSE-specific error codes, and CSE-specific stream events — is defined in [`../docs/CSE.md`](../docs/CSE.md). Teams adopting CSE should refer to that document directly; the hooks above ensure v0.1 core implementations are forward-compatible.

### 11.8 Deferred streaming features (v0.2)

The following are explicitly out of scope for v0.1 streaming and will be revisited:

- **Early abort** — client signalling "stop, I've seen enough" to halt a session based on streamed observations. Requires session-control semantics not in v0.1.
- **Bidirectional streaming** — client-supplied inputs mid-stream for interactive evaluation.
- **Cross-session streams** — observing multiple sessions over one connection for dashboard-style use cases.
- **Per-mode filtering** — asking for only `policy.event`s on a stream.

## 12. Error Handling

AEP errors use JSON-RPC 2.0 error-object form. REST endpoints return equivalent information as HTTP status codes with a JSON body of the same shape.

**AEP-REQ-079**: All AEP errors MUST use the error object:
```json
{
  "code": -32020,
  "message": "human-readable",
  "data": {
    "category": "Resource",
    "retryAfterMs": 2500,
    "correlationId": "string",
    "requirementRef": "AEP-REQ-NNN"
  }
}
```

**AEP-REQ-080**: `data.retryAfterMs` MUST be populated for retryable errors.

**AEP-REQ-081**: Servers MUST NOT expose internal details (stack traces, credentials, system prompts) in error messages.

### 12.1 Error code registry

| Code | Name | Category | Retry |
|------|------|----------|-------|
| -32001 | `agent_not_found` | Discovery | No |
| -32002 | `session_expired` | Lifecycle | No (create new) |
| -32003 | `mode_not_supported` | Modes | No |
| -32005 | `scenario_invalid` | Input | No |
| -32006 | `trace_sealed` | Immutability | No |
| -32010 | `authorization_failed` | Auth | No |
| -32011 | `not_initialized` | Lifecycle | No (initialize) |
| -32012 | `input_schema_violation` | Contract | No (fix input) |
| -32013 | `output_schema_violation` | Contract | Maybe (agent fault) |
| -32020 | `rate_limited` | Resource | Yes with backoff |
| -32021 | `limit_exceeded` | Resource | No |
| -32022 | `timeout` | Resource | Yes with jitter |
| -32030 | `invalid_state_transition` | Lifecycle | No |
| -32040 | `context_injection_invalid` | Context | No |
| -32041 | `context_injection_point_unknown` | Context | No |
| -32050 | `replay_integrity_violation` | Replay | No |
| -32060 | `agent_fault` | Agent | Limited retry |
| -32061 | `agent_unavailable` | Agent | Yes with backoff |

### 12.2 REST status code mapping

| Error code | HTTP status |
|-----------|-------------|
| -32001, -32041 | 404 |
| -32002, -32030 | 409 |
| -32003, -32005, -32012, -32040 | 400 |
| -32006 | 409 |
| -32010 | 401 or 403 |
| -32011 | 425 |
| -32013, -32060 | 502 |
| -32020 | 429 |
| -32021 | 413 |
| -32022 | 504 |
| -32050 | 409 |
| -32061 | 503 |

### 12.3 Retry obligations

**AEP-REQ-082**: Clients MUST respect `data.retryAfterMs` when present.

**AEP-REQ-083**: Clients MUST NOT retry errors without retry semantics (No in the table).

**AEP-REQ-084**: Clients SHOULD use jittered exponential backoff for retryable errors without explicit `retryAfterMs`.

**AEP-REQ-085**: Every error MUST be recorded in the session's Trace.

## 13. Security

### 13.1 Non-production exposure

**AEP-REQ-086**: AEP endpoints MUST NOT be reachable from public production traffic paths.

**AEP-REQ-087**: Servers SHOULD detect production environments at startup and refuse to start without explicit override; overrides MUST be logged.

### 13.2 Authorization

**AEP-REQ-088**: Authorization MUST be enforced per-agent and per-mode independently. Authorization for one agent or mode MUST NOT imply authorization for others.

**AEP-REQ-089**: Tenant boundaries MUST be preserved across artifact retrieval. An evaluator scoped to tenant A MUST NOT retrieve artifacts from tenant B.

**AEP-REQ-090**: Every authorization decision MUST be logged with session attribution.

### 13.3 Rate limiting

**AEP-REQ-091**: Servers MUST enforce rate limits per evaluator identity and per agent.

**AEP-REQ-092**: Rate-limited requests MUST return `-32020` with `data.retryAfterMs`.

**AEP-REQ-093**: Servers SHOULD publish rate-limit headers on successful responses (`X-RateLimit-Remaining`, `X-RateLimit-Reset`).

### 13.4 Input validation

**AEP-REQ-094**: Servers MUST validate scenarios against `scenario.schema.json` before execution.

**AEP-REQ-095**: Servers MUST validate injected context against declared injection-point schemas.

**AEP-REQ-096**: Servers MUST validate turn inputs against the agent's `inputSchema`.

### 13.5 Secret handling

**AEP-REQ-077**: Secrets MUST NOT appear in URLs, query strings, error messages, logs, traces, or findings.

**AEP-REQ-097**: Real credentials MUST NOT be accepted as injected context, even in development environments.

### 13.6 Audit logging

**AEP-REQ-098**: Servers MUST log authentication attempts, authorization decisions, session lifecycle transitions, and errors.

**AEP-REQ-099**: Audit logs MUST be tamper-evident, retained at least 90 days, and queryable by session, evaluator, and agent.

### 13.7 Cryptographic baseline

**AEP-REQ-100**: Transport MUST use TLS 1.3 or higher.

**AEP-REQ-101**: Integrity hashes MUST use SHA-256 or stronger.

### 13.8 Running in CI and shared infrastructure

The expected deployment pattern for AEP is not "always-on production endpoint" but "short-lived, scoped evaluation environment." CI pipelines, shared eval clusters, and ad-hoc test harnesses are first-class use cases. These patterns are normative guidance, not decorative:

**AEP-REQ-121**: Servers SHOULD support *ephemeral evaluator tokens* — credentials that are issued per evaluation run, scoped to a specific set of agents and modes, and automatically expire at the end of the run or a fixed TTL (recommended: ≤24 hours). Long-lived API keys MUST NOT be required for evaluation; they are an anti-pattern.

**AEP-REQ-122**: Servers SHOULD support *scoped evaluation sessions* — sessions whose authorization is restricted at creation time to specific agents, modes, extensions, and (where applicable) datasets. A scoped session MUST NOT be usable to evaluate agents outside its declared scope.

**AEP-REQ-123**: Servers SHOULD support *time-bound access* — evaluator credentials and session scopes with declared `notAfter` timestamps. Attempts to use a credential or scope after its `notAfter` MUST be rejected with `-32010`.

These three patterns together enable a CI workflow to: (a) request a short-lived token scoped to the agent under test, (b) run evaluations against that scope, and (c) have the token auto-expire without human cleanup. They address the most common real-world adoption question: "how do I run AEP safely in CI or shared infra?"

See [`../docs/THREAT-MODEL.md`](../docs/THREAT-MODEL.md) for the detailed threat analysis.

## 14. Capability Negotiation and Versioning

**AEP-REQ-102**: Interoperability is determined by the intersection of capabilities declared at `aep.initialize`, not by version comparison.

**AEP-REQ-103**: Servers MUST advertise only capabilities they fully implement.

**AEP-REQ-104**: Breaking changes to method semantics MUST increment the minor version (0.x) or major version (1.x+).

**AEP-REQ-105**: Additive capabilities MUST NOT require a version bump.

**AEP-REQ-106**: Vendor extensions MUST use reverse-DNS prefixes (e.g. `com.example.drift`) and MUST NOT override required AEP semantics.

## 15. Conformance by Role

AEP has three implementer roles. The authoritative per-role requirement lists are in [`../docs/CONFORMANCE.md`](../docs/CONFORMANCE.md); this section describes the roles and the rules for claiming compliance.

### 15.1 AEP-Agent
Applies to: any agent published for AEP evaluation.

An agent MUST declare at least {blackbox + graybox}. Richer mode sets are additive. Agents declaring modes beyond graybox MUST honor all requirements for those modes. See [`../docs/CONFORMANCE.md`](../docs/CONFORMANCE.md) §"AEP-Agent" for the requirement list.

### 15.2 AEP-Server
Applies to: any server implementing the AEP wire protocol.

A server MUST implement every requirement applicable to servers in the core and every requirement for any optional extension it claims support for. See [`../docs/CONFORMANCE.md`](../docs/CONFORMANCE.md) §"AEP-Server" for the requirement list.

### 15.3 AEP-Evaluator
Applies to: any client conducting evaluations against an AEP-Server.

See [`../docs/CONFORMANCE.md`](../docs/CONFORMANCE.md) §"AEP-Evaluator" for the requirement list.

### 15.4 Conformance declaration

**AEP-REQ-107**: Implementations claiming compliance MUST publish a declaration at `/.well-known/aep-compliance.json`:
```json
{
  "aepVersion": "0.1",
  "roles": ["AEP-Server", "AEP-Agent"],
  "extensions": ["policy-profile", "red-teaming"],
  "declaredAt": "2026-04-17T00:00:00Z",
  "exceptions": []
}
```

**AEP-REQ-108**: Exceptions (deviations from full compliance) MUST be listed explicitly. An implementation with unlisted exceptions is non-compliant.

## 16. Reference Implementation and Test Suite

A companion repository (`aep-reference`, planned v0.2) will host a minimal compliant server and client. When published, it is illustrative — servers passing its interaction pattern but failing other MUST-level requirements are not compliant.

A conformance test suite is planned for v0.2. Each test will reference an AEP-REQ identifier from this document. The absence of the suite does not reduce the normative force of MUST-level requirements.

## 17. Open Questions (v0.2 Candidates)

The following are known design gaps carried to v0.2:

1. Canonical trace serialisation format (binary framing for large traces)
2. Early-abort signalling via the streaming extension (§11.5)
3. Cost attribution as a first-class protocol concern
4. Multi-agent coordination scenarios
5. Cross-version replay semantics
6. Sandbox fidelity verification hooks
