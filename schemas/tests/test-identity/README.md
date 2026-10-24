# TestIdentity fixtures

Positive and negative fixtures for
[`test-identity.schema.json`](../../test-identity.schema.json) (spec §8.2 Test identities,
§7.1 Test identity discovery).

Naming convention (as for [`../capabilities-registry/`](../capabilities-registry/)):

- `valid-*.json` — MUST pass schema validation.
- `invalid-*.json` — MUST fail schema validation.
- `conformance-invalid-*.json` — passes portable schema validation but violates a
  normative requirement that JSON Schema cannot express. A conformance suite, not
  the schema, rejects these:
  - `conformance-invalid-attributes-carry-secret.json` violates AEP-REQ-145:
    `attributes` is an open object, so a token placed there validates
    structurally. The schema closes every *named* slot a credential could occupy
    (`additionalProperties: false`), but cannot judge the content of a free-form
    attribute; contract validation MUST.
  - `conformance-invalid-binding-snapshot-wrong-agent.json` is an
    `IdentityBinding` snapshot (the `$defs/IdentityBinding` shape recorded on
    `EvaluationSession.identity` / `Trace.identity`) that is structurally valid;
    whether the bound identity's `agentIds` covered the session's agent is a
    server-side check at bind time (AEP-REQ-148 b), not a schema property.

Validate with any Draft 2020-12 validator, e.g.:

```bash
npx ajv validate --spec=draft2020 --validate-formats=false \
  -s schemas/test-identity.schema.json \
  -d "schemas/tests/test-identity/valid-*.json"
```
