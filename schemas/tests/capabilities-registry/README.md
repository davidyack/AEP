# CapabilitiesRegistry fixtures — `extensionData`

Positive and negative fixtures for the `extensionData` property of
[`capabilities-registry.schema.json`](../../capabilities-registry.schema.json) (§5.1 Capability Extension Data).

Naming convention:

- `valid-*.json` — MUST pass schema validation.
- `invalid-*.json` — MUST fail schema validation.
- `conformance-invalid-*.json` — passes schema validation but violates a
  normative contract-validation requirement that portable JSON Schema 2020-12
  cannot express. `conformance-invalid-unadvertised-extension.json` violates
  AEP-REQ-134 (every `extensionData` key must also appear in `extensions`);
  a conformance suite, not the schema, rejects it.

Validate with any Draft 2020-12 validator, e.g.:

```bash
npx ajv validate --spec=draft2020 --validate-formats=false \
  -s schemas/capabilities-registry.schema.json \
  -d "schemas/tests/capabilities-registry/valid-*.json"
```

Note: `format` (used by `schemaRef`'s `uri-reference`) is annotation-only by
default in Draft 2020-12; enforcing it requires a validator run in
format-assertion mode.
