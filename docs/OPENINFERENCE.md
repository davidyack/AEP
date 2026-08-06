# OpenInference Trace Projection Extension

*A normative mapping from the AEP Trace to an OpenInference span tree, so every conforming Trace can land in Phoenix, Langfuse, Arize, or any OTLP backend without hand-written glue.*

**Extension ID:** `openinference-projection` (proposed) · **Spec section:** none yet — this is a draft proposal · **Status:** draft, revision 3

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

When both are present, well-formed §11.9 OpenTelemetry identifiers become OpenTelemetry **span links** on the projected spans (see [Correlation identifiers become links](#correlation-identifiers-become-links)) — the projected tree points at the externally-collected tree without pretending to be it.

### Relationship to Operation Context Resolution

None required. [`OPERATION-CONTEXT.md`](./OPERATION-CONTEXT.md) flows context *into* AEP from an externally observed operation; this extension flows the AEP record *out* to observability tooling. A team using both gets a closed loop — production observation → resolved context → AEP session → projected span tree next to the production telemetry — but neither depends on the other.

## Design principles

1. **Pure function.** The projection is computed from the sealed Trace and nothing else. No network access, no clock reads, no randomness.
2. **Deterministic to the byte.** Same sealed Trace in, same span tree out — including trace/span IDs, which are derived by hashing stable Trace coordinates rather than generated.
3. **Never synthesize.** If the Trace doesn't carry content (a digest-only model call under a restrictive traceability contract, a redacted field), the projection doesn't invent it. Absence in the Trace is absence in the projection. The same rule covers derived facts: the projection never infers what the Trace doesn't state (e.g. no parsing a model string into a provider).
4. **Lossless for evaluation-relevant fields.** Trace fields with no OpenInference equivalent ride along under an `aep.*` attribute namespace rather than being dropped. Fields the projection deliberately does not carry are enumerated in [Fields deliberately not projected](#fields-deliberately-not-projected) with reasons — nothing is dropped silently.
5. **Sensitivity travels with the data.** Sensitivity labels and redaction records project into attributes so downstream routing can honor them.

## Convention pinning

OpenInference semantic conventions version independently of AEP. A determinism guarantee defined "by reference to OpenInference" would drift: two implementations pinned to different upstream revisions could both claim conformance while emitting different attribute names.

This revision therefore **freezes** its OpenInference vocabulary: the attribute names, span kinds, and event conventions written in this document are normative copies taken from OpenInference semantic conventions as published in `openinference-semantic-conventions` release **[0.1.31](https://pypi.org/project/openinference-semantic-conventions/0.1.31/)** (released 2026-08-01). The package release is the versioning anchor because upstream offers no other: the [spec text itself](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md) carries no independent version marker. An upstream change after that release does not change this mapping. Adopting a newer conventions release is a **revision of this extension**, reflected in the `conventionsVersion` advertisement field below — never a silent re-mapping.

## Extension advertisement

The extension uses the standard advertisement mechanism: `openinference-projection` appears in `capabilities.extensions`, with structured metadata in `capabilities.extensionData` per spec §5.1 (AEP-REQ-134 through AEP-REQ-140).

```json
{
  "capabilities": {
    "extensions": ["openinference-projection"],
    "extensionData": {
      "openinference-projection": {
        "version": "0.1.0",
        "payload": {
          "conventionsVersion": "0.1.31",
          "projectionEndpoint": true,
          "maxValueBytes": 1048576
        }
      }
    }
  }
}
```

Payload fields:

- `conventionsVersion` (string, required) — the `openinference-semantic-conventions` release whose vocabulary the server's projection emits. For this extension revision the value is `"0.1.31"`; a different value signals a different extension revision, not a configuration choice.
- `projectionEndpoint` (boolean, required) — whether the server exposes the [server-side projection endpoint](#optional-server-side-projection-endpoint). `false` declares that projections the deployment publishes through its own pipeline — typically into a shared collector its consumers already read — conform to this extension, without serving them over the AEP surface. With no endpoint there is no artifact to fetch, so the claim is checked via AEP-REQ-OI-013: fetch the Trace, project it locally under the advertised `maxValueBytes`, and compare against the spans found in the collector.
- `maxValueBytes` (integer, optional) — the truncation limit `L` the server's projections apply to `input.value`/`output.value`, per [Value size and truncation](#value-size-and-truncation). Absent means unbounded.

Until the extension is registered, vendor implementations MUST use a reverse-DNS identifier (e.g. `com.example.openinference-projection`) per AEP-REQ-106.

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

All attribute names below are OpenInference semantic conventions (frozen per [Convention pinning](#convention-pinning)) unless prefixed `aep.`. Complex values (objects, arrays) are serialized to JSON strings using JCS canonical form (RFC 8785) — the same canonicalization the spec already uses for Trace signing (AEP-REQ-124) — so attribute bytes are deterministic.

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
| `aep.final_outcome.error_ref` | `finalOutcome.errorRef` (when present) — the route from a red root span to its terminal cause |
| `aep.canonical_digest` | `canonicalDigest` (when present) — lets a backend viewer verify which sealed artifact a span tree came from |
| `aep.replay.replayable` | `replay.replayable` (when present) |
| `aep.replay.completeness` | `replay.completeness` (when present) — "can I replay this trace" is answerable from the span viewer |
| `aep.replay.nondeterministic_points` | `replay.nondeterministicPoints` as JCS JSON string (when present and non-empty) |
| `aep.redactions.count` | number of `redactions[]` records (when `redactions` present) |
| `aep.sensitivity.level` / `aep.sensitivity.tags` | `sensitivity` (when present) |

Trace-level `decisionPath` nodes and `errors` entries carrying no `turnIndex` become span events on the root span (below).

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

### ModelCallRecord → LLM span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `LLM` |
| span name | `model` |
| parent | enclosing turn span |
| start / end | see [Timing rules](#timing-rules) |
| `llm.model_name` | `model`, verbatim |
| `llm.invocation_parameters` | `parameters` as JCS JSON string (when present) |
| `input.value` / `input.mime_type` | `prompt` / `text/plain` — **only when `prompt` is present** (traceability contract `modelCalls: "full"`) |
| `output.value` / `output.mime_type` | `response` / `text/plain` — only when present |
| `llm.token_count.prompt` | `inputTokens` (when present) |
| `llm.token_count.completion` | `outputTokens` (when present) |
| `llm.token_count.total` | `inputTokens + outputTokens` (only when both present) |
| `aep.model_call.id` | `modelCallId` |
| `aep.model_call.prompt_digest` / `aep.model_call.response_digest` | `promptDigest` / `responseDigest` (when present) |

When the traceability contract exposes only digests, the LLM span still exists — with timing, model name, and token counts — carrying digests instead of content. The projection MUST NOT fabricate `input.value`/`output.value` from digests.

`model` is an opaque string in the Trace schema. The projection maps it verbatim to `llm.model_name` and MUST NOT parse it to derive `llm.provider`, `llm.system`, or any other attribute — a string like `"openai/gpt-4"` is one implementation's naming choice, and splitting it is exactly the kind of inference that makes two projections silently diverge.

### ToolCallEvent → TOOL span

| OpenInference / OTel | AEP source |
|---|---|
| span kind | `TOOL` |
| span name | `toolName` |
| parent | enclosing turn span |
| start / end | see [Timing rules](#timing-rules) |
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

PolicyEvents, like reasoning steps below, carry a single timestamp and no duration — yet they project as spans while reasoning projects as events. The asymmetry is deliberate: `GUARDRAIL` is a first-class OpenInference span kind that backends filter, count, and visualize as spans, so a policy event materializes where those tools look for it; reasoning has no OpenInference span kind, so events on the turn span are its honest home.

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

Each `errors[]` entry projects as an OTel `exception` event — on the turn span of the first turn (in array order) whose `turnIndex` field equals the record's `turnIndex`; on the root span when `turnIndex` is absent **or matches no turn**:

- `exception.type`: `"aep.error"` · `exception.message`: `message` (when present)
- `aep.error.id`: `errorId` · `aep.error.code`: `code` · `aep.error.recoverable`: `recoverable` (when present)

### Content and MIME rules

For every `input.value` / `output.value` pair:

- Trace value is a **string** → emit verbatim, `mime_type` = `text/plain`.
- Trace value is any other JSON value (object, array, number, boolean, null) → emit its JCS (RFC 8785) serialization, `mime_type` = `application/json`.
- Trace field is **absent** (not exposed by the traceability contract, redacted, or never captured) → emit neither `*.value` nor `*.mime_type`. Digest attributes (`aep.*_digest`) still project when the Trace carries them.

### Value size and truncation

Turn inputs/outputs can be large, and OTel backends commonly cap attribute values. Truncation left to exporters would break determinism exactly at the observable boundary — the projection would be deterministic while what lands in the backend is not. Truncation is therefore part of the projection, not the transport:

- By default the projection is **unbounded**: values emit whole.
- A deployment MAY declare a byte limit **L** (advertised as `maxValueBytes` for server-side projections). When declared, every `input.value`/`output.value` whose UTF-8 encoding exceeds L bytes is cut to the longest prefix ≤ L bytes that ends on a UTF-8 code-point boundary, and the span additionally carries `aep.truncation.input = true` (respectively `aep.truncation.output = true`) and `aep.truncation.input_digest` (respectively `aep.truncation.output_digest`) — `sha256:` hex over the full untruncated value bytes. The digest makes the cut detectable and lets two projections be compared for divergence even when only one truncated; it is scoped under `aep.truncation.*` so it cannot be misread as the adjacent `aep.turn.input_digest`, which hashes the Trace's *normalized* input under an algorithm the schema does not pin — the two digests are different facts and will not agree.
- A value whose untruncated `mime_type` would be `application/json` MUST be emitted with `mime_type = text/plain` when truncation is applied: a prefix of a JSON document is not JSON, and consumers that parse `input.value` on the JSON MIME type (Phoenix and Langfuse both do, to render structured input) would show a parse failure instead of content — making the truncation path degrade worse than not truncating. The `aep.truncation.*` marker tells a consumer why the type changed; a viewer showing a truncated JSON string is strictly better than one showing an error.
- Two conforming projections with the same L produce identical output; the [determinism requirement](#requirements-provisional) is parameterized by L, with unbounded as the default.
- Downstream truncation (collector or backend limits) is outside the projection's conformance boundary; deployments SHOULD configure attribute-value limits at or above L so the projected bytes survive intact.

### Timing rules

OTel spans require start and end timestamps; the projection never invents timing the Trace doesn't support without saying so. Any span with an inferred start **or** end carries `aep.timing.inferred = true`.

- Elements with recorded `startedAt`/`completedAt` use them directly (RFC 3339 → epoch nanoseconds). Nothing inferred.
- **Root span**: start = `startedAt` (recorded, required by the schema). End = `completedAt` when present (recorded); else the last turn's `completedAt`; else `startedAt` (zero duration) — both fallbacks flag the root span inferred. `sealedAt`, though guaranteed present on sealed Traces, is deliberately not used: it bounds the session from above but includes post-completion sealing latency, and the last turn's end is the better estimate.
- **ToolCallEvent** without `completedAt`: end = `startedAt + latencyMs` when `latencyMs` is present, else a zero-duration span at `startedAt`. Inferred either way.
- **ModelCallRecord** carries no timestamps in the v0.1 schema. Model-call spans within a turn are laid **end-to-end in array order**: the first starts at the turn's `startedAt`; each subsequent span starts where the previous one ends; each spans its own `latencyMs` (absent → zero duration). All are flagged inferred. End-to-end placement is chosen over starting every span at the turn's `startedAt` because overlapping spans render as parallel fan-out in Phoenix/Langfuse — a false claim about the agent's behavior in the exact tools this extension serves — while sequential placement matches the common sequential-call case and is equally deterministic. See [Core hooks](#core-hooks-needed-in-the-spec) for the additive fix that removes the inference entirely.

Sequential placement orders model-call spans against *each other*; it cannot place them correctly against recorded sibling spans. An inferred model-call span can therefore still overlap a tool-call span with real timestamps — the [worked example](#worked-example) shows exactly this — and a viewer will render apparent concurrency between a model call and the tool call it likely caused. The projection deliberately does not clamp inferred spans against recorded ones (that would stack a second inference on the first, and mis-render the genuinely-concurrent case): `aep.timing.inferred = true` is the signal that placement, not the Trace, produced the apparent overlap. The `ModelCallRecord` timestamp core hook removes the entire problem.

### ID derivation

Span identity is derived, not generated, so the projection is reproducible and collision-free across implementations. Let `H(s)` be SHA-256 over the UTF-8 bytes of `s`, and `hexN(b)` the lowercase hex of the first N bytes.

| Identifier | Derivation |
|---|---|
| OTel `trace_id` (16 bytes) | `hex16(H("aep-oi:" + traceId))` |
| root `span_id` (8 bytes) | `hex8(H("aep-oi:" + traceId + ":session"))` |
| turn `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + t))` |
| model-call `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + t + ":model:" + i))` |
| tool-call `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + t + ":tool:" + i))` |
| policy-event `span_id` | `hex8(H("aep-oi:" + traceId + ":turn:" + t + ":policy:" + i))` |

where `t` and `i` are decimal integers with no leading zeros: `t` is the turn's **position in the `turns` array**, and `i` is the element's **position in its containing array** (`modelCalls`, `toolCalls`, `policyEvents`).

Array positions — not `turnIndex`, `modelCallId`, `toolCallId`, or `policyEventId` — feed the hash, for two reasons. The schema requires those fields to be present but not unique: duplicate element IDs would collide two elements into one span, and a duplicated `turnIndex` is worse — every child path embeds the turn coordinate, so two turns sharing an index would collide their turn spans *and* every model/tool/policy span beneath them. Positions are unique by construction. And the element IDs are arbitrary strings, so hashing them invites delimiter injection (a `toolCallId` of `0:policy:x` colliding with a policy-event path); with positions, every variable component is either a decimal integer or the `[A-Za-z0-9_-]`-constrained `traceId`, and the path grammar is unambiguous with no escaping needed. The demoted fields still ride on the spans as attributes (`aep.turn.index`, `aep.*.id`). Array order is fixed in the sealed Trace, so position-based identity is exactly as stable as the Trace itself.

OpenTelemetry treats all-zero `trace_id`/`span_id` values as invalid. Should a derivation produce all zeros — astronomically improbable from SHA-256, but a normative spec closes the case — the hash input is re-derived with `":r1"` appended (then `":r2"`, and so on) until the result is nonzero. The escape is itself deterministic, so it does not weaken the identity guarantee.

`traceId` here is the AEP Trace's own `traceId` (already required to be high-entropy per AEP-REQ-126, so the derived OTel IDs inherit its unpredictability). The `"aep-oi:"` domain-separation prefix keeps derived IDs from colliding with any other hash-derived ID scheme over the same `traceId`.

### Correlation identifiers become links

A §11.9 `observability` block projects according to its `provider` and the well-formedness of its identifiers:

1. **`provider: "opentelemetry"` with well-formed IDs** — `traceId` is 32 lowercase hex characters and not all zeros, **and** `spanId` is 16 lowercase hex characters and not all zeros (W3C Trace Context) — projects as an OTel **span link** targeting the recorded external `traceId`/`spanId`: on the root span for the session-level block, on the turn's span for a per-turn block. Each link carries `aep.link.provider` = the block's `provider`.
2. **`provider: "opentelemetry"` with absent or malformed IDs** — the schema requires only `provider`, and the hex encoding is a SHOULD, so `{"provider": "opentelemetry"}` alone, or with a non-W3C value, is a valid Trace. No link can be formed; the block falls back to attributes on the corresponding span: `aep.observability.provider`, plus `aep.observability.trace_id` / `aep.observability.span_id` for whichever identifiers are present, verbatim.
3. **Any other `provider`** — no OTel identity to link to; same attribute fallback as branch 2.

In no branch does the projection adopt the recorded IDs as the projected spans' *own* identities — the externally-collected spans already exist in some backend under those IDs, and emitting a second span under the same identity would collide or silently merge.

## Redaction and sensitivity

The projection operates on the sealed Trace *after* redaction, so redacted content is structurally absent and cannot resurface. In addition:

- The root span carries `aep.redactions.count` whenever the Trace has a `redactions[]` array, so "was anything redacted" is answerable from any viewer.
- A span carries `aep.redacted = true` when redaction verifiably touched it. This is computable only for redaction records whose `path` parses as an RFC 6901 JSON Pointer: each projected span corresponds to exactly one Trace element with an unambiguous pointer (e.g. the second tool call of turn 0 is `/turns/0/toolCalls/1`), and a record touches a span when either pointer is a whole-segment prefix of the other — the record redacts content inside the element, or the element sits inside a redacted subtree. The **root span is excluded** from prefix matching: its element pointer is the empty string, which is a whole-segment prefix of every pointer, so matching it would set the flag on the root for any redaction anywhere and collapse its meaning into `aep.redactions.count > 0` — the count, which the root already carries, is the session-level signal. Records whose `path` does not parse as an RFC 6901 pointer (the schema currently allows free-text "trace paths") contribute to `aep.redactions.count` only; implementations MUST NOT guess at matching them, because heuristic matching is precisely the silent-divergence failure the sibling proposal documents in AEP-REQ-OC-019. See [Core hooks](#core-hooks-needed-in-the-spec) for the schema tightening that makes every record matchable.
- Trace- and turn-level sensitivity labels project into `aep.sensitivity.*` attributes as specified in the mapping tables. Deciding which telemetry backends may receive traces of a given sensitivity level is deployment policy, out of scope here — but because the labels travel in-band, an OTel collector can enforce that policy with a standard attribute-based filter/routing rule, with no AEP-specific logic.

## Fields deliberately not projected

Principle 4 promises no silent drops, so here is the explicit list:

| Field | Why not projected |
|---|---|
| `injectedContext` | Replay material, not telemetry — and routinely the most sensitive content in the Trace. |
| `turns[].stateSnapshot` | Same: replay material capturing internal agent state. |
| `seed` | Replay material; publishing it invites replaying outside AEP's mode-authorization controls. |
| `sealed` | Constant — the projection is defined only over sealed Traces, so the attribute would always be `true`. |
| `sealedAt`, `canonicalSerialization`, `signature` | Sealing envelope. `aep.canonical_digest` alone anchors a span tree to its sealed artifact; signature verification requires the Trace bytes anyway, so carrying the envelope in telemetry adds surface without adding verifiability. |
| `redactions[].reason` / `redactedAt` / `originalDigest` | Audit metadata. The projection carries the count and per-span `aep.redacted` flags; the audit detail stays in the Trace. |
| `turns[].latencyMs` | Redundant with span duration — `TraceTurn` requires both `startedAt` and `completedAt`, so turn duration is always recorded. |
| `toolCalls[].latencyMs` (when `completedAt` recorded) | Redundant with span duration. (It *is* consumed by the [Timing rules](#timing-rules) when `completedAt` is missing.) |

## Optional server-side projection endpoint

The projection needs no server support — any client holding a Trace can compute it, and a conforming client-side projector works against every AEP server today. Servers that want to serve it directly (e.g. to feed a collector without shipping full Traces to clients) advertise `projectionEndpoint: true` and expose:

```
GET /aep/traces/{traceId}/projections/openinference
```

JSON-RPC equivalent: `aep.trace.projectOpenInference`. The response is the span tree in OTLP/JSON form (`ExportTraceServiceRequest`). Two transport details are normative because generic tooling gets them wrong:

- **ID encoding.** In OTLP/JSON, `trace_id` and `span_id` MUST be encoded as lowercase hexadecimal strings (32 and 16 characters), per the OTLP specification's explicit override of protobuf's canonical JSON mapping. A generic protobuf-JSON serializer emits base64 for `bytes` fields and produces output that is silently non-interoperable — and non-conforming here.
- **Sealed only.** The endpoint is defined only for sealed Traces; requesting the projection of an unsealed Trace fails with `-32030` (`invalid_state_transition`).

Authorization, **mode-based access**, and existence-hiding semantics are identical to fetching the Trace itself. The projection reveals a subset of the Trace — including tool calls and (under a permissive traceability contract) prompts — so it MUST NOT be reachable by any caller who could not fetch the Trace, and the Trace schema's mode rule carries over: a projection of a Trace captured in `toolbox` MUST NOT be served to a caller authorized only for `blackbox`.

## Requirements (provisional)

**AEP-REQ-OI-001**: A conforming projection MUST be a pure function of a single sealed Trace and the declared truncation limit L (default: unbounded): it MUST NOT read external systems, clocks, or other configuration that varies the output, and MUST NOT be defined for unsealed Traces.

**AEP-REQ-OI-002**: Two conforming projections of the same sealed Trace under the same L MUST produce identical span trees — identical span/trace IDs, parentage, names, kinds, timestamps, attributes, events, and links. (Resource-level attributes added by an exporter, e.g. `service.name`, are outside the projection and exempt.)

**AEP-REQ-OI-003**: A conforming projection MUST emit the OpenInference vocabulary exactly as frozen in this document ([Convention pinning](#convention-pinning), `openinference-semantic-conventions` 0.1.31) and MUST NOT substitute names, kinds, or event conventions from any other OpenInference release. Adopting a newer release requires a revision of this extension and a changed `conventionsVersion` advertisement.

**AEP-REQ-OI-004**: Trace and span IDs MUST be derived exactly per [ID derivation](#id-derivation), with array positions — not `turnIndex` or element IDs — as the variable path components, and with the deterministic all-zero re-derivation escape. Implementations MUST NOT substitute generated or externally-recorded IDs for derived ones.

**AEP-REQ-OI-005**: Span kinds MUST follow the mapping tables (`AGENT` root, `CHAIN`/`AGENT` turns, `LLM` model calls, `TOOL` tool calls, `GUARDRAIL` policy events). Implementations MUST NOT remap kinds or invent spans for Trace elements this document maps to span events.

**AEP-REQ-OI-006**: The projection MUST NOT synthesize content or facts absent from the Trace. Where the Trace carries only digests, the projection carries only digests; `input.value`/`output.value` MUST be emitted only from content literally present in the Trace, byte-preserved per the [Content and MIME rules](#content-and-mime-rules) (subject to AEP-REQ-OI-010); and `ModelCallRecord.model` MUST map verbatim to `llm.model_name` — implementations MUST NOT parse it to emit `llm.provider`, `llm.system`, or any derived attribute.

**AEP-REQ-OI-007**: Sensitivity labels present in the Trace MUST project into the corresponding `aep.sensitivity.*` attributes. When the Trace carries `redactions[]`, the root span MUST carry `aep.redactions.count`; `aep.redacted = true` MUST be set on exactly the **non-root** spans matched by RFC 6901-parseable redaction paths under the whole-segment-prefix rule ([Redaction and sensitivity](#redaction-and-sensitivity)) — the root span is excluded from matching — and implementations MUST NOT apply heuristic matching to unparseable paths. `injectedContext`, `stateSnapshot`, and `seed` MUST NOT be projected.

**AEP-REQ-OI-008**: Timestamps MUST follow the [Timing rules](#timing-rules), including `latencyMs`-based end inference for tool calls and end-to-end sequential placement for model-call spans; any span whose start or end is inferred rather than recorded MUST carry `aep.timing.inferred = true`.

**AEP-REQ-OI-009**: §11.9 `observability` blocks MUST project per the three-branch rule in [Correlation identifiers become links](#correlation-identifiers-become-links): a span link only when `provider` is `"opentelemetry"` and both identifiers are well-formed W3C values; the `aep.observability.*` attribute fallback otherwise. Recorded identifiers MUST NOT become the projected spans' own identities in any branch.

**AEP-REQ-OI-010**: When a truncation limit L is declared, truncation MUST be applied within the projection per [Value size and truncation](#value-size-and-truncation) — UTF-8-boundary prefix cut, `aep.truncation.input`/`aep.truncation.output` marker, full-value digest, and `mime_type` downgraded to `text/plain` for values that would otherwise be `application/json` — and MUST NOT be delegated to exporters or collectors.

**AEP-REQ-OI-011**: Servers advertising `openinference-projection` MUST publish the extensionData payload of [Extension advertisement](#extension-advertisement) (`conventionsVersion`, `projectionEndpoint`, and `maxValueBytes` when a limit applies). Servers advertising `projectionEndpoint: true` MUST implement the projection endpoint for sealed Traces with authorization, mode-based access, and existence-hiding semantics identical to Trace retrieval, and MUST reject projection of unsealed Traces with `-32030`.

**AEP-REQ-OI-012**: OTLP/JSON responses MUST encode `trace_id` and `span_id` as lowercase hexadecimal strings per the OTLP specification's JSON mapping. Base64-encoded IDs (protobuf's default JSON mapping for `bytes`) are non-conforming.

**AEP-REQ-OI-013**: A server-side projection MUST equal the projection a conforming client would compute from the fetched Trace under the same L (AEP-REQ-OI-002 applies across the client/server boundary). A client reproducing a server's projection MUST take L from the server's `maxValueBytes` advertisement (absent → unbounded); computing under the unbounded default against a server that declares a limit fails this requirement by construction.

## Core hooks needed in the spec

Two additive changes to the core would remove the only approximations in the mapping:

1. **`ModelCallRecord.startedAt` / `completedAt`** (optional, RFC 3339) — v0.1 records only `latencyMs`, forcing the inferred sequential-placement rule above. Optional timestamps are additive under the compatibility policy (§14, AEP-REQ-105); Traces without them continue to project with `aep.timing.inferred = true`.
2. **`RedactionRecord.path` constrained to RFC 6901** — the schema currently describes `path` as "JSON pointer or trace path", a free-text union no consumer can match against reliably. The sibling Operation Context proposal already requires RFC 6901 with explicit `~0`/`~1` escaping for exactly this reason (AEP-REQ-OC-019: an unescaped emitter produces a *valid* pointer addressing a different location, so implementations disagree silently rather than erroring). The same tightening here would make every redaction record matchable and let the per-span `aep.redacted` flag cover the whole `redactions[]` array instead of its parseable subset.

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
        ├── LLM "claude-sonnet-5"                 span_id = H(...":turn:0:model:0")
        │     llm.model_name = "claude-sonnet-5"
        │     llm.token_count.prompt = 812 · completion = 96 · total = 908
        │     aep.model_call.id = "mc_1"
        │     aep.timing.inferred = true          (12:00:00.000 → 12:00:01.800)
        │
        └── TOOL "lookupOrder"                    span_id = H(...":turn:0:tool:0")
              tool.name = "lookupOrder"
              input.value  = {"orderId":"123"}    (application/json)
              output.value = {"status":"refunded"}
              aep.tool_call.id = "tc_1"
              aep.tool_call.sandbox_fidelity = "recorded"
```

Point an OTLP exporter at that span set and the session renders in Phoenix or Langfuse as an agent trace with no further work.

Note the inferred LLM span (12:00:00.000 → 12:00:01.800) overlaps the recorded tool span (12:00:01 → 12:00:02) by 800ms, so a viewer shows the model call running concurrently with the tool call it almost certainly caused. This is the residual-overlap artifact described in [Timing rules](#timing-rules) — flagged by `aep.timing.inferred`, and eliminated once `ModelCallRecord` carries real timestamps per the [core hook](#core-hooks-needed-in-the-spec).

## Conformance summary

An implementation producing projections (client- or server-side):

1. MUST project only sealed Traces, as a pure function of the Trace and the declared truncation limit (AEP-REQ-OI-001).
2. MUST produce byte-identical span trees for the same Trace and limit, across implementations and across the client/server boundary (AEP-REQ-OI-002, AEP-REQ-OI-013).
3. MUST emit the OpenInference vocabulary frozen at `openinference-semantic-conventions` 0.1.31, upgrading only via extension revision (AEP-REQ-OI-003).
4. MUST derive trace/span IDs from array positions per the derivation table, never generating or reusing external IDs (AEP-REQ-OI-004).
5. MUST keep span kinds fixed per the mapping tables (AEP-REQ-OI-005).
6. MUST NOT synthesize content or inferred facts — digest-only stays digest-only, and `llm.provider` is never derived from the model string (AEP-REQ-OI-006).
7. MUST propagate sensitivity labels and redaction markers, matching redaction paths only when RFC 6901-parseable, and MUST NOT project `injectedContext`, `stateSnapshot`, or `seed` (AEP-REQ-OI-007).
8. MUST follow the timing rules and flag every inferred start/end with `aep.timing.inferred` (AEP-REQ-OI-008).
9. MUST project §11.9 correlation blocks as links only when well-formed OpenTelemetry identifiers are present, falling back to attributes otherwise (AEP-REQ-OI-009).
10. MUST apply any truncation inside the projection with markers and full-value digests, never delegating it to exporters (AEP-REQ-OI-010).

A server advertising the extension additionally:

11. MUST publish `conventionsVersion` and `projectionEndpoint` (and `maxValueBytes` when limited) in `capabilities.extensionData` per §5.1 (AEP-REQ-OI-011).
12. MUST, when `projectionEndpoint: true`, serve sealed-Trace projections with authorization, mode-based access, and existence-hiding identical to Trace retrieval, rejecting unsealed Traces with `-32030` (AEP-REQ-OI-011) and encoding OTLP/JSON IDs as lowercase hex (AEP-REQ-OI-012).

## Non-goals

- **Not an exporter or transport spec.** How the projected spans reach a backend (OTLP/gRPC, OTLP/HTTP, in-process SDK) is out of scope; the projection defines the span tree, not its delivery.
- **No reverse mapping.** OpenInference span trees do not project back into Traces; the Trace remains the canonical record and the projection is derived, never authoritative.
- **No streaming projection.** The projection is defined only over sealed Traces. Projecting live from stream events (§11.5) is deferred — it would need rules for spans whose end is not yet known.
- **No backend query standardization.** Echoing §11.9's non-goal: clients query Phoenix/Langfuse/etc. with those products' own surfaces.
- **No evaluation-result projection.** `EvaluationResult` and `Finding` artifacts are not part of the Trace and are out of scope here; projecting scores as OpenInference span annotations is a plausible follow-up once this mapping has implementation experience.

## Open questions

1. **Composite-session graph conventions.** The mapping emits `graph.node.id` on composite-session turn spans, following OpenInference's agent-graph conventions as frozen at the pinned release. Those conventions are newer than the core span-kind vocabulary; if they shift upstream, the CSE-specific rows are the ones most likely to need attention in the next convention-revision cycle. Pinning contains the blast radius but doesn't remove it.
2. **Reasoning as events vs. spans.** Reasoning steps project as span events because they carry no duration. Implementations with rich, long-running reasoning phases may prefer spans; that would require optional start/end timestamps on `ReasoningStep` (an additive core hook this proposal does not yet request).
3. **Conventions-version pinning granularity.** This revision pins a whole `openinference-semantic-conventions` release. If upstream begins versioning its spec text independently of the package, or if per-namespace stability markers appear, pinning could become finer-grained than release-level. Until then, release-level is the only anchor upstream offers.
