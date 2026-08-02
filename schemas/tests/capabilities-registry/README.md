# CapabilitiesRegistry fixtures — `extensionData`

Positive and negative fixtures for the `extensionData` property of
[`capabilities-registry.schema.json`](../../capabilities-registry.schema.json) (§5.1 Capability Extension Data).

Naming convention:

- `valid-*.json` — MUST pass schema validation.
- `invalid-*.json` — MUST fail schema validation.
- `conformance-invalid-*.json` — passes portable schema validation but violates
  a normative contract-validation requirement that portable JSON Schema 2020-12
  does not enforce by default. A conformance suite, not the portable schema,
  rejects these:
  - `conformance-invalid-unadvertised-extension.json` violates AEP-REQ-134
    (every `extensionData` key must also appear in `extensions`), which is not
    expressible as a portable cross-field schema constraint.
  - `conformance-invalid-malformed-schemaref.json` violates AEP-REQ-135's
    requirement that `schemaRef` be a valid URI reference. The schema declares
    `format: "uri-reference"`, but Draft 2020-12 treats `format` as
    annotation-only by default, so AEP contract validation MUST check it —
    either via a validator run in format-assertion mode or via an explicit
    URI-reference check.

Validate with any Draft 2020-12 validator, e.g.:

```bash
npx ajv validate --spec=draft2020 --validate-formats=false \
  -s schemas/capabilities-registry.schema.json \
  -d "schemas/tests/capabilities-registry/valid-*.json"
```

Note: `format` (used by `schemaRef`'s `uri-reference`) is annotation-only by
default in Draft 2020-12; enforcing it requires a validator run in
format-assertion mode.
