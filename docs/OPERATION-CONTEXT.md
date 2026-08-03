# Operation Context Resolution Extension

*Resolving externally observed production entity references into AEP-compatible injected context.*

**Extension ID:** `operation-context` (proposed) · **Spec section:** none yet — this is a draft proposal · **Status:** draft

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

Without a standard, every application invents a private resolution endpoint and every consumer writes a per-application integration. Standardizing Operation Context Resolution lets any compliant dataset builder, replay engine, audit tool, or debugging tool resolve production context without application-specific integration. The extension standardizes resolution semantics, reconstruction fidelity, and security requirements, while leaving application storage and history mechanisms implementation-defined. Because the output is shaped for AEP injection points (spec §8), resolved context flows directly into session creation without transformation — which is the specific value of putting this in AEP rather than leaving it as a generic snapshot API.

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
        │  returns application context + fidelity report
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
  "digest": {
    "algorithm": "sha256",
    "value": "ab12cd34..."
  },
  "observedAt": "2026-08-02T16:30:00Z"
}
```

- `type` (string, required) — application-defined entity type
- `id` (string, required) — application-defined identifier
- `version` (string, optional) — the revision observed by the operation
- `digest` (object, optional) — content digest recorded at observation time; `algorithm` and `value` both required when present
- `observedAt` (string, optional) — RFC 3339 timestamp of observation; advisory. When both `version` and `observedAt` are supplied and conflict, `version` takes precedence.

EntityReference is defined by this extension. If other extensions later need the same shape, promoting it to the core specification is a separate proposal (see Open Questions).

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
          "supportedInjectionPoints": ["user", "tenant", "environment"],
          "maxEntitiesPerRequest": 50
        }
      }
    }
  }
}
```

Payload fields:

- `historicalResolution` (boolean, required) — whether the host may return the historical entity revision observed by the original operation. `false` means only current state is resolvable.
- `retention` (object, optional) — approximate window during which historical context is expected to remain resolvable. `mode` is one of `none`, `bounded`, `indefinite`, `unknown`; `days` applies to `bounded`. Informational: it does not guarantee successful resolution for any specific entity.
- `supportedEntityTypes` (array, optional) — entity-reference types the resolver can resolve.
- `supportedInjectionPoints` (array, optional) — injection-point names the resolver may return.
- `maxEntitiesPerRequest` (integer, optional) — maximum entity references accepted per request. Requests exceeding it fail with `-32021 limit_exceeded`.

Until the extension is registered, vendor implementations MUST use a reverse-DNS identifier (e.g. `com.example.operation-context`) per AEP-REQ-106.

## Protocol surface

Both AEP surfaces are exposed, consistent with the core's REST/JSON-RPC equivalence:

| JSON-RPC | REST |
|----------|------|
| `aep.operationContext.resolve` | `POST /aep/operation-context/resolve` |

## Request

```json
{
  "correlation": {
    "provider": "opentelemetry",
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "00f067aa0ba902b7"
  },
  "entityReferences": [
    {
      "type": "task",
      "id": "42",
      "version": "17",
      "digest": {
        "algorithm": "sha256",
        "value": "ab12cd34..."
      }
    }
  ],
  "requestedInjectionPoints": ["user", "tenant", "environment"]
}
```

- `correlation` (object, optional) — external-observability correlation identifiers, in the same shape as the Observability Correlation extension (spec §11.9). Advisory: a resolver MAY use it to locate an operation-scoped snapshot or audit record, but resolution is driven by the supplied `entityReferences`.
- `entityReferences` (array, required) — the entities to resolve.
- `requestedInjectionPoints` (array, optional) — injection-point names the consumer wants returned. The resolver returns only requested, supported, and authorized points. Omitting the field requests no injection-point context (entities only).

## Response

```json
{
  "resolvedContext": {
    "user": {
      "id": "user-7",
      "role": "travel-coordinator"
    },
    "tenant": {
      "id": "tenant-3"
    },
    "environment": {
      "time": "2026-08-02T16:30:00Z"
    }
  },
  "unsupportedInjectionPoints": [],
  "entityResolutions": [
    {
      "reference": {
        "type": "task",
        "id": "42",
        "version": "17"
      },
      "status": "resolved",
      "fidelity": "historical",
      "digestVerified": true,
      "content": {
        "title": "Book flights to Berlin",
        "status": "open"
      }
    }
  ],
  "warnings": []
}
```

- `resolvedContext` — context keyed by injection-point name, shaped so it can be supplied directly as `injectedContext` at `POST /aep/sessions` / `aep.session.start` without transformation. See "Injection point integration" below.
- `unsupportedInjectionPoints` — requested injection points the resolver does not support or the caller is not authorized to receive. Explicit, so consumers can classify reconstructions as complete or partial.
- `entityResolutions` — one entry per requested EntityReference, in request order.
- `warnings` — non-fatal notes (e.g. an entity near its retention boundary), each with `code` and `message`.

### EntityResolution

```json
{
  "reference": { "type": "task", "id": "42", "version": "17" },
  "status": "resolved",
  "fidelity": "historical",
  "digestVerified": true,
  "content": {}
}
```

`status` (required) is one of:

| Status | Meaning |
|--------|---------|
| `resolved` | Content was returned; `fidelity` states how faithful it is |
| `expired` | Historical resolution was supported but the revision is outside the retention window |
| `version_mismatch` | The entity exists but no content matching the requested version or digest could be returned |
| `not_found` | The entity cannot be found |
| `unsupported` | The resolver does not support this entity type |
| `unauthorized` | The caller is not authorized for this entity |

Composition rules:

- When `status` is not `resolved`: `content` and `fidelity` MUST be absent.
- When `status` is `resolved`: `fidelity` and `content` are REQUIRED; `digestVerified` is present when a digest was supplied in the reference.

`fidelity` (required when resolved) is one of:

- `historical` — the requested historical revision was returned.
- `current_digest_match` — a historical revision was unavailable, but current content matches the digest recorded at observation time. The resolver has demonstrated the entity has not changed; consumers MAY treat this as equivalent to the observed content.
- `current_version_only` — only current content was available and the resolver cannot prove it matches what the operation observed. Consumers MUST NOT silently treat this as an exact reconstruction.

**AEP-REQ-OC-010**: A resolver MUST NOT report `historical` or `current_digest_match` unless the returned content matches the supplied digest (when one was supplied). A digest mismatch MUST be reported as `version_mismatch`, or as `resolved` with `fidelity: "current_version_only"` and `digestVerified: false` when returning current content is still useful diagnostically. A resolver MUST NOT silently substitute newer content for the historical state observed by the original operation.

**AEP-REQ-OC-011**: Resolvers subject to existence-hiding policy MAY report `unauthorized` entities as `not_found`, consistent with the core's existence-hiding semantics (AEP-REQ-128). Consumers MUST NOT infer entity existence from `not_found`.

Per-entity conditions are reported in `entityResolutions`; partial reconstruction is a first-class outcome, not an error. Request-level failures (authentication, malformed request, rate limits, limits exceeded) use the standard AEP error registry (spec §12). The extension defines no new top-level error model and no new error codes.

## Digest validation

When a digest is supplied in an EntityReference, the resolver SHOULD verify returned content against it and report the result in `digestVerified`. Digests SHOULD be computed over a canonical representation defined by the entity provider; the canonicalization scheme is identified by `digest.algorithm` conventions the provider documents (see Open Questions).

This requirement chain exists for one reason: to prevent replay systems from unknowingly testing against modified production data.

## Injection point integration

`resolvedContext` is keyed by injection-point names and shaped for direct use as `injectedContext` at session creation, subject to the target agent's declared `contextModel.injectionPoints` schemas (AEP-REQ-034 through AEP-REQ-036).

This extension introduces no new reserved injection-point types. The core's reserved types (spec §8.1) cover the common cases:

- `user` — the acting user profile
- `tenant` — tenant or workspace configuration
- `environment` — locale, timezone, feature flags; the observed clock belongs here (e.g. `environment.time`)

Domain-specific state — approval status, workflow stage, and similar — belongs in entity content, or in an application-declared custom injection point using a reverse-DNS name per §8.1. If a new reserved type proves broadly necessary, that is a separate core proposal; this extension does not introduce one.

**AEP-REQ-OC-012**: Resolvers SHOULD return `resolvedContext` values that validate against the corresponding injection-point schemas declared in the relevant Agent Contract, so consumers can pass resolved context to session creation without transformation.

## Security

This extension introduces a controlled read path from authorized evaluation-surface consumers to production business data. That is a deliberate and bounded expansion of what the evaluation surface serves, and it must not become a production ingress. Security requirements are therefore normative.

### Deployment model

Operation Context Resolution is an evaluation-surface capability. It is exposed only through the AEP evaluation surface, to evaluator-audience credentials, under the core deployment invariants: unreachable from public production traffic paths (AEP-REQ-086), logically separated from user-facing production APIs (AEP-REQ-129), and audience-separated credentials (AEP-REQ-130 — production user credentials MUST NOT authorize the resolver).

An implementation MAY internally access production application data in order to resolve context — an event store, an audit log, the live application database. That access is implementation-internal. The extension does not expose a general-purpose production data API, does not add a production ingress, and does not make the evaluation surface reachable from production execution paths. The resolver reads *from* production; nothing in production calls *into* the resolver.

### Requirements

**AEP-REQ-OC-001**: The resolver MUST authenticate the caller.

**AEP-REQ-OC-002**: The resolver MUST authorize access at both the tenant and entity level. Tenant boundaries follow the core rule (AEP-REQ-089): a caller scoped to tenant A MUST NOT resolve entities from tenant B.

**AEP-REQ-OC-003**: Possession of an entity reference, digest, correlation identifier, trace identifier, or version MUST NOT confer authorization. References are routinely present in telemetry visible to parties who must not see entity content.

**AEP-REQ-OC-004**: The resolver MUST return only requested, supported, and authorized injection points.

**AEP-REQ-OC-005**: The resolver MUST minimize returned data to the requested context. It MUST NOT return unrelated tenant or application data.

**AEP-REQ-OC-006**: Successful and failed resolution requests MUST be auditable, with caller attribution, consistent with the core audit requirements (AEP-REQ-090, AEP-REQ-098).

**AEP-REQ-OC-007**: Resolvers SHOULD support field-level filtering and source-side masking where required by application policy.

## Privacy

The resolver SHOULD return only the business context required to reconstruct the requested situation. Applications MAY omit fields the caller is not authorized to access; omissions SHOULD be detectable (e.g. via `warnings` or `unsupportedInjectionPoints`) rather than silent where policy permits.

Consumers remain responsible for masking, retention, storage, and governance of resolved content after receipt. Resolved content is production data until the consumer's own controls say otherwise.

## Conformance summary

An implementation advertising `operation-context`:

1. MUST advertise the extension through `capabilities.extensions`, with metadata in `capabilities.extensionData` per §5.1.
2. MUST expose equivalent JSON-RPC and REST surfaces.
3. MUST authenticate callers and authorize resolution at tenant and entity level (AEP-REQ-OC-001 through 003).
4. MUST distinguish `historical`, `current_digest_match`, and `current_version_only` fidelity, and MUST NOT silently substitute modified content (AEP-REQ-OC-010).
5. MUST report per-entity conditions explicitly through `EntityResolution.status`.
6. MUST use the standard AEP error registry for request-level failures.
7. SHOULD verify supplied digests and report `digestVerified`.
8. SHOULD shape `resolvedContext` against declared injection points (AEP-REQ-OC-012).
9. MAY support historical resolution, partial reconstruction, and source-side masking.

## Open questions

1. Should `EntityReference` be promoted to the core specification for reuse by other extensions, or remain extension-local until a second consumer exists?
2. Should digest canonicalization be standardized per entity type (provider-declared scheme identifiers traveling with the digest), or left fully implementation-defined in v0.1?
3. Should resolvers be able to cryptographically attest that returned content corresponds to the requested revision and digest (signed resolution proofs)? Likely a later-version concern.
4. Should source-side masking capability be advertised in the extension payload so consumers can predict field omissions before resolving?
