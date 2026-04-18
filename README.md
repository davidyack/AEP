# AEP — Agent Evaluation Protocol

**Version:** 0.1 · **License:** MIT

AEP is a protocol for exposing AI agents to evaluation systems without going through the product UI they live in. One normative specification, canonical JSON schemas, and a pattern language.

## Start here

**If you're implementing AEP** → [`docs/WALKTHROUGH.md`](./docs/WALKTHROUGH.md). One end-to-end example from discovery to result retrieval, with REST and JSON-RPC side-by-side. Ten minutes.

**If you're evaluating AEP** → [`patterns/TEST-MODES.md`](./patterns/TEST-MODES.md). Five minutes, the portable vocabulary.

**If you're writing a conformance suite or implementation checklist** → [`docs/CONFORMANCE.md`](./docs/CONFORMANCE.md). Every requirement indexed by stable ID.

**Authoritative spec** → [`spec/AEP-0.1.md`](./spec/AEP-0.1.md). Self-contained; everything else is commentary.

## Why this exists

Most AI agents today live inside a product — a chat surface, a workspace, a workflow. To evaluate them, you log in, create a workspace, navigate to the right screen, assemble context, and only then interact with the agent. Every test run pays that tax. Automated regression suites become brittle Playwright scripts. Red teaming becomes manual. CI becomes aspirational.

AEP gives the agent a second door — an evaluation surface, separate from the user surface, designed specifically for automated and human-in-the-loop testing. Same agent, different door.

## What's in the spec

Required core (every compliant implementation):
- **Agent Contract** every agent publishes: identity, input/output schemas, context model, capabilities
- **Discovery with filtering, pagination, and incremental sync** — `GET /aep/agents` returns summaries with `authorizedModes`, filterable by domain, mode, extension, risk, and lifecycle
- **Five-state session lifecycle**: created → active → evaluated → reported → closed
- **Equivalent REST and JSON-RPC surfaces** — REST for ergonomics, JSON-RPC for batching and negotiation
- **Context injection** so tests are reproducible
- **Execution model** specifying when scenarios complete and how scoring fires
- **Trace production** for every session, sealed and replayable
- **19 standardised error codes** with retry semantics
- **Enforced security model**: auth, authorisation, rate limits, isolation
- **Conformance declaration** at `/.well-known/aep-compliance.json`

Optional extensions (implement if you need them):
- **Policy Profile** — machine-readable policy rules
- **Tool Capability** — structured tool descriptors
- **Red-teaming** — adversarial testing interface
- **Deterministic replay** — byte-identical reproduction

Every MUST in the spec carries an `AEP-REQ-NNN` identifier; a future conformance test suite will reference these directly.

## What's in the repo

```
aep/
├── spec/
│   ├── AEP-0.1.md           # THE spec — self-contained, normative
│   └── CHANGELOG.md
├── schemas/                  # Canonical data contracts
│   ├── agent-contract.schema.json           ← required
│   ├── agent-summary.schema.json            ← required
│   ├── agent-descriptor.schema.json         ← required
│   ├── capabilities-registry.schema.json    ← required
│   ├── context-bundle.schema.json           ← required
│   ├── scenario.schema.json                 ← required
│   ├── evaluation-session.schema.json       ← required
│   ├── evaluation-result.schema.json        ← required
│   ├── trace.schema.json                    ← required
│   ├── finding.schema.json                  ← required
│   ├── policy-profile.schema.json           ← optional extension
│   ├── tool-capability.schema.json          ← optional extension
│   ├── stream-event.schema.json             ← optional extension
│   └── dataset.schema.json                  ← optional extension
├── patterns/
│   ├── PATTERNS.md          # Pattern language (non-normative)
│   └── TEST-MODES.md        # Six-mode taxonomy (portable)
├── docs/
│   ├── WALKTHROUGH.md       # Happy-path end-to-end example
│   ├── CONFORMANCE.md       # AEP-REQ-NNN index
│   ├── RATIONALE.md         # Design decisions
│   ├── THREAT-MODEL.md      # Threat analysis
│   ├── SCORING.md           # Scoring concerns not unified in spec
│   ├── REPLAY.md            # Replay design framing
│   ├── RED-TEAMING.md       # Adversarial interface spec
│   ├── DATASET.md           # Dataset extension companion
│   ├── CSE.md               # Composite Session Evaluation extension
│   └── ROADMAP.md
├── examples/
│   └── reference-agent/     # Fictional agent with full contract and scenarios
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
└── LICENSE
```

Everything in `docs/` is non-normative. The spec is self-contained; read it alone and you have everything you need to build a compliant implementation.

## Scope

What AEP is:
- A wire protocol for discovering, inspecting, and exercising agents
- A small required core with optional extensions for richer scenarios
- A pattern language for describing evaluation concerns

What AEP is not:
- A chatbot protocol
- A tool-execution protocol (that's MCP; spec §3)
- A replacement for observability or tracing
- A user-facing interface

## Status

v0.1. First public draft — April 2026. The spec is normative and self-contained; the schemas are validated; the requirements are stable IDs; the optional extensions are designed to be adopted independently. Breaking changes will be versioned and documented in [`spec/CHANGELOG.md`](./spec/CHANGELOG.md).

### What's here and what isn't

**What's here:** the protocol itself — wire surface, data contracts, required lifecycle, error handling, security model, optional extensions (Policy Profile, Tool Capability, Red-teaming, Deterministic Replay, Streaming, Dataset, CSE hooks). Enough to implement an AEP server or client from the spec alone.

**What's not here yet:** a reference implementation. The most common next question after "is the spec sound?" is "show me the smallest working server/client pair." That will live in a companion `aep-reference` repository, not in this one, so the spec can evolve without being coupled to any single implementation's test harness. Tracking in [`docs/ROADMAP.md`](./docs/ROADMAP.md); this is the highest-priority v0.2 deliverable.

Other v0.2 work: conformance test suite (each test cites an `AEP-REQ-NNN`), canonical binary trace serialisation, cost attribution as a protocol concern. See [`docs/ROADMAP.md`](./docs/ROADMAP.md) for the full list.

## Attribution

First public draft: **April 2026**. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) to get involved.

## License

MIT. See [`LICENSE`](./LICENSE).
