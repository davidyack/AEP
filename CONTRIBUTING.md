# Contributing to AEP

Thank you for considering a contribution. AEP is a small project with an explicit point of view, and your contribution is more likely to land quickly if you read this first.

## What we're optimising for

**Evidence over opinion.** If you have an implementation that hit a limitation in the spec, that matters more than a thought about what might matter. "We tried to do X and the schema wouldn't let us" is a strong contribution. "The schema should probably support X" is a weaker one.

**Keep the pattern language and the wire protocol separate.** Proposals that conflate the two (adding protocol methods for concepts not in the patterns, or adding patterns that only make sense at the wire level) will usually be sent back for disambiguation.

**Don't unify prematurely.** Where the spec deliberately leaves surface exposed (scoring, rubrics, matcher schemas), the burden of proof is on proposals to unify it. See [`docs/SCORING.md`](./docs/SCORING.md) and [`docs/RATIONALE.md`](./docs/RATIONALE.md).

## How to contribute

### Reporting issues

Use the issue tracker. Useful issues include:

- A concrete evaluation scenario the spec doesn't support well
- A schema field you found ambiguous and how you resolved it
- A case where two implementations diverged because the spec was silent
- A threat or failure mode not covered in [`docs/THREAT-MODEL.md`](./docs/THREAT-MODEL.md)

Less useful (though still welcome):

- "Have you considered X?" without context on what problem X solves
- Style or naming preferences that aren't driven by implementation pain

### Proposing changes

Small changes (fixing typos, clarifying ambiguities, adding examples): open a PR directly.

Substantive changes (new methods, new schema fields, changes to existing semantics): open an issue first describing the problem before proposing a solution. We'd rather talk about the problem than negotiate a patch.

### What we're unlikely to accept

- Additions that only matter to one vendor. Use the vendor extension mechanism (reverse-DNS prefixes on capabilities and methods).
- Proposals to make AEP compete with MCP. AEP is evaluation-focused; if your proposal would make sense as an MCP extension, it probably should be one.
- Protocol-level specifications of RAI benchmarks or scoring methods — these churn faster than protocols.

## Style

- Markdown files use ATX-style headings (`#`, `##`) and reference-style links where links would clutter prose.
- JSON schemas are Draft 2020-12.
- Keep prose in British or American English — pick one per document and be consistent.
- Avoid marketing language. This is a spec, not a pitch.

## Review

A PR is reviewed on three questions:

1. Does it solve a real problem an implementer has?
2. Does it fit the existing vocabulary (patterns, schemas, methods)?
3. Does it maintain the separation between pattern language and wire protocol?

Changes that fail any of these will get feedback rather than an immediate merge.

## Governance

AEP is maintained by its original authors. There is no formal governance structure at v0.1 because the project is too small to need one. If that changes, we'll describe it here.

## Code of Conduct

See [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
