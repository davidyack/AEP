# Dataset Extension

*Portable dataset format for agent evaluation.*

**Extension ID:** `dataset` · **Spec section:** §11.6 · **Status:** v0.1

---

This is the companion document for the Dataset extension (AEP-REQ-072 through AEP-REQ-078). It explains the design, covers implementation guidance, and describes the foreign-format import patterns that tooling is expected to provide.

## Why this extension exists

AEP core defines per-session semantics: one agent, one session, one or more turns. Real evaluation workflows drive thousands of sessions from a curated set of inputs — a **dataset**. Without a shared dataset shape, every tool mints its own (OpenAI Evals JSONL, LangSmith examples, HuggingFace parquet, Braintrust examples, …). The friction is in interoperability, not in any one format being inadequate.

This extension defines a canonical shape so datasets port cleanly between AEP-compliant tools. Strictly additive: agents and servers that don't implement the extension remain fully compliant with the core.

## What's in scope

- A canonical JSON schema for a dataset document ([`../schemas/dataset.schema.json`](../schemas/dataset.schema.json))
- A discoverability hook on the Agent Contract (`datasetRefs`) so tools can surface "here are datasets that cover this agent"
- A conformance rule that dataset example inputs MUST validate against the agent's declared `inputSchema` when the dataset declares an `agentId` — closing the gap between dataset portability and AEP core

## What's out of scope

- Dataset storage, versioning, or distribution protocols
- Registries, marketplaces, or hosting conventions
- Mandatory adapters for foreign formats (OpenAI Evals, HuggingFace, LangSmith, Braintrust). These are tooling concerns; the spec defines the shape everyone lands in, not the conversion logic.
- Scoring methodology — datasets carry examples; how a judge or grader uses them is the grading tool's concern (see [`SCORING.md`](./SCORING.md))

## The dataset document

```json
{
  "id": "example.inquiry-quality-v1",
  "name": "Inquiry Assistant Quality Suite",
  "version": "1.0.0",
  "description": "Quality evaluation covering assumption-probing behaviours.",
  "agentId": "example.inquiry-assistant",
  "evaluator": "inquiry_quality",
  "policyProfileRef": "policy:example.default@2026-04-01",
  "sourceRef": "internal:pm-eval-library/inquiry-v1",
  "tags": ["quality", "inquiry"],
  "license": "CC-BY-4.0",
  "examples": [
    {
      "id": "ex-001",
      "input": { "message": "I want to add AI to our product." },
      "expected": { "response": "Clarifying question about the problem." },
      "tags": ["vague-request"],
      "difficulty": "easy"
    }
  ]
}
```

### Field semantics

| Field | Purpose |
|-------|---------|
| `id`, `name`, `version` | Identity. `version` is a semver-style monotonic string so tools can diff across versions. |
| `description` | Human-readable summary of the dataset's coverage. |
| `agentId` | Optional. When present, binds the dataset to a specific agent; example inputs MUST validate against that agent's `inputSchema`. |
| `evaluator` | Optional. Named RAI evaluator this dataset calibrates for (e.g. `harmful_content`, `hallucination`). |
| `policyProfileRef` | Optional. Ties the dataset to a Policy Profile (§11.1). Enables "which datasets exercise this rule set" queries. |
| `sourceRef` | Optional. Provenance pointer: HuggingFace slug, OpenAI Evals path, internal ticket, git URL. Required when imported from a foreign format (AEP-REQ-078). |
| `tags`, `license`, `metadata` | Classification and discovery aids. |
| `examples[]` | The actual data. At least one required. |

### The example object

| Field | Purpose |
|-------|---------|
| `id` | Stable identifier within the dataset. |
| `input` | The input submitted to the agent. Validated against `inputSchema` when `agentId` is set. |
| `expected` | Optional reference answer for reference-aware grading. |
| `references` | Optional array of multiple acceptable references, for judges that accept variant correct answers. |
| `context` | Optional per-example context to inject via the agent's `contextModel`. |
| `tags`, `metadata`, `difficulty` | Classification aids for filtering during evaluation runs. |

## Requirements summary

See [`CONFORMANCE.md`](./CONFORMANCE.md) for the canonical list. In brief:

- **AEP-REQ-072**: Advertise `"dataset"` in `capabilities.extensions`
- **AEP-REQ-073**: Datasets conform to `dataset.schema.json`
- **AEP-REQ-074**: When `agentId` is declared, inputs MUST validate against agent's `inputSchema` at import time (not just at runtime)
- **AEP-REQ-075**: Agents MAY publish `datasetRefs` in their Agent Contract for discoverability
- **AEP-REQ-076**: Datasets MUST NOT contain secrets, PII, or tenant-isolated data
- **AEP-REQ-078**: Foreign-format converters MUST populate `sourceRef` to preserve provenance

## Import from foreign formats

Foreign-format adapters are a tooling concern, not a protocol concern. The extension defines only the landing shape. Tooling SHOULD provide adapters for at least the three common sources.

### OpenAI Evals JSONL

One JSON object per line, typically one of these shapes:
```jsonl
{"input": "prompt text", "ideal": "expected output"}
{"inputs": [...], "expected": "..."}
```

Mapping to AEP dataset format:
- `input` or `inputs` → `examples[].input`
- `ideal` or `expected` → `examples[].expected`
- Wrap in a dataset envelope with generated `id`, `version`, and `sourceRef: "openai-evals:<path>"`

### HuggingFace Datasets

Via the `datasets` library's JSON or CSV export. Metadata comes from the dataset card.

Mapping:
- Each row → `examples[]` entry
- Dataset-card fields → top-level metadata
- `sourceRef: "hf:<slug>"` for provenance

### LangSmith / Braintrust

Both use an `{inputs, outputs, metadata}` example shape.

Mapping:
- `inputs` → `examples[].input`
- `outputs` → `examples[].expected`
- `metadata` → `examples[].metadata`
- `sourceRef: "langsmith:<project>"` or `sourceRef: "braintrust:<project>"`

## Consumer pattern (how to use a dataset in evaluation)

```
1. Fetch dataset document
2. Validate against dataset.schema.json
3. If agentId present, validate each examples[].input against agent's inputSchema
4. For each example:
   a. Start session (POST /aep/sessions) with example.context as injectedContext
   b. Execute turn (POST /aep/sessions/{id}/turns) with example.input
   c. Capture output
   d. Score output against example.expected or example.references using the chosen grading method
5. Aggregate scores into EvaluationResult
6. Close session
```

For high-throughput eval, step 4 parallelises naturally — each example is independent.

## Versioning and diffs

`version` is opaque to the protocol. Tooling is expected to implement semver-style comparison if needed. Key practices:

- Bump `version` whenever `examples[]` changes (add, remove, modify)
- Bump `version` when `expected` or `references` change (affects grading outcomes)
- Don't bump for `description`, `tags`, or `metadata` changes that don't affect grading

## Security

**AEP-REQ-076** prohibits secrets, PII, or tenant-isolated data in dataset documents. Concrete implications:

- Dataset publishers SHOULD run redaction passes before publishing
- Dataset consumers SHOULD scan received datasets for obvious secret patterns (API keys, JWT structure, email addresses)
- Datasets carrying sensitive content MUST be kept in private storage; the extension does not define encryption or access-control mechanisms

The rule is absolute: a dataset with inline secrets is non-compliant regardless of the access controls around the dataset's storage. Embed references to credential stores, not the credentials themselves.

## What this extension is not

- Not a replacement for scenario documents (`scenario.schema.json`). Scenarios carry branching, assertions, and failure conditions; datasets carry bulk inputs. Use scenarios for curated tests; use datasets for scale.
- Not a benchmark format. Datasets are inputs for evaluation; benchmarks are outputs (leaderboards, cross-agent comparisons). The extension is silent on benchmark publication.
- Not an eval harness. Tools that consume datasets to drive evaluation runs are implementations that sit on top of this extension; the extension defines the shared data shape, not the harness.
