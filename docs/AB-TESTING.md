# Blind A/B Model Comparison

**Status:** Non-normative companion to spec §11.10 (extension id `ab-testing`). The requirements live in the spec (AEP-REQ-154 through AEP-REQ-163); this document explains how to use them.

## The problem

You run an agent on model X and are considering model Y. You want a fair answer to "is Y better for *this agent*?" Two things get in the way:

1. **Bias.** If the evaluator — a human rater, a red-teamer, or the person writing the LLM-as-judge rubric — knows which output came from the shiny new model, their judgement shifts. Blinding is standard practice in experiments for this reason.
2. **Plumbing.** Without a protocol hook, comparing models means standing up a second copy of the agent, which differs from the first in more than the model, and wiring the evaluator to both.

The extension solves both: one agent, one contract, a `variant` field at session start, and a guarantee that the protocol never says which model is behind `A` or `B`.

## Roles

| Who | Knows the mapping? | Does |
|-----|-------------------|------|
| **Operator** (runs the AEP server) | Yes | Configures which model each label maps to, publishes the commitment, decides when to unblind |
| **Evaluator** (tester, harness, grader) | No | Picks `A` or `B` per session, runs identical scenarios, compares results |
| **Auditor** (optional) | After unblinding | Checks the disclosed mapping against the commitment recorded in every Trace |

## End-to-end

### 1. Discover

The agent's contract advertises the extension:

```json
"capabilities": {
  "extensions": ["ab-testing"],
  "extensionData": {
    "ab-testing": {
      "version": "1.0.0",
      "payload": {
        "experimentId": "exp_7Qm2xR9kLp",
        "experimentVersion": "3",
        "variants": ["A", "B"],
        "selection": "optional",
        "commitment": "sha256:976a28aafa5289e08d28fe5977e432ebcc06f59771c8f509dab9416f688e73fb"
      }
    }
  }
}
```

`GET /aep/agents?extension=ab-testing` finds every agent with an experiment running.

### 2. Run variant A

```http
POST /aep/sessions
{
  "agentId": "inquiry-agent",
  "testMode": "graybox",
  "variant": "A",
  "injectedContext": { "...": "..." }
}
```

```json
{
  "sessionId": "s_Ak3...",
  "agentId": "inquiry-agent",
  "agentVersion": "2.4.0",
  "testMode": "graybox",
  "variant": {
    "experimentId": "exp_7Qm2xR9kLp",
    "experimentVersion": "3",
    "label": "A",
    "commitment": "sha256:976a28aa…",
    "boundAt": "2026-11-11T10:00:00Z"
  },
  "state": "active",
  "createdAt": "2026-11-11T10:00:00Z",
  "expiresAt": "2026-11-11T10:15:00Z"
}
```

JSON-RPC is the same with `aep.session.start` and `"variant": "A"` in `params`.

### 3. Run variant B with everything else identical

Same `testMode`, same `injectedContext`, same `identityId`, same scenarios, same `seed` if the agent is seeded. Only `variant` changes. Interleave A and B sessions (A, B, B, A, …) rather than running all of A first, so time-of-day effects and provider drift fall evenly on both.

### 4. Compare

Every `EvaluationResult` carries the `variant` snapshot, so grouping is a filter:

```text
group results by (variant.experimentId, variant.experimentVersion, variant.label)
```

If `experimentVersion` changes partway through, stop. The mapping changed; results from before and after are not the same variant (AEP-REQ-161). Start a new comparison.

For preference judging (show a rater an A output next to a B output and ask which is better), randomize left/right placement per pair. The labels are already blind; the position shouldn't be a cue either.

### 5. Unblind (outside AEP)

When the operator decides the comparison is over, they publish the commitment preimage through whatever channel they like (a report, a ticket, an email):

```json
{
  "experimentId": "exp_7Qm2xR9kLp",
  "experimentVersion": "3",
  "assignments": {
    "A": "provider-y/model-y-2026-09 temperature=0.2 prompt-rev=14",
    "B": "provider-x/model-x-2026-03 temperature=0.2 prompt-rev=12"
  },
  "salt": "q8Zf0v3pX1nT7cKe9LmR2w"
}
```

Anyone can canonicalize it with RFC 8785, hash it, and compare against the `commitment` in the contract and in every Trace. A match proves the operator didn't swap labels after seeing which one won.

## Why the design looks like this

**Why single letters?** Free-text labels drift into `gpt-ish`, `new`, `fast`, `control`. A one-character pattern keeps the schema itself enforcing opacity.

**Why a commitment at all?** Blinding protects the evaluator from bias. The commitment protects the evaluator from the *operator*: without it, "A won, and A was model Y all along" can't be checked. With it, the claim can be verified.

**Why salt?** A bare hash of `{A: "model-x", B: "model-y"}` can be brute-forced from the short list of plausible models. 128 bits of salt makes that infeasible.

**Why no fallback under a label?** If B's provider has an outage and the server quietly serves B's sessions from A's model, the comparison is corrupted, and nobody can tell. Failing with `agent_unavailable` is visible; silent substitution is not (AEP-REQ-158).

**Why a new agent version when the mapping changes?** Swapping the model materially changes behavior, and AEP-REQ-140 already says that requires a contract version bump. It also means `agentVersion` plus `experimentVersion` pin exactly what ran.

**Why an extension and not core?** Most agents never run a model comparison. Keeping it optional keeps the Core Profile small, and the only core changes are optional `variant` fields that mean nothing unless the extension is advertised. That follows the same pattern as Observability Correlation (§11.9).

## Residual leakage

The protocol doesn't name the model. That doesn't make the model impossible to guess. Operators should know where guesses come from:

| Channel | Example | Mitigation |
|---------|---------|-----------|
| Self-identification in output | "As a model trained by …" | Agent system prompt instructs the model not to name itself; this is behavior, so it's part of what's being evaluated |
| Style fingerprinting | Recognizable phrasing or formatting habits | None at the protocol level; accept it, or use automated graders that don't care |
| Token counts | Tokenizers differ, so `inputTokens` for the same prompt differs per model | Under `modelCalls: metadata` or higher, consider reporting counts in a normalized unit, or omitting them in blinded sessions |
| Latency | One model is reliably faster | Comparing latency is often the point; if it isn't, coarsen `latencyMs` |
| Context window / refusal behavior | Very long inputs fail on one variant only | The failure is legitimate evidence; it's also a hint. Nothing to do |
| External telemetry | Span attributes in the APM backend name the model | AEP-REQ-159(d): AEP carries only correlation IDs; restrict evaluator access to the telemetry backend for the experiment's duration |

The spec draws the line at "the protocol never tells you." Making the model *unguessable* isn't possible without changing the evidence being evaluated.

## Implementation checklist

Server:
- [ ] Advertise `ab-testing` at `aep.initialize` and in the conformance declaration
- [ ] Publish a schema-valid `extensionData["ab-testing"]` entry for each agent with an experiment; refuse to start without it
- [ ] Generate a fresh ≥128-bit salt and a new commitment for every `experimentVersion`; keep the preimage as long as the Traces
- [ ] Assign labels to configurations at random per `experimentVersion`
- [ ] Validate `variant` at session start after agent and mode checks; return `-32080` with `data.reason`
- [ ] Resolve the binding at created→active and write the same snapshot to session, Trace, result, and `session.started`
- [ ] Rewrite `ModelCallRecord.model` to `variant:<label>`; scrub deployment names, endpoints, and provider names from parameters, bundles, tool detail, error text, and headers
- [ ] Never route a label to another model; surface provider failures as `-32061` / `-32060`
- [ ] Re-execution replay binds the original label and version or fails with `-32081`
- [ ] Audit-log every binding with the real configuration, in a log evaluator credentials can't read

Evaluator:
- [ ] Read `variants` and `selection` from the contract before starting sessions
- [ ] Hold mode, context, identity, scenarios, and seed constant across variants
- [ ] Interleave variant sessions; randomize presentation order for preference judging
- [ ] Group by (`experimentId`, `experimentVersion`, `label`); never pool across versions
- [ ] Record the commitment alongside conclusions so they can be checked after unblinding
