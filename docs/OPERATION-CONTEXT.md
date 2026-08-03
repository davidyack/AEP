# Operation Context Resolution Extension

*Resolving externally observed production entity references into AEP-compatible injected context.*

**Extension ID:** `operation-context` (proposed) · **Spec section:** none yet — this is a draft proposal · **Status:** draft, revision 5

---

This document proposes an optional AEP extension that lets an authorized consumer resolve application-owned entity references — typically obtained from an external observability system — into context shaped for AEP's injection points, so the situation a production agent operation observed can be frozen into a replayable artifact before the application state changes or expires.

Requirement identifiers in this document use a provisional `AEP-REQ-OC-NNN` series. If the extension is adopted, they would be renumbered into the global `AEP-REQ-NNN` registry and indexed in [`CONFORMANCE.md`](./CONFORMANCE.md).

## Why this extension exists

AEP replay is self-contained by design. A Trace captures the full `injectedContext` snapshot (AEP-REQ-037), so an AEP-native session can be replayed without ever contacting the originating application again. That design decision — capture the snapshot, not the pointer — is documented in [`REPLAY.md`](./REPLAY.md) and this proposal does not revisit it.

But many production agent operations never execute through AEP. For those, the record of what happened lives in an external observability system (OpenTelemetry, Application Insights, Tempo, …), which typically identifies:

- which agent executed
- correlation identifiers
- business entity references
- entity versions
- entity digests

Those systems intentionally do not retain customer business content. Copying entity content into telemetry would duplicate customer data and PII into observability infrastructure, inflate telemetry volume, subject content to telemetry retention and sampling policies, and turn the telemetry backend into an accidental application-data store.

So a consumer who wants to turn a production observation into a replayable test case knows *that* the agent touched `task-42` at version 17 with a recorded digest — but not what `task-42` contained. To build the test case, the consumer must recover that content while the application still retains it. Today there is no standard mechanism: every dataset builder, audit tool, and debugging tool integrates against application-specific APIs or goes without.

### Relationship to Observability Correlation

This extension is the application-side complement to the Observability Correlation extension (spec §11.9). That extension answers "which externally observed operation corresponds to this AEP agent?" and explicitly declines to standardize any query API against the telemetry backend. This extension answers the next question: "what application context did that operation observe?" The two are complementary; neither depends on the other.

### Why standardize this in AEP

Without a standard, every application invents a private resolution endpoint and every consumer writes a per-application integration. Standardizing Operation Context Resolution lets any compliant dataset builder, replay engine, audit tool, or debugging tool resolve production context without application-specific integration. The extension standardizes resolution semantics, reconstruction fidelity, and security requirements, while leaving application storage and history mechanisms implementation-defined. Because the output is an `injectedContext` shaped against the target agent's declared injection points (spec §8), with completeness reported explicitly, resolved context flows directly into session creation without transformation — which is the specific value of putting this in AEP rather than leaving it as a generic snapshot API.

## What's out of scope

This extension does not define:

- replay execution or replay fidelity
- evaluation, grading, or reference answers
- fixtures or datasets
- production telemetry or trace storage
- immutable storage or historical archives

A consumer MAY use resolved context to create a fixture. Fixtures are consumer-owned artifacts and are outside this specification (see [`DATASET.md`](./DATASET.md) for the portable dataset shape a consumer might land in).

## Relationship to replay

This extension is intentionally not part of replay. Replay uses immutable injected context captured in the Trace; Operation Context Resolution exists only to obtain that context *before* it is frozen. Resolution is late-binding, but it happens exactly once — at artifact-creation time, not at replay time.

```
External observability system
        │  identifies operation + entity references
        ▼
aep.operationContext.resolve
        │  returns injectedContext + fidelity report
        ▼
Consumer masks and freezes the context
        │
        ▼
Immutable, consumer-owned artifact
        │
        ▼
AEP session creation with injectedContext
        (replay never contacts the resolver)
```

A replay system SHOULD NOT invoke the resolver during replay execution. This mirrors the core replay rule that replay MUST NOT fall through to live sources (`-32050 replay_integrity_violation`): a replay that re-resolves context against a live application is neither a replay nor a live run.

## Terminology

### EntityReference

A reference to application-owned business state:

```json
{
  "type": "task",
  "id": "42",
  "version": "17",
  "alias": "task-42-before",
  "digest": {
    "algorithm": "sha256",
    "canonicalization": "rfc8785",
    "schemaId": "com.example.task",
    "schemaVersion": "3",
    "value": "ab12cd34..."
  },
  "observedAt": "2026-08-02T16:30:00Z",
  "injectionPoint": "retrieval"
}
```

- `type` (string, required) — application-defined entity type
- `id` (string, required) — application-defined identifier
- `version` (string, optional) — the revision observed by the operation
- `alias` (string, optional) — consumer-chosen name for this reference, unique within the request. When supplied, it becomes the placement key for the resolved content (see "Context placement"), decoupling the consumer from the default key format.
- `digest` (object, optional) — content digest recorded at observation time; see "Digest shape" below
- `observedAt` (string, conditional) — RFC 3339 timestamp of observation. REQUIRED when both `version` and the request-level `asOf` are absent, so a reference is never structurally incapable of historical resolution without the consumer having chosen that. Precedence for historical resolution is `version`, then per-entity `observedAt`, then request-level `asOf`.
- `injectionPoint` (string, conditional) — the declared injection point the resolved entity content should be placed under in the returned `injectedContext`. REQUIRED on every reference when the request is agent-bound (AEP-REQ-OC-018). When omitted on an unbound request, the content is returned in the resolution report only and the consumer maps it itself.

EntityReference is defined by this extension. Reviewer positions on promoting it to the core specification are split (see Open Questions); it remains extension-local in this draft because promotion is a core-spec change outside an extension proposal's authority.

### Digest shape

```json
{
  "algorithm": "sha256",
  "canonicalization": "rfc8785",
  "schemaId": "com.example.task",
  "schemaVersion": "3",
  "value": "ab12cd34..."
}
```

- `algorithm` (required) — the hash function
- `canonicalization` (required) — identifier of the canonicalization scheme the digest was computed under (e.g. `rfc8785` for JCS, or a provider-defined reverse-DNS identifier). The hash function and the canonicalization are different fields because they are different facts: two systems that both say `sha256` can still disagree on the bytes.
- `schemaId` / `schemaVersion` (optional) — the entity schema the canonical representation was derived from, when the provider versions entity shapes
- `value` (required) — the digest value

**AEP-REQ-OC-016**: Every digest MUST carry a `canonicalization` identifier. A resolver MUST compute digest verification under the declared scheme. If the resolver does not support the declared scheme, it MUST report `digestVerification: "scheme_unsupported"` and MUST NOT report `current_digest_match` fidelity.

Canonicalization is required from v0.1 deliberately: `current_digest_match` is a cross-system comparison — the digest is recorded by one implementation and verified against content canonicalized by another. Retrofitting the field later would invalidate every digest recorded in the meantime.

### Operation context

The business context required to recreate the situation originally observed by an agent: business entities plus contextual state (user, tenant, environment), expressed using the target agent's declared AEP injection points (spec §8).

### Resolver

The capability, implemented by an application host and exposed through the AEP evaluation surface, that resolves EntityReferences into operation context.

## Extension advertisement

The extension uses the standard advertisement mechanism: `operation-context` appears in `capabilities.extensions`, with structured metadata in `capabilities.extensionData` per spec §5.1 (AEP-REQ-134 through AEP-REQ-140).

```json
{
  "capabilities": {
    "extensions": ["operation-context"],
    "extensionData": {
      "operation-context": {
        "version": "0.1.0",
        "payload": {
          "historicalResolution": true,
          "retention": {
            "mode": "bounded",
            "days": 30
          },
          "supportedEntityTypes": ["task", "document"],
          "supportedInjectionPoints": ["user", "tenant", "environment", "retrieval"],
          "canonicalizationSchemes": ["rfc8785", "com.example.task-canon-v3"],
          "sourceMasking": true,
          "maxEntitiesPerRequest": 50
        }
      }
    }
  }
}
```

Payload fields:

- `historicalResolution` (boolean, required) — whether the host may return the historical entity revision observed by the original operation. `false` means only current state is resolvable.
- `retention` (object, optional) — approximate window during which historical context is expected to remain resolvable. `mode` is one of `none`, `bounded`, `indefinite`, `unknown`; `days` applies to `bounded`. Advisory at the advertisement level; the response carries an actionable `retentionHorizon` (see Response).
- `supportedEntityTypes` (array, optional) — entity-reference types the resolver can resolve.
- `supportedInjectionPoints` (array, optional) — injection-point names the resolver may return.
- `canonicalizationSchemes` (array, optional) — digest canonicalization scheme identifiers the resolver can verify.
- `sourceMasking` (boolean, optional) — whether the resolver applies source-side masking or field filtering under application policy. Advertised so consumers can predict field omissions before resolving.
- `maxEntitiesPerRequest` (integer, optional) — maximum entity references accepted per request.

Advertisement exists so consumers can preflight: a consumer planning resolution (or scheduling when mining must run, given retention) reads both the target agent's contract and this payload before issuing any request.

Until the extension is registered, vendor implementations MUST use a reverse-DNS identifier (e.g. `com.example.operation-context`) per AEP-REQ-106.

## Protocol surface

Both AEP surfaces are exposed, consistent with the core's REST/JSON-RPC equivalence:

| JSON-RPC | REST |
|----------|------|
| `aep.operationContext.resolve` | `POST /aep/operation-context/resolve` |

## Request

```json
{
  "agent": {
    "agentId": "signal-synthesis",
    "version": "1.4.0"
  },
  "correlation": {
    "provider": "opentelemetry",
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "00f067aa0ba902b7"
  },
  "asOf": "2026-08-02T16:30:00Z",
  "entityReferences": [
    {
      "type": "task",
      "id": "42",
      "version": "17",
      "alias": "task-42",
      "digest": {
        "algorithm": "sha256",
        "canonicalization": "rfc8785",
        "value": "ab12cd34..."
      },
      "injectionPoint": "retrieval"
    }
  ],
  "requestedInjectionPoints": ["user", "tenant", "environment"]
}
```

- `agent` (object, optional) — the target agent and version the resolved context is intended for. When present, the resolver MUST validate `requestedInjectionPoints` and per-entity `injectionPoint` mappings against that agent contract version's declared `contextModel.injectionPoints`, and MUST reject an unknown `agentId` or unavailable version with `-32001 agent_not_found` (existence-hiding semantics per AEP-REQ-128) rather than silently validating against a different contract version. When absent, injection-point shaping is best-effort and compatibility with any particular agent is the consumer's responsibility. The field is optional rather than required so consumers with no replay target — audit and diagnostic tooling — can resolve entity content without naming an agent.
- `correlation` (object, optional) — external-observability correlation identifiers, in the same shape as the Observability Correlation extension (spec §11.9). Advisory: a resolver MAY use it to locate an operation-scoped snapshot or audit record, but resolution is driven by the supplied `entityReferences`.
- `asOf` (string, optional) — RFC 3339 timestamp of the operation. Resolvers with time-travel storage use it to resolve entities as they stood at that moment, independent of versions or digests being recorded. Precedence per entity is `version`, then per-entity `observedAt`, then `asOf`. Consumers SHOULD supply it whenever the observability system recorded an operation time.
- `entityReferences` (array, required) — the entities to resolve.
- `requestedInjectionPoints` (array, optional) — non-entity injection-point names the consumer wants returned. The resolver returns only requested, supported, and authorized points. Omitting the field requests no injection-point context (entities only).

**AEP-REQ-OC-018**: When the request includes `agent`, every entity reference MUST specify `injectionPoint`, and a resolver MUST reject an agent-bound request containing a reference without one (`-32012 input_schema_violation`). An agent-bound response returns an `injectedContext` intended for session creation, with completeness reported explicitly per AEP-REQ-OC-020. When the request omits `agent`, `injectionPoint` MAY be omitted; such a response is a diagnostic resolution result, and its `injectedContext` — containing only mapped content and requested points — is not complete and MUST NOT be treated as directly usable for session creation.

## Response

```json
{
  "resolvedAt": "2026-08-03T02:00:00Z",
  "sessionReady": true,
  "weakestFidelity": "historical",
  "injectedContext": {
    "user": {
      "id": "user-7",
      "role": "travel-coordinator"
    },
    "tenant": {
      "id": "tenant-3"
    },
    "environment": {
      "time": "2026-08-02T16:30:00Z"
    },
    "retrieval": {
      "task-42": {
        "title": "Book flights to Berlin",
        "status": "open"
      }
    }
  },
  "resolutionReport": {
    "entities": [
      {
        "reference": { "type": "task", "id": "42", "version": "17", "alias": "task-42" },
        "injectionPoint": "retrieval",
        "contextPath": "/retrieval/task-42",
        "status": "resolved",
        "fidelity": "historical",
        "digestVerification": "verified"
      }
    ],
    "injectionPoints": [
      { "name": "user", "status": "resolved" },
      { "name": "tenant", "status": "resolved" },
      { "name": "environment", "status": "resolved" }
    ],
    "retentionHorizon": "2026-07-04T00:00:00Z"
  },
  "dataHandling": {
    "maskingApplied": true,
    "restrictedFieldsOmitted": true,
    "evaluationUse": "same-tenant",
    "humanReviewAllowed": false,
    "policyRef": "capture-policy-42",
    "policyVersion": "7"
  },
  "warnings": []
}
```

- `resolvedAt` (RFC 3339, required) — when this resolution was performed. Frozen artifacts carry it as provenance, and time-relative fields in the response are interpreted against it.
- `sessionReady` (boolean, agent-bound responses only) — whether `injectedContext` is complete for session creation; see AEP-REQ-OC-020. Completeness only — it says nothing about fidelity.
- `weakestFidelity` (agent-bound responses only) — the weakest fidelity across all placed entities; the baseline-use counterpart to `sessionReady` (AEP-REQ-OC-020).
- `injectedContext` — the context object. On an agent-bound request with `sessionReady: true` it is complete and ready to supply verbatim at `POST /aep/sessions` / `aep.session.start`; on an unbound request it is a diagnostic partial (AEP-REQ-OC-018). It contains both the requested non-entity injection points and the resolved entity content placed under each entity's requested `injectionPoint` (see "Context placement"). Entity content belongs *inside* `injectedContext` because AEP agents consume context only through declared injection points (AEP-REQ-038); a response that returned entities outside it would omit the primary application state from the object actually passed to session creation.
- `resolutionReport` — the fidelity and status record: one entry per requested entity reference and one per requested injection point. This is the part a consumer freezes alongside the artifact as provenance.
- `resolutionReport.retentionHorizon` (RFC 3339, optional) — the earliest instant, as of `resolvedAt`, for which the resolver expects historical resolution to succeed. Actionable per-response: a batch consumer sweeping a week of operations can detect "I am resolving near the edge" without a second call, instead of discovering a silently degrading fidelity gradient after the fact. (The advertised `retention` is a duration rather than an instant for the complementary reason: extension metadata is fetched and cached, and a cached instant goes stale.)
- `dataHandling` — metadata describing the effective data-use policy and transformations applied (see Security). Field names are illustrative; the shape is provider-extensible. When resolved content spans sources with differing effective policies, the response-level object MUST describe the most restrictive policy that applies to any returned content; a resolver MAY additionally attach a per-entity `dataHandling` override to individual resolution entries where a less restrictive policy governs that entity.
- `warnings` — non-fatal notes, each with `code` (stable, machine-readable) and `message`.

**AEP-REQ-OC-020**: An agent-bound response MUST include `sessionReady`: `true` only when every requested entity reference reports `resolved` and was placed, and every requested injection point reports `resolved`; `false` otherwise. When `sessionReady` is `false`, consumers MUST NOT supply the response's `injectedContext` at session creation without accounting for the reported gaps. The field MUST be absent on unbound (diagnostic) responses. Rejecting a whole agent-bound request because some placement cannot be produced is deliberately *not* required: partial reconstruction stays first-class so consumers can classify near-miss operations — but the gap is machine-visible in one field rather than inferable only by cross-checking every report entry.

`sessionReady` reports completeness, not fidelity. A `sessionReady: true` response MAY consist entirely of `current_version_only` entities; consumers MUST still apply AEP-REQ-OC-015 before using the artifact as a baseline. So that the two signals sit adjacent rather than one being buried in the report, an agent-bound response MUST also include `weakestFidelity` — the weakest fidelity across all placed entities, ordered `historical` > `current_digest_match` > `current_version_only`. A consumer's go/no-go for session creation is `sessionReady`; its go/no-go for baseline use is `weakestFidelity` plus AEP-REQ-OC-015 — one field cannot answer both questions.

### Context placement

Entity content is placed in `injectedContext` under its target injection point, keyed by:

1. the reference's `alias`, when supplied;
2. otherwise `{type}:{id}@{version}`, when `version` was supplied;
3. otherwise `{type}:{id}`.

The separator is `:` rather than `/` deliberately: `/` is JSON Pointer's segment separator, and a default key containing it would make every `contextPath` depend on correct RFC 6901 escaping — a trap better removed than documented.

The version-qualified default exists because a single operation can legitimately touch the same entity at two revisions — the agent reads `task:42` at v17, then writes it, producing v18, and both matter to an evaluator (the read is the evidence, the write is the outcome). An unversioned key would silently overwrite one with the other while `resolutionReport.entities` continued to list both as resolved — a report corroborating a context that doesn't match it.

**AEP-REQ-OC-019**: Every resolved entry placed in `injectedContext` MUST carry `contextPath` — a JSON Pointer (RFC 6901) from the `injectedContext` root to the placed content — and placements MUST be unique. When forming `contextPath`, each key segment MUST be escaped per RFC 6901 (`~` as `~0`, `/` as `~1`). The default key format contains neither character, but aliases and entity identifiers are arbitrary strings the specification does not control: a reference with alias `tasks/before` targeting `retrieval` yields `"contextPath": "/retrieval/tasks~1before"`, and an emitter that skips the escape produces `/retrieval/tasks/before` — a *valid* pointer addressing a different, usually nonexistent location, so the two implementations disagree silently rather than erroring. A request whose references would produce colliding placements MUST be rejected whole with `-32012 input_schema_violation`, so a collision surfaces as an error rather than a silent drop. Uniqueness is over the full pointer: the same entity MAY legitimately be placed at multiple injection points through multiple references (an agent declaring both `retrieval` and a workspace point may want the same task in both); only identical placements collide. `contextPath` is the authoritative correlation between report and context: consumers MUST correlate by pointer rather than by reconstructing key conventions, which also keeps them independent of the injection point's value shape.

### Entity resolution entries

```json
{
  "reference": { "type": "task", "id": "42", "version": "17", "alias": "task-42" },
  "injectionPoint": "retrieval",
  "contextPath": "/retrieval/task-42",
  "status": "resolved",
  "fidelity": "historical",
  "digestVerification": "verified",
  "content": {}
}
```

`status` (required) is one of:

| Status | Meaning |
|--------|---------|
| `resolved` | Content was returned; `fidelity` states how faithful it is |
| `expired` | No content could be returned and the requested historical revision is outside the retention window |
| `version_mismatch` | The entity exists but no content matching the requested version or digest could be returned |
| `not_found` | The entity cannot be found |
| `unsupported` | The resolver does not support this entity type |
| `unauthorized` | The caller is not authorized for this entity, or the effective data-use policy prohibits returning it |

Composition rules:

- When `status` is not `resolved`: `content`, `fidelity`, and `contextPath` MUST be absent.
- When `status` is `resolved`: `fidelity` and `digestVerification` are REQUIRED. Content is placed in `injectedContext` under the requested `injectionPoint` with `contextPath` recording the placement; when the reference specified no `injectionPoint` (unbound requests only), the entry's `content` field carries it instead. Content MUST NOT appear in both places.

`digestVerification` (required when resolved) is one of:

| Value | Meaning |
|-------|---------|
| `verified` | A digest was supplied and the returned content matches under the declared canonicalization |
| `mismatch` | A digest was supplied and the returned content does not match — positive evidence the content differs from what the operation observed |
| `scheme_unsupported` | A digest was supplied but the resolver cannot verify under the declared canonicalization scheme — no evidence either way |
| `not_attempted` | No digest was supplied in the reference |

`mismatch` and `scheme_unsupported` are deliberately distinct values: the first is evidence of drift, the second says nothing about drift — the content may be byte-identical. Consumers MUST NOT treat `scheme_unsupported` as evidence of change.

Fidelity and verification interact per AEP-REQ-OC-010: `current_digest_match` requires `verified`; `historical` is prohibited with `mismatch` and permitted with `verified`, `scheme_unsupported`, or `not_attempted` — in the latter two cases it stands as an unverified resolver assertion, detectable from this field.

`fidelity` (required when resolved) is one of:

- `historical` — the requested historical revision was returned.
- `current_digest_match` — a historical revision was unavailable, but current content matches the digest recorded at observation time (`digestVerification` MUST be `verified`). The resolver has demonstrated the entity has not changed; consumers MAY treat this as equivalent to the observed content.
- `current_version_only` — only current content was available and the resolver cannot prove it matches what the operation observed. Consumers MUST NOT silently treat this as an exact reconstruction.

**AEP-REQ-OC-010**: `current_digest_match` MUST be accompanied by `digestVerification: "verified"` — it is definitionally a verification claim. `historical` MUST NOT be reported when `digestVerification` is `"mismatch"`: a mismatch against the recorded digest is contradictory evidence, and MUST surface as `version_mismatch`, or as `resolved` with `fidelity: "current_version_only"` and `digestVerification: "mismatch"` when returning the content is still useful diagnostically. `historical` with `"scheme_unsupported"` or `"not_attempted"` is permitted: it is a provenance claim grounded in the resolver's own version-addressed storage, unverified against the recorded digest, and the `digestVerification` value says exactly that. (Prohibiting it would force byte-identical content down to `current_version_only` — and into reduced-fidelity marking under AEP-REQ-OC-015 — merely because the digest's canonicalization scheme is foreign, which conflates "unverified" with "changed.") A resolver MUST NOT silently substitute newer content for the historical state observed by the original operation.

**AEP-REQ-OC-013**: When any returnable content exists for an entity, the resolver MUST report `resolved` with the appropriate (possibly degraded) fidelity. `expired` is reserved for the case where no content can be returned at all. This rule exists because "historical is outside retention but current content exists" would otherwise be reportable two defensible ways, and consumers branch differently on each.

**AEP-REQ-OC-014**: The response MUST contain exactly one `resolutionReport.entities` entry per requested reference, in request order. A resolver MUST NOT truncate the reference list; a request exceeding `maxEntitiesPerRequest` MUST be rejected whole with `-32021 limit_exceeded`. This makes completeness a checkable invariant rather than a consumer diligence obligation.

**AEP-REQ-OC-011**: Resolvers subject to existence-hiding policy MAY report `unauthorized` entities as `not_found`, consistent with the core's existence-hiding semantics (AEP-REQ-128). A resolver doing so MUST include a response-level warning that existence-hiding is in effect — so consumers know their `not_found` set is untrustworthy as a deletion signal rather than assuming it isn't — and the audit record (AEP-REQ-OC-006) MUST carry the true per-entity outcome. Consumers MUST NOT infer entity existence or absence from `not_found` when that warning is present.

Entity resolution entries reserve a `proof` field for a future signed-attestation capability (see Open Questions), so adding it later is a compatible change.

### Injection point resolution entries

Each requested non-entity injection point is reported individually:

```json
{ "name": "user", "status": "resolved" }
```

`status` is one of `resolved`, `unsupported`, `unauthorized`, `not_found`, `unavailable`. Unsupported and unauthorized are distinct outcomes with different remedies and MUST NOT be conflated where policy permits disclosure; a resolver applying existence-hiding MAY collapse `unauthorized` and `not_found` into `unavailable`, under the same warning and audit obligations as AEP-REQ-OC-011.

### Consumer obligations

**AEP-REQ-OC-015**: A consumer producing a replay artifact from a resolution containing any `current_version_only` entity MUST mark the artifact as reduced-fidelity, and MUST NOT use it as a scoring or regression baseline without explicit operator acknowledgement. This failure mode is silent otherwise: a gold case built on drifted content grades the agent against a situation that never existed, and presents as a legitimate regression. The acknowledgement mechanism is consumer-defined; a conforming example is allowing reduced-fidelity artifacts to be promoted and run while excluding them from gate-blocking runs until a reviewer records an acknowledgement, stored with attribution on the case.

**AEP-REQ-OC-017**: A consumer that persists resolved content MUST persist the accompanying `dataHandling` metadata with it and MUST propagate it to any derived artifact. An artifact frozen without its policy provenance carries content whose constraints — human-review prohibitions, same-tenant evaluation limits, retention bounds — have become invisible to every downstream system.

## Digest validation

When a digest is supplied in an EntityReference, the resolver SHOULD verify returned content against it under the declared canonicalization scheme and MUST report the result in `digestVerification` per the composition rules above.

This requirement chain exists for one reason: to prevent replay systems from unknowingly testing against modified production data. Requiring the canonicalization identifier keeps `current_digest_match` a verifiable claim rather than an assertion the consumer must take on trust.

## Injection point integration

`injectedContext` is keyed by injection-point names and returned complete, for direct use at session creation subject to the target agent's declared `contextModel.injectionPoints` schemas (AEP-REQ-034 through AEP-REQ-036).

This extension introduces no new reserved injection-point types. The core's reserved types (spec §8.1) cover the common cases:

- `user` — the acting user profile
- `tenant` — tenant or workspace configuration
- `environment` — locale, timezone, feature flags; the observed clock belongs here (e.g. `environment.time`)
- `retrieval` — simulated retrieval results; the natural home for resolved entity content an agent originally obtained by lookup

Domain-specific state — approval status, workflow stage, and similar — belongs in entity content, or in an application-declared custom injection point using a reverse-DNS name per §8.1. If a new reserved type proves broadly necessary, that is a separate core proposal; this extension does not introduce one.

**AEP-REQ-OC-012**: When the request names an `agent`, the resolver MUST validate requested injection points and entity mappings against that agent contract version and MUST return `injectedContext` values that validate against the corresponding declared schemas. When no `agent` is supplied, resolvers SHOULD still shape values against the reserved-type conventions, and compatibility with a specific agent is the consumer's responsibility.

The agent's declared injection-point schema always governs the value shape; the keyed map of "Context placement" is the default shape, not a normative constraint on the contract. When the declared schema requires a different shape — an agent declaring `retrieval` as an array of documents, say — the resolver places content per the declared schema and each entry's `contextPath` records where it landed (a JSON Pointer addresses array elements as readily as map keys). If the resolver cannot produce content conforming to the declared schema for the targeted point, it MUST report that entity's `status` as `unsupported` rather than emit non-conforming context; a resolver and an agent are never both conformant yet silently incompatible.

## Security

This extension introduces a controlled read path from authorized evaluation-surface consumers to production business data. That is a deliberate and bounded expansion of what the evaluation surface serves, and it must not become a production ingress. Security requirements are therefore normative.

### Deployment model

Operation Context Resolution is an evaluation-surface capability. It is exposed only through the AEP evaluation surface, to evaluator-audience credentials, under the core deployment invariants: unreachable from public production traffic paths (AEP-REQ-086), logically separated from user-facing production APIs (AEP-REQ-129), and audience-separated credentials (AEP-REQ-130 — production user credentials MUST NOT authorize the resolver).

An implementation MAY internally access production application data in order to resolve context — an event store, an audit log, the live application database. That access is implementation-internal. The extension does not expose a general-purpose production data API, does not add a production ingress, and does not make the evaluation surface reachable from production execution paths. The resolver reads *from* production; nothing in production calls *into* the resolver.

### Requirements

**AEP-REQ-OC-001**: The resolver MUST authenticate the caller.

**AEP-REQ-OC-002**: The resolver MUST authorize access at both the tenant and entity level. Tenant boundaries follow the core rule (AEP-REQ-089): a caller scoped to tenant A MUST NOT resolve entities from tenant B.

**AEP-REQ-OC-003**: Possession of an entity reference, digest, correlation identifier, trace identifier, or version MUST NOT confer authorization. References are routinely present in telemetry visible to parties who must not see entity content.

**AEP-REQ-OC-004**: The resolver MUST return only requested, supported, and authorized injection points, reporting each requested point's outcome per the injection-point resolution entries above.

**AEP-REQ-OC-005**: The resolver MUST minimize returned data to the requested context. It MUST NOT return unrelated tenant or application data.

**AEP-REQ-OC-006**: Successful and failed resolution requests MUST be auditable, with caller attribution, consistent with the core audit requirements (AEP-REQ-090, AEP-REQ-098). Where existence-hiding collapses a response status, the audit record MUST carry the true outcome.

**AEP-REQ-OC-007**: The resolver MUST apply the effective data-use policy of the owning application and tenant — including required source-side masking and field filtering — before content leaves the application boundary. Evaluator credentials do not bypass tenant data-handling settings.

**AEP-REQ-OC-008**: The resolver MUST NOT return content whose evaluation use or export the effective policy prohibits. Policy denial is reported as `unauthorized` (subject to existence-hiding).

**AEP-REQ-OC-009**: When masking or filtering was applied, or the effective policy constrains downstream use (retention limits, human-review prohibitions, same-tenant-only evaluation), the response MUST include `dataHandling` metadata describing the effective policy and transformations, so downstream systems can honor constraints they cannot otherwise see.

## Privacy

The resolver SHOULD return only the business context required to reconstruct the requested situation. Applications MAY omit fields the caller is not authorized to access; omissions SHOULD be detectable (via `dataHandling`, `warnings`, or per-point statuses) rather than silent where policy permits.

Consumers remain responsible for masking, retention, storage, and governance of resolved content after receipt, within the constraints declared in `dataHandling`. Resolved content is production data until the consumer's own controls say otherwise.

## Conformance summary

An implementation advertising `operation-context`:

1. MUST advertise the extension through `capabilities.extensions`, with metadata in `capabilities.extensionData` per §5.1.
2. MUST expose equivalent JSON-RPC and REST surfaces.
3. MUST authenticate callers and authorize resolution at tenant and entity level (AEP-REQ-OC-001 through 003).
4. MUST apply effective data-use policy before returning content and describe applied transformations (AEP-REQ-OC-007 through 009).
5. MUST distinguish `historical`, `current_digest_match`, and `current_version_only` fidelity, and MUST NOT silently substitute modified content (AEP-REQ-OC-010).
6. MUST verify digests under the declared canonicalization scheme, reporting the four-state `digestVerification` outcome — never conflating "mismatched" with "could not verify" (AEP-REQ-OC-016).
7. MUST report exactly one resolution entry per requested reference, with no truncation (AEP-REQ-OC-014), and prefer degraded-fidelity `resolved` over `expired` whenever content exists (AEP-REQ-OC-013).
8. MUST validate against the named agent contract version when the request supplies one, require per-entity injection-point mappings on agent-bound requests, and report session-readiness explicitly (AEP-REQ-OC-012, AEP-REQ-OC-018, AEP-REQ-OC-020).
9. MUST record every placement with a unique `contextPath` and reject colliding placements (AEP-REQ-OC-019).
10. MUST use the standard AEP error registry for request-level failures.
11. MAY support historical resolution, partial reconstruction, source-side masking, and existence-hiding (with the warning and audit obligations of AEP-REQ-OC-011).

Consumers producing replay artifacts are bound by AEP-REQ-OC-015 (reduced-fidelity marking and acknowledgement) and AEP-REQ-OC-017 (dataHandling persistence and propagation).

## Open questions

1. Should `EntityReference` be promoted to the core specification? Reviewer positions are split: one implementation reports two internal consumers that need exactly this shape today; another argues promotion creates a core compatibility surface before a second *external* consumer exists. This draft keeps it extension-local — promotion is a core-spec change that should ride its own proposal — and records the demand signal here.
2. Should canonicalization scheme identifiers be drawn from a registry (as `rfc8785` suggests) or remain fully provider-defined reverse-DNS names? v0.1 requires the identifier but does not enumerate schemes.
3. Signed resolution proofs — cryptographic attestation that returned content corresponds to the requested revision and digest. Agreed to be a later-version concern; the `proof` field on entity resolution entries is reserved so adding it is non-breaking. Declared canonicalization plus the audit trail covers the realistic v0.1 threat.
