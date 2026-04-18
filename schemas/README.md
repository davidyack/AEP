# AEP Schemas

Canonical JSON Schema (Draft 2020-12) definitions for AEP data models. These are normative: AEP-compliant implementations MUST accept and produce values conforming to these schemas.

## Core contracts

| Schema | Purpose |
|--------|---------|
| [`agent-contract.schema.json`](./agent-contract.schema.json) | **Required.** The strict contract every AEP-compliant agent publishes. Includes identity, input/output schemas, context model, capabilities, determinism, endpoints, limits, traceability. |
| [`agent-summary.schema.json`](./agent-summary.schema.json) | **Required.** Compact summary returned by `GET /aep/agents` list endpoint. Includes name, risk, lifecycle, supported and authorized modes. |
| [`agent-descriptor.schema.json`](./agent-descriptor.schema.json) | Identity, risk tier, lifecycle, ownership. A subcomponent of the Agent Contract. |
| [`capabilities-registry.schema.json`](./capabilities-registry.schema.json) | Declared domains, actions, modalities, quality claims, out-of-scope statements. |
| [`context-bundle.schema.json`](./context-bundle.schema.json) | Evaluator-oriented curation: purpose, example tasks, boundaries, policy summary, tool descriptors. |

## Tools and policy (optional extensions)

These schemas support [optional extensions](../spec/AEP-0.1.md#11-optional-extensions) of the protocol. Implementations are not required to support them; those that do MUST follow the schemas below.

| Schema | Purpose |
|--------|---------|
| [`tool-capability.schema.json`](./tool-capability.schema.json) | **Optional.** Structured per-tool attributes: reversibility, observability, blast radius, sandbox fidelity. |
| [`policy-profile.schema.json`](./policy-profile.schema.json) | **Optional.** Structured policy rules with conditions, actions, exceptions, severity. |
| [`stream-event.schema.json`](./stream-event.schema.json) | **Optional.** Envelope and registered event types for the SSE streaming extension (§11.5). |
| [`dataset.schema.json`](./dataset.schema.json) | **Optional.** Portable dataset format for agent evaluation (§11.6 Dataset extension). |

## Evaluation

| Schema | Purpose |
|--------|---------|
| [`scenario.schema.json`](./scenario.schema.json) | Executable test cases with branching, adaptive inputs, assertions, and failure conditions. |
| [`evaluation-session.schema.json`](./evaluation-session.schema.json) | Session state, lifecycle record, injected context, scenarios, and trace references. |
| [`evaluation-result.schema.json`](./evaluation-result.schema.json) | Canonical output with named scoring dimensions, typed violations, and per-scenario results. |

## Artifacts

| Schema | Purpose |
|--------|---------|
| [`trace.schema.json`](./trace.schema.json) | **Required.** Complete run record with reasoning steps, tool calls, policy events, state snapshots, and replay metadata. |
| [`finding.schema.json`](./finding.schema.json) | Structured observations with typed evidence pointers. |

## Validation

All schemas are Draft 2020-12. Validate with any compliant validator:

```bash
# ajv-cli example
npx ajv validate -s schemas/agent-contract.schema.json -d your-agent-contract.json --spec=draft2020
```

## Stability

These are canonical, not illustrative. Schemas follow the protocol version. Breaking changes to schemas increment the protocol major version. Additive changes (new optional fields) do not.

## Extension

Implementations may add fields using reverse-DNS prefixes (e.g. `com.example.driftScore`). Schemas with `"additionalProperties": false` reject these by default; extend via a wrapper field (`"vendorExtensions": {...}`) or subclass the schema.
