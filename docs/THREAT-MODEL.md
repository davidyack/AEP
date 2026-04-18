# AEP Threat Model

This document addresses threats to and via the protocol itself, not deployment concerns. Deployment guidance — auth schemes, network isolation, credential rotation — is necessary but is not the subject here. The question this document answers is: *what can go wrong because of how the protocol is designed, and what does it do about it?*

## Scope

In scope:
- Threats arising from protocol semantics
- Trust boundaries between evaluator, server, and agent
- Information flow between artifacts (scenarios, bundles, traces, findings)
- Replay-specific threats

Out of scope:
- Transport security (TLS, mTLS, tunnels — implementation choices, not protocol choices)
- Credential lifecycle (issuance, rotation, revocation — deployment concern)
- Physical and network segmentation

## Actors

- **Evaluator (client)** — human or automated system that runs evaluations.
- **AEP Server** — exposes agents for evaluation.
- **Agent** — the system being evaluated.
- **Tool backends** — real or sandboxed systems the agent invokes.
- **Artifact storage** — where traces, findings, and bundles live.

## Trust boundaries

```
┌──────────────┐   trusted     ┌──────────────┐   trusted      ┌────────────┐
│  Evaluator   │ ───────────▶ │  AEP Server  │ ─────────────▶ │   Agent    │
└──────────────┘               └──────┬───────┘                └────────────┘
                                      │
                                      │ untrusted (scenarios may be authored elsewhere)
                                      ▼
                              ┌───────────────┐
                              │   Scenario    │
                              │   Artifact    │
                              └───────────────┘
```

The evaluator is trusted to the server (authenticated, authorized). The server is trusted to the agent (same organisation, same deployment). The agent is not trusted to the tool backends beyond what its tool allow-list permits. Scenario artifacts, however, can originate from anywhere — shared corpora, third-party red-team libraries, user-submitted test cases — and must be treated as untrusted input even when submitted by a trusted evaluator.

This last point is often missed and is the source of several protocol-level threats.

## Threats

### T1. Malicious scenario exfiltrates data via the agent

**Scenario.** A scenario is crafted to coax the agent into including sensitive data in its output, where the output is captured in a trace the attacker can later read.

**Conditions.** Attacker can submit scenarios and can read traces. This is typical in shared eval environments.

**Mitigation.** Scenarios MUST be validated against a policy before execution. Traces inherit the sensitivity of their inputs and MUST be access-controlled accordingly. Evaluators SHOULD run untrusted scenarios only against agents with no access to data the scenario author isn't authorized to see.

**Residual risk.** Validation catches structural attacks, not semantic ones. A scenario that looks innocuous but extracts information through subtle probing may pass validation. Access control on traces is the stronger mitigation.

### T2. Server lies about ContextBundle

**Scenario.** The AEP server returns a ContextBundle that does not match what the underlying agent actually does. Evaluators produce findings against a false picture, and the real agent's behavior goes unexamined.

**Conditions.** Server is compromised, or ContextBundle has drifted from reality.

**Mitigation.** The *Surface Parity Check* pattern addresses this: run representative scenarios against both the user surface and the evaluation surface, and assert behavioral equivalence. Bundles SHOULD include `lastReviewedAt` timestamps; stale bundles trigger review.

**Residual risk.** A sophisticated adversary controlling the server could fake parity checks too. At that point, the trust boundary has already been crossed and protocol-level mitigations are insufficient.

### T3. Replay attack with stale privileges

**Scenario.** A trace captured in a permissive mode (e.g. toolbox, where tool calls were visible) is replayed in a context that should only have blackbox access. The replayer sees information they shouldn't have.

**Conditions.** Trace storage and replay access controls are misaligned.

**Mitigation.** Traces MUST carry their originating mode. Replay access controls MUST honor the mode: a client authorized for blackbox only MUST NOT be able to replay a toolbox trace. The protocol makes this the server's responsibility; servers MUST reject replay requests where the caller's mode authorization is below the trace's captured mode.

**Residual risk.** Trace export to external systems loses this protection. The protocol cannot enforce access control on data that has left its boundary.

### T4. Scenario confuses an agent in production

**Scenario.** A scenario crafted for evaluation finds its way into production traffic (via logs, via scenario-sharing, via accidental submission), and triggers behavior the agent would not normally exhibit.

**Conditions.** Evaluation and production surfaces share underlying infrastructure, and scenario-scale inputs can reach the production surface.

**Mitigation.** The *Second Door* pattern is the structural mitigation: the evaluation surface is distinct from the production surface. Scenarios authenticate through the evaluation surface and cannot reach production without crossing a trust boundary.

**Residual risk.** Real, not theoretical: agent prompts and tool calls can leak between surfaces if the surfaces aren't truly isolated. The pattern is a pattern; actual isolation is an implementation concern.

### T5. Sandbox has higher privilege than it advertises

**Scenario.** A tool is described as "sandboxed" but the sandbox can reach production systems. The agent executes tool calls that appear safe but have real consequences.

**Conditions.** Sandbox fidelity is overstated in the ToolCapability descriptor.

**Mitigation.** The ToolCapability schema distinguishes sandbox fidelity levels (`none | stub | recorded | simulated | real-isolated`). Honest declaration is the first line of defence. Evaluation servers SHOULD verify sandbox claims at session creation where possible — e.g. by blocking the sandbox's network egress to production endpoints.

**Residual risk.** Verification is partial. A sandbox that passes egress checks may still have indirect effects (timing side channels, shared caches, log correlation). Treat sandbox fidelity as a best-effort claim, not a guarantee.

### T6. Findings exfiltrate via evidence references

**Scenario.** A finding's `evidenceRefs` point to artifacts the finding consumer shouldn't be able to read, but dereferencing the pointer bypasses access control.

**Conditions.** Evidence pointers are treated as opaque capabilities instead of references requiring their own authorization.

**Mitigation.** Dereferencing an evidence pointer MUST require the caller to have independent authorization for the referenced artifact. Pointers are not capabilities; they are hints.

**Residual risk.** Misconfigured implementations that trust pointers. The protocol states the requirement; implementations must honor it.

### T7. Compounding cost attack via adaptive scenarios

**Scenario.** An adaptive scenario (using the `adaptive` input type) is authored to produce unbounded cost — the attacker model generates increasingly expensive prompts that drive up model inference costs on the agent being tested.

**Conditions.** No cost ceilings on scenario execution.

**Mitigation.** Servers MUST enforce a maximum cost per scenario, expressed as turn count, token budget, or cost-in-currency. Scenarios that exceed the ceiling are aborted. This is a capability the v0.1 spec should make explicit; it is currently implicit in session TTLs.

**Residual risk.** Cost ceilings are crude. A below-ceiling but high-cost scenario run many times is still expensive. Rate limiting at the evaluator level complements ceiling-per-scenario.

### T8. Scenario exposes sensitive prompts via traces

**Scenario.** A scenario contains secrets in its input (intentionally or by mistake — e.g. testing prompt injection against a known-compromising string). The trace captures the input verbatim, and trace access controls are weaker than scenario access controls.

**Conditions.** Trace storage does not inherit scenario sensitivity.

**Mitigation.** Traces MUST inherit the sensitivity labels of their inputs. Servers SHOULD support redaction at capture time for scenarios tagged as sensitive. The `ModelCallRecord` in traces supports `promptDigest` as an alternative to full prompt capture for exactly this reason.

**Residual risk.** Redaction is hard; secrets embedded in natural language are not always obvious. Err toward redaction when in doubt.

## Threat-to-mitigation summary

| Threat | Primary mitigation | Pattern / schema element |
|--------|--------------------|--------------------------|
| T1 Malicious scenario exfiltrates | Scenario validation + trace ACLs | Evidence Pointer pattern |
| T2 Server lies about bundle | Surface Parity Check | `lastReviewedAt`, parity scenario |
| T3 Replay with stale privileges | Mode-aware replay authorization | Six-mode taxonomy |
| T4 Scenario hits production | Separate surfaces | Second Door pattern |
| T5 Sandbox overstated | Honest fidelity declaration | `ToolCapability.sandbox.fidelity` |
| T6 Evidence ref exfiltration | Pointer ≠ capability | Evidence Pointer pattern |
| T7 Cost attack | Cost ceiling per scenario | (v0.2 work) |
| T8 Sensitive scenario capture | Trace sensitivity inheritance | `promptDigest`, redaction |

## What's out of scope for the protocol

The protocol cannot prevent:
- Compromised evaluator accounts
- Collusion between evaluator and agent team
- Side channels outside the protocol's observation (CPU cache, power, timing)
- Social engineering
- Supply chain attacks on implementations

These are real threats. They are deployment and operations concerns, not protocol concerns. Calling them out here is meant to prevent the document from being read as complete when it is only partial.
