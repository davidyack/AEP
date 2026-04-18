# Reference Agent Example

This directory shows how a representative product-embedded agent would describe itself under AEP. The agent is fictional — it exists only to make the schemas concrete. Do not adopt it as-is; use it as a shape to imitate.

## Files

| File | Demonstrates |
|------|--------------|
| [`agent-contract.json`](./agent-contract.json) | The full Agent Contract — inputs, outputs, context model, capabilities, determinism, endpoints, limits, traceability. This is what an AEP-compliant agent MUST publish. |
| [`agent-descriptor.json`](./agent-descriptor.json) | Identity, risk tier, lifecycle, owner (a subcomponent of the Agent Contract shown standalone) |
| [`context-bundle.json`](./context-bundle.json) | Intended use, non-goals, behavioral boundaries, tool capability descriptors |
| [`scenarios/quality-probes-assumptions.json`](./scenarios/quality-probes-assumptions.json) | A linear quality scenario with multiple assertion types |
| [`scenarios/safety-refuses-adversarial.json`](./scenarios/safety-refuses-adversarial.json) | A branching safety scenario with conditional next-turn selection |
| [`datasets/inquiry-quality.json`](./datasets/inquiry-quality.json) | A dataset demonstrating the Dataset extension (§11.6) — five examples with mixed `expected`/`references` shapes |

## What to notice

**The `nonGoals` array.** It's longer than the `intendedUse` array. This is usually correct. Agents fail more often by doing things they shouldn't than by failing to do things they should, and evaluators need the negative space to be explicit.

**Tool capability structure.** Each tool has `reversibility`, `observability`, `blastRadius`, and `sandbox` fidelity. A single `sideEffectLevel` enum would flatten distinctions that matter — a read-only query and a sandboxed notification look alike by "side effect" but differ substantially in evaluation implications.

**Branching scenario.** The safety scenario conditions its second turn on whether the first turn produced a refusal. Linear scenarios can't express "what happens if the agent fails at step one and then we push harder" — which is exactly the shape of real adversarial testing.

**Typed evidence references.** Scenarios don't include evidence refs directly (those are produced during runs), but the schema for Findings uses typed pointers into traces — `trace-turn`, `tool-call`, `policy-event`, `external` — not opaque strings. This makes triage navigable instead of forensic.

## Running the example

There is no runnable server in this repo. The example is declarative — it shows what an AEP-compliant agent would publish. A reference server implementation is planned for v0.2.

To validate the JSON against the schemas:

```bash
npx ajv validate -s ../../schemas/agent-descriptor.schema.json -d agent-descriptor.json
npx ajv validate -s ../../schemas/context-bundle.schema.json -d context-bundle.json --all-errors
npx ajv validate -s ../../schemas/scenario.schema.json -d 'scenarios/*.json' --all-errors
```

(The `--all-errors` flag surfaces every validation issue rather than halting at the first.)
