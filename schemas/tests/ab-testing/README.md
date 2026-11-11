# AbTesting fixtures

Positive and negative fixtures for
[`ab-testing.schema.json`](../../ab-testing.schema.json) (spec §11.10 Blind A/B
Model Comparison).

Naming convention (as for [`../capabilities-registry/`](../capabilities-registry/)):

- `valid-*.json` — MUST pass schema validation. `valid-binding.json` targets
  `$defs/VariantBinding`; the others target the document root (the
  `extensionData["ab-testing"].payload`).
- `invalid-*.json` — MUST fail schema validation. `invalid-binding-*.json`
  target `$defs/VariantBinding`; the rest fail at the root: a single variant,
  duplicate labels, descriptive (`fast`, `smart`) or lowercase labels,
  per-variant descriptions, an `assignments` field that would put the mapping on
  the wire (AEP-REQ-159), a missing commitment, and a digest without its
  algorithm prefix. `invalid-binding-model-field.json` adds a `model` property
  to the session snapshot, which `additionalProperties: false` rejects.
- `conformance-invalid-*.json` — passes portable schema validation but violates a
  normative requirement that JSON Schema cannot express:
  - `conformance-invalid-experiment-id-names-model.json` uses an `experimentId`
    that hints at the models under comparison (AEP-REQ-155). The pattern allows
    the characters; only a reviewer or a deny-list knows they're model names.
- `commitment-preimage.json` — not a schema document. It is the operator-held
  preimage behind the `commitment` used by every fixture here and by the
  examples in spec §11.10 and `docs/AB-TESTING.md`. Canonicalizing it with
  RFC 8785 and hashing it with SHA-256 yields
  `976a28aafa5289e08d28fe5977e432ebcc06f59771c8f509dab9416f688e73fb`, which
  shows what a verifiable unblinding looks like (AEP-REQ-160). It is disclosed
  here only because it's an example; a real preimage never crosses the AEP
  surface.
