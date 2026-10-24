# TestIdentity fixtures

Positive and negative fixtures for
[`test-identity.schema.json`](../../test-identity.schema.json) (spec §8.2 Test identities,
§7.1 Test identity discovery).

Naming convention (as for [`../capabilities-registry/`](../capabilities-registry/)):

- `valid-*.json` — MUST pass schema validation.
- `invalid-*.json` — MUST fail schema validation. Two of them target the
  `$defs/IdentityBinding` subschema rather than the document root:
  `invalid-binding-missing-authorization-ref.json`. The rest fail at the root:
  a credential-shaped top-level field, a credential-like `attributes` property
  name (`sessionCookie`; the `propertyNames` deny pattern), empty
  `applicability`, a fixture without `resetPolicy`, an unknown or `replay`
  mode in `authorizedModes`, a non-reverse-DNS custom kind, `availableAt` on
  an `available` identity, and a truncated digest.
- `*-identity-model-*.json` — fixtures for `$defs/IdentityModel`, the agent-side
  declaration shared by `AgentContract.contextModel.identity` and
  `AgentSummary.identityBinding`. `valid-identity-model-permode-split.json` is
  the "synthetic below, live principal in sandboxed-live" agent the walkthrough's
  sidebar describes. The invalid ones put `replay`, an unknown mode or an empty
  object under `perMode`, or a value outside `none | optional | required`.
- `conformance-invalid-*.json` — passes portable schema validation but violates a
  normative requirement that JSON Schema cannot express. A conformance suite, not
  the schema, rejects these:
  - `conformance-invalid-attributes-carry-secret.json` violates AEP-REQ-145:
    the property name (`note`) is innocuous, so the deny pattern cannot help, and
    only the value is a credential. Content is contract validation's job.
  - `conformance-invalid-duplicate-agent-entries.json` has two `applicability`
    entries for the same agent (AEP-REQ-141 requires distinct `agentId`s);
    cross-item uniqueness on one property is not portable schema.
  - `conformance-invalid-none-reset-in-sandboxed-live.json` lists
    `sandboxed-live` for an identity whose fixture has `resetPolicy: none`
    (AEP-REQ-150).
  - `conformance-invalid-contract-permode-outside-evaluation-modes.json` is a
    full `AgentContract` (validate it against `agent-contract.schema.json` with
    `agent-descriptor`, `capabilities-registry` and `test-identity` as references)
    whose `contextModel.identity.perMode` names `sandboxed-live` while
    `evaluationModes` does not include it. AEP-REQ-146 makes that a startup
    contract-validation error; the subset check is not portable schema.
  - `conformance-invalid-session-identity-not-applicable.json` is an
    `EvaluationSession` document (validate it against
    `evaluation-session.schema.json` with `test-identity.schema.json` as a
    reference) whose bound identity, `eval.member-standard`, is not applicable to the
    session's agent `example.release-notes` per the identity fixtures here.
    Applicability is a cross-document check at bind time (AEP-REQ-148 c).

Validate with any Draft 2020-12 validator, e.g.:

```bash
npx ajv validate --spec=draft2020 --validate-formats=false \
  -s schemas/test-identity.schema.json \
  -d "schemas/tests/test-identity/valid-*.json"

npx ajv validate --spec=draft2020 --validate-formats=false \
  -s schemas/evaluation-session.schema.json -r schemas/test-identity.schema.json \
  -d schemas/tests/test-identity/conformance-invalid-session-identity-not-applicable.json
```
