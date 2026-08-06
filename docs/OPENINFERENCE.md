# OpenInference Trace Projection Extension

*A normative mapping from the AEP Trace to an OpenInference span tree, so every conforming Trace can land in Phoenix, Langfuse, Arize, or any OTLP backend without hand-written glue.*

**Extension ID:** `openinference-projection` (proposed) · **Spec section:** none yet — this is a draft proposal · **Status:** draft, revision 1

---

This document proposes an optional AEP extension defining a deterministic projection from a sealed AEP `Trace` (spec §10, [`trace.schema.json`](../schemas/trace.schema.json)) to an [OpenInference](https://github.com/Arize-ai/openinference) span tree carried over OpenTelemetry. The projection is a pure function of the Trace: it introduces no protocol dependency, no new data on the wire during evaluation, and no change to sealing or replay semantics. Its entire value is that the mapping is *normative* — two independent implementations projecting the same sealed Trace produce the same span tree, so LLM-observability tooling that speaks OpenInference works against any conforming implementation for free.

Requirement identifiers in this document use a provisional `AEP-REQ-OI-NNN` series. If the extension is adopted, they would be renumbered into the global `AEP-REQ-NNN` registry and indexed in [`CONFORMANCE.md`](./CONFORMANCE.md).

## Why this extension exists

The Observability Correlation extension (spec §11.9) answers *"which externally-collected telemetry corresponds to this AEP Trace?"* — it records correlation identifiers (`traceId`/`spanId`) inside the Trace so a client can join AEP's record against spans some other instrumentation already emitted. It deliberately stops there: it does not say what AEP's own record *looks like* as telemetry.

That leaves a gap on the other side. The AEP Trace is already a complete span tree in everything but name: a session with timed turns, each carrying timed tool calls, model calls with model names and token counts, and policy events. The LLM-observability ecosystem has converged on OpenInference semantic conventions (over OpenTelemetry) as the lingua franca for exactly this shape — Arize Phoenix consumes it natively, Langfuse and other backends ingest it via OTLP. But because AEP defines no mapping, every implementation that wants its evaluation traces visible in those tools hand-writes the projection — and each hand-written projection makes different choices about span kinds, attribute names, and ID derivation, so no two are comparable. Most implementations simply skip it.

Standardizing the projection costs the protocol nothing:

- **No dependency.** The Trace remains self-contained and replayable without any external system (§10). The projection reads the sealed Trace; nothing reads back.
- **No wire change.** Evaluation sessions, streaming, and sealing are untouched. The projection is computed after sealing, by whoever holds the Trace — client-side or server-side.
- **No new semantics.** Every field the projection emits already exists in the Trace. The extension only fixes *where each field goes* in OpenInference terms.

What it buys: every conforming server's Traces become drop-in visible in Phoenix, Langfuse, Arize, and any OTLP collector — with identical span trees regardless of which implementation produced them, so dashboards, saved queries, and evaluators built against one AEP implementation's projected traces work against all of them.

### Relationship to Observability Correlation (§11.9)

The two extensions are complementary and independent:

- **§11.9** correlates the AEP Trace with spans *someone else* collected (the deployment's own APM instrumentation).
- **This extension** projects the AEP Trace *itself* into span form.

When both are present, the recorded §11.9 identifiers become OpenTelemetry **span links** on the projected spans (see [Correlation identifiers become links](#correlation-identifiers-become-links)) — the projected tree points at the externally-collected tree without pretending to be it.

### Relationship to Operation Context Resolution

None required. [`OPERATION-CONTEXT.md`](./OPERATION-CONTEXT.md) flows context *into* AEP from an externally observed operation; this extension flows the AEP record *out* to observability tooling. A team using both gets a closed loop — production observation → resolved context → AEP session → projected span tree next to the production telemetry — but neither depends on the other.

## Design principles

1. **Pure function.** The projection is computed from the sealed Trace and nothing else. No network access, no clock reads, no randomness.
2. **Deterministic to the byte.** Same sealed Trace in, same span tree out — including trace/span IDs, which are derived by hashing stable Trace coordinates rather than generated.
3. **Never synthesize.** If the Trace doesn't carry content (a digest-only model call under a restrictive traceability contract, a redacted field), the projection doesn't invent it. Absence in the Trace is absence in the projection.
4. **Lossless where OpenInference has no home.** AEP fields with no OpenInference equivalent ride along under an `aep.*` attribute namespace rather than being dropped.
5. **Sensitivity travels with the data.** Sensitivity labels and redaction records project into attributes so downstream routing can honor them.

## The span tree

```
AGENT  ─ session root span            ← Trace
 ├── CHAIN  ─ turn 0                  ← TraceTurn (AGENT + graph.node.id for composite sessions)
 │    ├── LLM       ─ model call      ← ModelCallRecord
 │    ├── TOOL      ─ tool call       ← ToolCallEvent
 │    ├── GUARDRAIL ─ policy event    ← PolicyEvent
 │    └── (span events)               ← ReasoningStep[]
 ├── CHAIN  ─ turn 1
 │    └── ...
 └── (span events)                    ← DecisionNode[], TraceError[] without turnIndex
```

Children are emitted in `startedAt` order (ties broken by array order in the Trace). Finer-grained interleaving than timestamps record is not reconstructed — the Trace is the limit of fidelity.

## Mapping

All attribute names below are OpenInference semantic conventions unless prefixed `aep.`. Complex values (objects, arrays) are serialized to JSON strings using JCS canonical form (RFC 8785) — the same canonicalization the spec already uses for Trace signing (AEP-REQ-124) — so attribute bytes are deterministic.

### Trace → root span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `AGENT` |
| span name | `agentId`, or `"aep.session"` for composite sessions (no top-level `agentId`) |
| start / end | `startedAt` / `completedAt` (fallback: `completedAt` of last turn; else `startedAt`) |
| `session.id` | `sessionId` |
| status | `OK` when `finalOutcome.status` = `completed`; `ERROR` otherwise |
| `aep.trace_id` | `traceId` |
| `aep.version` | `aepVersion` |
| `aep.test_mode` | `testMode` |
| `aep.agent_id` / `aep.agent_version` | `agentId` / `agentVersion` (when present) |
| `aep.final_outcome.status` | `finalOutcome.status` |
| `aep.final_outcome.summary` | `finalOutcome.summary` (when present) |
| `aep.sealed` | `sealed` |
| `aep.canonical_digest` | `canonicalDigest` (when present) — lets a backend viewer verify which sealed artifact a span tree came from |
| `aep.sensitivity.level` / `aep.sensitivity.tags` | `sensitivity` (when present) |

Trace-level `decisionPath` nodes and `errors` entries carrying no `turnIndex` become span events on the root span (below). `injectedContext` is **not** projected — it is replay material, not telemetry, and routinely the most sensitive content in the Trace.

### TraceTurn → turn span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `CHAIN` for single-agent sessions; `AGENT` for composite-session turns |
| span name | `"turn <turnIndex>"`; for composite turns, `nodeId` |
| parent | root span |
| start / end | `startedAt` / `completedAt` |
| `input.value` / `input.mime_type` | `input` — see [Content and MIME rules](#content-and-mime-rules) |
| `output.value` / `output.mime_type` | `output` — same rules |
| `graph.node.id` | `nodeId` (composite sessions; OpenInference graph conventions) |
| `aep.turn.index` | `turnIndex` |
| `aep.turn.id` | `turnId` (when present) |
| `aep.turn.input_digest` / `aep.turn.output_digest` | `inputDigest` / `outputDigest` |
| `aep.sensitivity.input.level` / `aep.sensitivity.output.level` | `inputSensitivity.level` / `outputSensitivity.level` (when present) |

`latencyMs` is not projected as an attribute; it is redundant with span duration.

### ModelCallRecord → LLM span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `LLM` |
| span name | `model` |
| parent | enclosing turn span |
| start / end | see [Timing rules](#timing-rules) |
| `llm.model_name` | `model` |
| `llm.invocation_parameters` | `parameters` as JCS JSON string (when present) |
| `input.value` / `input.mime_type` | `prompt` / `text/plain` — **only when `prompt` is present** (traceability contract `modelCalls: "full"`) |
| `output.value` / `output.mime_type` | `response` / `text/plain` — only when present |
| `llm.token_count.prompt` | `inputTokens` (when present) |
| `llm.token_count.completion` | `outputTokens` (when present) |
| `llm.token_count.total` | `inputTokens + outputTokens` (only when both present) |
| `aep.model_call.id` | `modelCallId` |
| `aep.model_call.prompt_digest` / `aep.model_call.response_digest` | `promptDigest` / `responseDigest` (when present) |

When the traceability contract exposes only digests, the LLM span still exists — with timing, model name, and token counts — carrying digests instead of content. The projection MUST NOT fabricate `input.value`/`output.value` from digests.

### ToolCallEvent → TOOL span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `TOOL` |
| span name | `toolName` |
| parent | enclosing turn span |
| start / end | `startedAt` / `completedAt` (fallback: zero-duration at `startedAt`) |
| `tool.name` | `toolName` |
| `input.value` / `input.mime_type` | `arguments` — see [Content and MIME rules](#content-and-mime-rules) |
| `output.value` / `output.mime_type` | `result` — same rules |
| status | `ERROR` when `error` present, with an OTel `exception` span event (`exception.message` = `error`); `OK` otherwise |
| `aep.tool_call.id` | `toolCallId` |
| `aep.tool_call.sandboxed` | `sandboxed` (when present) |
| `aep.tool_call.sandbox_fidelity` | `sandboxFidelity` (when present) — evaluation-specific signal observability tools don't model: whether the tool result came from a stub, a recording, or the real thing |

### PolicyEvent → GUARDRAIL span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `GUARDRAIL` |
| span name | `ruleId` |
| parent | enclosing turn span |
| start / end | `timestamp` / `timestamp` (zero duration — PolicyEvents are instants) |
| status | `ERROR` when `outcome` is `refused` or `escalated`; `OK` otherwise |
| `aep.policy.event_id` | `policyEventId` |
| `aep.policy.rule_id` | `ruleId` |
| `aep.policy.outcome` | `outcome` |
| `aep.policy.severity` | `severity` (when present) |
| `aep.policy.matched_condition` | `matchedCondition` (when present) |
| `aep.policy.action_taken` | `actionTaken` (when present) |

### ReasoningStep → span event on the turn span

Reasoning steps carry a single `timestamp` and no duration, so they project as OTel span events rather than spans:

- event name: `aep.reasoning`
- event time: `timestamp` (fallback: turn `startedAt`)
- attributes: `aep.reasoning.step_id`, `aep.reasoning.type`, `aep.reasoning.summary` (when present), `aep.reasoning.detail` (when present — governed by the traceability contract's reasoning level, which the Trace has already applied)

### DecisionNode → span event on the root span

- event name: `aep.decision`
- event time: `timestamp` (fallback: root span start)
- attributes: `aep.decision.node_id`, `aep.decision.type`, `aep.decision.rationale` (when present), `aep.decision.alternatives` (JCS JSON string, when present)

### TraceError → exception span event

Each `errors[]` entry projects as an OTel `exception` event — on the matching turn span when `turnIndex` is present, else on the root span:

- `exception.type`: `"aep.error"` · `exception.message`: `message` (when present)
- `aep.error.id`: `errorId` · `aep.error.code`: `code` · `aep.error.recoverable`: `recoverable` (when present)

### Content and MIME rules

For every `input.value` / `output.value` pair:

- Trace value is a **string** → emit verbatim, `mime_type` = `text/plain`.
- Trace value is any other JSON value (object, array, number, boolean, null) → emit its JCS (RFC 8785) serialization, `mime_type` = `application/json`.
- Trace field is **absent** (not exposed by the traceability contract, redacted, or never captured) → emit neither `*.value` nor `*.mime_type`. Digest attributes (`aep.*_digest`) still project when the Trace carries them.

### Timing rules

OTel spans require start and end timestamps; the projection never invents timing the Trace doesn't support:

- Elements with `startedAt`/`completedAt` use them directly (RFC 3339 → epoch nanoseconds).
- Elements with `startedAt` only: zero-duration span at `startedAt`.
- `ModelCallRecord` carries no timestamps in the v0.1 schema. Its LLM span starts at the enclosing turn's `startedAt` and spans `latencyMs` when present (else zero duration), and MUST set `aep.timing.inferred = true` so consumers know the placement is approximate. See [Core hooks](#core-hooks-needed-in-the-spec) for the additive fix.

### ID derivation

Span identity is derived, not generated, so the projection is reproducible and collision-free across implementations. Let `H(s)` be SHA-256 over the UTF-8 bytes of `s`, and `hexN(b)` the lowercase hex of the first N bytes.

| Identifier | Derivation |
|---|---|
| OTel `trace_id` (16 bytes) | `hex16(H("aep-oi:" + traceId))` |
| root `span_id` (8 bytes) | `hex8(H("aep-oi:" + traceId + ":session"))` |
| turn `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + turnIndex))` |
| model-call `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + turnIndex + ":model:" + modelCallId))` |
| tool-call `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + turnIndex + ":tool:" + toolCallId))` |
| policy-event `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + turnIndex + ":policy:" + policyEventId))` |

`traceId` here is the AEP Trace's own `traceId` (already required to be high-entropy per AEP-REQ-126, so the derived OTel IDs inherit its unpredictability). The `"aep-oi:"` domain-separation prefix keeps derived IDs from colliding with any other hash-derived ID scheme over the same `traceId`.

### Correlation identifiers become links

When the Trace carries §11.9 `observability` blocks with `provider: "opentelemetry"`, the projection MUST NOT adopt those IDs as the projected spans' own IDs — the externally-collected spans already exist in some backend under those IDs, and emitting a second span under the same identity would collide or silently merge. Instead:

- A session-level `observability` block projects as an OTel **span link** on the root span, targeting the recorded external `traceId`/`spanId`.
- A per-turn `observability` block projects as a span link on that turn's span.
- Each link carries `aep.link.provider` = the block's `provider`.

`observability` blocks with non-OpenTelemetry providers (no well-formed trace/span identity to link to) project as attributes instead: `aep.observability.provider`, `aep.observability.trace_id`, `aep.observability.span_id` on the corresponding span.

## Redaction and sensitivity

The projection operates on the sealed Trace *after* redaction, so redacted content is structurally absent and cannot resurface. In addition:

- `redactions[]` records project as `aep.redacted = true` on any span whose mapped source path falls under a redaction record's `path`, so a viewer can distinguish "never captured" from "captured and redacted".
- Trace- and turn-level sensitivity labels project into `aep.sensitivity.*` attributes as specified in the tables above. Deciding which telemetry backends may receive traces of a given sensitivity level is deployment policy, out of scope here — but because the labels travel in-band, an OTel collector can enforce that policy with a standard attribute-based filter/routing rule, with no AEP-specific logic.

## Optional server-side projection endpoint

The projection needs no server support — any client holding a Trace can compute it, and a conforming client-side projector works against every AEP server today. Servers that want to serve it directly (e.g. to feed a collector without shipping full Traces to clients) advertise the extension and expose:

```
GET /aep/traces/{traceId}/projections/openinference
```

JSON-RPC equivalent: `aep.trace.projectOpenInference`. The response is the span tree in OTLP/JSON form (`ExportTraceServiceRequest`). The endpoint is defined only for sealed Traces; requesting the projection of an unsealed Trace fails with `-32030` (`invalid_state_transition`). Authorization, mode-based access, and existence-hiding semantics are identical to fetching the Trace itself — the projection reveals a subset of the Trace, so it MUST NOT be reachable by any caller who could not fetch the Trace.

## Requirements (provisional)

**AEP-REQ-OI-001**: A conforming projection MUST be a pure function of a single sealed Trace: it MUST NOT read external systems, clocks, or configuration that varies the output, and MUST NOT be defined for unsealed Traces.

**AEP-REQ-OI-002**: Two conforming projections of the same sealed Trace MUST produce identical span trees — identical span/trace IDs, parentage, names, kinds, timestamps, attributes, events, and links. (Resource-level attributes added by an exporter, e.g. `service.name`, are outside the projection and exempt.)

**AEP-REQ-OI-003**: Trace and span IDs MUST be derived exactly per [ID derivation](#id-derivation). Implementations MUST NOT substitute generated or externally-recorded IDs for derived ones.

**AEP-REQ-OI-004**: Span kinds MUST follow the mapping tables (`AGENT` root, `CHAIN`/`AGENT` turns, `LLM` model calls, `TOOL` tool calls, `GUARDRAIL` policy events). Implementations MUST NOT remap kinds or invent spans for Trace elements this document maps to span events.

**AEP-REQ-OI-005**: The projection MUST NOT synthesize content absent from the Trace. Where the Trace carries only digests, the projection carries only digests; `input.value`/`output.value` MUST be emitted only from content literally present in the Trace, byte-preserved per the [Content and MIME rules](#content-and-mime-rules).

**AEP-REQ-OI-006**: Sensitivity labels present in the Trace MUST project into the corresponding `aep.sensitivity.*` attributes, and spans whose source content falls under a `redactions[]` record MUST carry `aep.redacted = true`. `injectedContext` MUST NOT be projected.

**AEP-REQ-OI-007**: Timestamps MUST follow the [Timing rules](#timing-rules); any span whose timing is inferred rather than recorded MUST carry `aep.timing.inferred = true`.

**AEP-REQ-OI-008**: §11.9 `observability` blocks with `provider: "opentelemetry"` MUST project as span links, never as the projected spans' own identities.

**AEP-REQ-OI-009**: Servers advertising `openinference-projection` in `capabilities.extensions` MUST implement the projection endpoint for sealed Traces with authorization and existence-hiding semantics identical to Trace retrieval, and MUST reject projection of unsealed Traces with `-32030`.

**AEP-REQ-OI-010**: A server-side projection MUST equal the projection a conforming client would compute from the fetched Trace (AEP-REQ-OI-002 applies across the client/server boundary).

## Core hooks needed in the spec

One additive schema change would remove the only approximation in the mapping:

1. **`ModelCallRecord.startedAt` / `completedAt`** (optional, RFC 3339) — v0.1 records only `latencyMs`, forcing the inferred-timing rule above. Optional timestamps are additive under the compatibility policy (§14, AEP-REQ-105); Traces without them continue to project with `aep.timing.inferred = true`.

No other hook is required: every other field the projection consumes is already in the v0.1 Trace schema.

## Worked example

A one-turn Trace fragment:

```json
{
  "traceId": "q8Xk2nRvB4tYw9zL5mCj7A",
  "sessionId": "sess_p3Fh6dKm1sVx8bNq4gWz2E",
  "agentId": "example.support-agent",
  "testMode": "graybox",
  "turns": [{
    "turnIndex": 0,
    "input": { "question": "Where is my refund for order 123?" },
    "output": { "answer": "Refund issued, arriving in 3-5 days." },
    "startedAt": "2026-08-06T12:00:00Z",
    "completedAt": "2026-08-06T12:00:04Z",
    "modelCalls": [{ "modelCallId": "mc_1", "model": "claude-sonnet-5",
                     "inputTokens": 812, "outputTokens": 96, "latencyMs": 1800 }],
    "toolCalls": [{ "toolCallId": "tc_1", "toolName": "lookupOrder",
                    "arguments": { "orderId": "123" },
                    "result": { "status": "refunded" },
                    "sandboxed": true, "sandboxFidelity": "recorded",
                    "startedAt": "2026-08-06T12:00:01Z",
                    "completedAt": "2026-08-06T12:00:02Z" }]
  }],
  "finalOutcome": { "status": "completed" }
}
```

projects to (IDs abbreviated):

```
trace_id = hex16(H("aep-oi:q8Xk2nRvB4tYw9zL5mCj7A"))

AGENT "example.support-agent"                     span_id = H(...":session")
  session.id = "sess_p3Fh6dKm1sVx8bNq4gWz2E"
  aep.test_mode = "graybox" · status = OK
  │
  └── CHAIN "turn 0"                              span_id = H(...":turn:0")
        input.value  = {"question":"Where is my refund for order 123?"}   (application/json, JCS)
        output.value = {"answer":"Refund issued, arriving in 3-5 days."}  (application/json, JCS)
        │
        ├── LLM "claude-sonnet-5"                 span_id = H(...":turn:0:model:mc_1")
        │     llm.model_name = "claude-sonnet-5"
        │     llm.token_count.prompt = 812 · completion = 96 · total = 908
        │     aep.timing.inferred = true          (12:00:00 + 1800ms)
        │
        └── TOOL "lookupOrder"                    span_id = H(...":turn:0:tool:tc_1")
              tool.name = "lookupOrder"
              input.value  = {"orderId":"123"}    (application/json)
              output.value = {"status":"refunded"}
              aep.tool_call.sandbox_fidelity = "recorded"
```

Point an OTLP exporter at that span set and the session renders in Phoenix or Langfuse as an agent trace with no further work.

## Non-goals

- **Not an exporter or transport spec.** How the projected spans reach a backend (OTLP/gRPC, OTLP/HTTP, in-process SDK) is out of scope; the projection defines the span tree, not its delivery.
- **No reverse mapping.** OpenInference span trees do not project back into Traces; the Trace remains the canonical record and the projection is derived, never authoritative.
- **No streaming projection.** The projection is defined only over sealed Traces. Projecting live from stream events (§11.5) is deferred — it would need rules for spans whose end is not yet known.
- **No backend query standardization.** Echoing §11.9's non-goal: clients query Phoenix/Langfuse/etc. with those products' own surfaces.
- **No evaluation-result projection.** `EvaluationResult` and `Finding` artifacts are not part of the Trace and are out of scope here; projecting scores as OpenInference span annotations is a plausible follow-up once this mapping has implementation experience.

## Open questions

1. **Composite-session graph conventions.** The mapping emits `graph.node.id` on composite-session turn spans, following OpenInference's agent-graph conventions. Those conventions are newer than the core span-kind vocabulary; if they shift, the CSE-specific rows of the mapping may need a revision independent of the rest.
2. **Reasoning as events vs. spans.** Reasoning steps project as span events because they carry no duration. Implementations with rich, long-running reasoning phases may prefer spans; that would require optional start/end timestamps on `ReasoningStep` (an additive core hook this proposal does not yet request).
3. **Attribute size limits.** Turn inputs/outputs can be large, and OTel backends commonly truncate attribute values. The projection is defined without size limits; whether to specify a normative truncation marker (e.g. `aep.truncated = true`) or leave truncation to exporters needs implementation experience.
