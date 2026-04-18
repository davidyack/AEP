# Red-Teaming Interface

*Adversarial testing against AEP-compliant agents.*

---

Red-teaming is evaluation where the evaluator's goal is to make the agent fail. It has distinct requirements from normal quality testing: scenarios adapt, scoring emphasises refusal fidelity, and rate limits and observation levels differ from standard evaluation.

This document specifies the required behaviour of AEP servers that claim red-teaming support.

## Applicability

A server claims red-teaming support by advertising `redteam: true` in its capabilities at `aep.initialize`. Servers **MAY** claim red-teaming support without implementing every **SHOULD** in this document, but **MUST** implement every **MUST**.

## Required capabilities

### Session requirements

- Red-teaming sessions **MUST** be created with `testMode` of `policybox`, `toolbox`, or `sandboxed-live`. `blackbox` and `graybox` do not surface policy events and are therefore unsuitable for adversarial observation.
- Sessions **MUST** carry `metadata.redteam: true` to flag the session for attack-focused logging.
- Servers **MUST** retain red-teaming session traces for at least 30 days, independent of standard trace retention.

### Adaptive scenarios

- Servers **MUST** support the `adaptive` input type defined in `scenario.schema.json`.
- An `adaptive` input carries an attacker-model prompt rather than a literal input. The server (or evaluator) generates the concrete input by running the attacker prompt against the prior turns' context.
- Servers **MUST** record both the attacker prompt and the generated input in the Trace. The generated input's provenance (which model, which parameters) **MUST** be recoverable.

### Refusal and escalation observation

- All refusals, redactions, and escalations during red-teaming sessions **MUST** surface as explicit `PolicyEvent`s in the Trace.
- Silent refusals (the agent declined without a detectable policy event) **MUST** be flagged by the server with a synthetic `PolicyEvent` of outcome `inferred-refusal`.
- Compounding failures — where the agent refuses turn 1 but capitulates on turn 3 — **MUST** produce findings with severity `critical` and a dedicated category `compounding-failure`.

### Rate limits

- Red-teaming sessions **MAY** be granted elevated rate limits compared to standard sessions, to support high-throughput adaptive attacks.
- Servers granting elevated limits **MUST** apply a separate quota and **MUST NOT** allow red-teaming traffic to consume quality-testing quota.
- Rate-limited red-teaming requests **MUST** return error `-32020` with a `Retry-After` hint, like any other rate-limited request.

## Recommended capabilities

### PyRIT integration

- Servers **SHOULD** accept PyRIT-generated attack prompts in `adaptive` input fields. PyRIT is a common adversarial tooling library; interoperability reduces integration cost.
- A PyRIT-originated input **SHOULD** be tagged in the Trace with `metadata.origin: "pyrit"` and the version used.

### Attack categorisation

Servers **SHOULD** categorise red-team scenarios using a shared vocabulary. Recommended categories:

- `jailbreak` — attempts to subvert safety instructions
- `prompt-injection` — attempts to override system instructions via input
- `data-exfiltration` — attempts to extract training data or system prompts
- `impersonation` — attempts to make the agent assume an unauthorised identity
- `privilege-escalation` — attempts to invoke tools or capabilities outside declared scope
- `adversarial-persuasion` — attempts to wear down refusals through escalation or reframing
- `context-poisoning` — attempts to manipulate injected context to influence downstream behaviour

These categories **MAY** be extended; custom categories **MUST** be prefixed with reverse-DNS.

### Attack surface coverage

Servers **SHOULD** expose, via the Trace or a dedicated method, which attack categories a session exercised. This lets evaluators detect coverage gaps — a red-team suite that never exercises `context-poisoning` has a blind spot worth naming.

## Findings and results

- Findings from red-teaming sessions **MUST** use the `safety`, `security`, or `robustness` categories of the `Finding` schema.
- Severity **MUST** reflect impact, not intent. An attack that succeeded once in a hundred tries is still critical if the success case produces a compliance violation.
- The `reproducible` field of the Finding **SHOULD** be populated — red-team findings with `frequency < 1.0` are typical, and consumers need the frequency to triage.

## Scoring

Red-team scores differ from quality scores:

- **Refusal rate** — fraction of adversarial scenarios where the agent refused correctly.
- **Refusal durability** — of scenarios where the agent refused initially, fraction where the refusal held through follow-up pressure.
- **Compounding failure rate** — fraction of sessions where a correct refusal at turn N was followed by a capitulation at turn N+k.
- **Category coverage** — fraction of the configured attack categories exercised.

Servers producing red-team EvaluationResults **SHOULD** populate these in `scores` using the names above.

## Isolation requirements

Red-teaming touches the most sensitive failure modes. Servers **MUST**:

- Isolate red-teaming traffic from quality-testing traffic at the rate-limit and logging layers.
- Reject red-teaming sessions against agents marked `lifecycle.stage: "ga"` unless the caller holds an explicit `redteam-production` authorisation.
- Require elevated authentication (mTLS or equivalent) for red-teaming endpoints. Bearer tokens alone are insufficient for red-teaming against non-dev agents.

## What red-teaming is not

This interface is for pre-release and staging adversarial testing. It is **not**:

- A runtime safety monitor (that's an observability concern)
- A replacement for human red-team judgement
- A certification of safety (passing red-team suites is necessary but not sufficient for safety claims)

Red-teaming produces evidence. What the evidence means for deployment decisions is a human call.
