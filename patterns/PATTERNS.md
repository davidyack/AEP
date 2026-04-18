# Patterns of Agent Evaluation

*A pattern language for testing AI agents that live inside products.*

---

This document is the conceptual half of AEP. It describes recurring problems in agent evaluation and the patterns we've found useful for addressing them. The patterns are independent of any particular wire protocol — you can adopt the vocabulary without adopting AEP's methods, and you probably should start there.

Each pattern follows a compact form: **Context**, **Problem**, **Forces**, **Pattern**, **Consequences**. This isn't rigidly Alexandrian but it keeps the thinking honest.

---

## Pattern 1: Second Door

**Context.** An agent is reachable only through the product it lives inside — login, workspace, navigation, context assembly.

**Problem.** Every test run pays the ceremony tax. Automation becomes brittle UI scripting. Evaluators spend more time setting up than testing.

**Forces.**
- The product surface is designed for humans, not test harnesses.
- The agent's behaviour depends on product context (workspace, prior messages, persona).
- Exposing the agent raw bypasses the context and produces misleading results.
- Hardening a test endpoint is real engineering work.

**Pattern.** Give the agent a **second door** — an evaluation surface that is architecturally distinct from the user surface, but that invokes the same underlying agent with the same context-assembly logic. The user door and the evaluation door share the agent; they do not share the chrome.

**Consequences.** Tests run without UI ceremony. The agent under test is the same agent users interact with. The cost is ongoing: the evaluation surface must be maintained alongside the user surface, and drift between them is a real risk (see *Surface Parity Check*).

---

## Pattern 2: Context Bundle

**Context.** Evaluators need to know what an agent is for before they can tell whether it's working.

**Problem.** Agent teams routinely fail to document intent, boundaries, and non-goals. Evaluators reverse-engineer the agent from its API surface and arrive at conclusions the agent's authors would reject.

**Forces.**
- Intent lives in designers' heads and rarely gets written down.
- "What should this agent refuse?" is harder to articulate than "what should it do."
- Policy documents exist elsewhere and get stale.
- Evaluators without intent context produce findings that are noise to the agent team.

**Pattern.** Curate a **Context Bundle** that travels with the agent: user-facing purpose, example tasks, known constraints, behavioural boundaries, policy summary, and a tool summary. Make it a first-class artifact, not an afterthought. Require it as a precondition for evaluation.

**Consequences.** Evaluators arrive oriented. Agent teams are forced to articulate intent. Findings become precise because they reference explicit boundaries. The cost is authoring and maintenance — a Context Bundle that drifts from reality is worse than none.

**Related:** *Spec as Context Source* — if the agent was built spec-first, the Context Bundle should be derived from the spec, not written separately.

---

## Pattern 3: Evaluation Mode Spectrum

**Context.** "Black-box testing" and "white-box testing" are not enough categories for agent evaluation.

**Problem.** Agent evaluation involves varying levels of internal visibility: tool calls, policy events, intermediate reasoning, underlying model inputs. Teams conflate these levels and produce test suites that can't be reproduced or interpreted.

**Forces.**
- Different questions need different visibility levels.
- Security questions need policy visibility; quality questions often don't.
- Visibility costs (performance, surface area, trust).
- Visibility should be granted explicitly, not leaked accidentally.

**Pattern.** Define an explicit **mode spectrum** with named levels. Each level grants specific visibility and nothing more. Modes are negotiated at session creation, not assumed.

See [`TEST-MODES.md`](./TEST-MODES.md) for the six modes we've found useful.

**Consequences.** Conversations about test coverage become precise. "We test in graybox" means something concrete. The cost is upfront design: each mode needs a clear definition of what is exposed and what is not.

---

## Pattern 4: Finding Over Pass/Fail

**Context.** Traditional test frameworks produce booleans. Agent evaluation rarely warrants one.

**Problem.** An agent response that is "technically correct but concerning" has no home in pass/fail. A response that violates policy in a minor way looks the same as catastrophic failure. Aggregate metrics lose the texture that matters.

**Forces.**
- Agent outputs are probabilistic and graded.
- Severity matters as much as presence.
- The same observation can be safety-relevant, quality-relevant, or both.
- Evidence matters: a finding without a traceable artifact is a rumour.

**Pattern.** Produce **Findings**, not pass/fail. A Finding has a category (safety / quality / security / performance), a severity, an observed behaviour, an expected behaviour, and typed references to the evidence. Aggregate boolean metrics are derived from Findings, not the other way around.

**Consequences.** Triage becomes meaningful. Regression detection keys off severity, not presence. The cost is that every evaluation must produce structured output, and teams used to green-bar thinking must adjust.

---

## Pattern 5: Scenario as Executable Artifact

**Context.** Test cases need to be reproducible, shareable, and version-controlled.

**Problem.** Scenarios written as prose, or as ad-hoc scripts, are not reproducible across runs, evaluators, or agent versions. Red-team findings become anecdotes.

**Forces.**
- Scenarios need to capture intent (what's being tested) and procedure (how).
- Multi-turn scenarios need state and branching, not just linear message lists.
- Adversarial scenarios need to adapt to agent responses.
- Deterministic scenarios need to be distinguishable from model-in-the-loop ones.

**Pattern.** Treat scenarios as **executable artifacts** with a formal structure: identity, category, persona, a state machine of turns, expected behaviours, failure conditions, and a scoring rubric. Linear message lists are a degenerate case.

**Consequences.** Scenarios can be versioned, diffed, and replayed. A failing scenario produces a reproducible trace, not a story. The cost is authoring effort — scenarios become software, and software requires maintenance.

---

## Pattern 6: Evidence Pointer, Not Evidence Copy

**Context.** Findings reference what happened during evaluation. Traces can be large.

**Problem.** Copying evidence into every finding bloats artifacts and creates consistency problems. Opaque string references ("evidence-1234") lose the ability to navigate back to the specific turn, tool call, or policy event.

**Forces.**
- Evidence volume can be large (multi-turn traces, tool call payloads).
- Findings need to survive independently of any one evaluator's workspace.
- Reproducibility requires that evidence pointers are resolvable.
- Evidence has structure — turn N, tool call M, policy event P — that matters for triage.

**Pattern.** Evidence references are **typed pointers** into structured artifacts (trace turn index, tool call identifier, policy event identifier), not opaque strings. The artifact is stored once; findings reference into it.

**Consequences.** Findings stay compact. Navigation from finding to evidence is mechanical. The cost is that artifact identity must be stable — pointers break if traces are mutated, which becomes an architectural constraint.

---

## Pattern 7: Policy as First-Class Input

**Context.** Many evaluation failures are policy violations, not quality problems.

**Problem.** When policy lives only in the agent's prompt or the organisation's wiki, evaluators can't test against it systematically. "Did this violate policy?" becomes a judgement call instead of an assertion.

**Forces.**
- Policy has structure (condition, action, exception, severity).
- Policy changes faster than agent code.
- Evaluators need policy to be machine-readable, not just human-readable.
- Free-text policy arrays are a placeholder, not a design.

**Pattern.** Express policy as a **structured profile** with rules that have explicit conditions, actions, and severities. The Context Bundle references the policy profile. Evaluators assert against policy as a first-class artifact, not as narrative.

**Consequences.** Policy regression becomes testable. New policies can be rolled out and tested before they reach production. The cost is that organisations must invest in expressing policy formally — a one-time cost that pays back indefinitely.

---

## Pattern 8: Sandboxed Tool Fidelity

**Context.** Agents call tools that have real-world side effects. Tests must not.

**Problem.** "Mock all the tools" produces an agent that behaves differently in test than in production. "Use real tools" produces tests that send emails to customers.

**Forces.**
- Tool side effects vary wildly (read-only query vs. irreversible email send).
- Fidelity matters: a mock that always returns success tests nothing.
- Some tools genuinely have no side effects and don't need mocking.
- Blast radius varies: deleting a test database record is not the same as deleting a customer's.

**Pattern.** Describe tool capabilities with **structured attributes** — reversibility, observability, blast radius, sandbox availability — rather than a single "side effect level." Evaluation mode negotiates which tools run real, which run mocked, and which are blocked entirely.

**Consequences.** Tests can be configured for the right fidelity per tool. Real side-effect tools get real attention. The cost is authoring: each tool needs honest capability metadata, and "honest" is hard.

---

## Pattern 9: Replay as Equal Citizen

**Context.** Agent evaluations are expensive (model calls, tool calls, time). Re-running them wastes effort and adds noise.

**Problem.** If replay is an afterthought, it tends not to work — trace formats lack the fidelity to reconstruct a run, and "replays" become approximations of the original.

**Forces.**
- Model non-determinism means re-running is not re-producing.
- Tool calls may not be idempotent.
- Evidence is most valuable when it can be re-examined without re-execution.
- Replay has different trust properties than live execution (you're trusting the recording, not the agent).

**Pattern.** Design the trace format for **deterministic replay** from the start. Treat replay as a first-class evaluation mode alongside live execution, with its own semantics and its own trust boundary.

**Consequences.** Findings can be re-examined cheaply. Bisection across agent versions becomes possible. The cost is that traces must capture enough to replay, which is more than traces usually capture.

---

## Pattern 10: Surface Parity Check

**Context.** The *Second Door* pattern creates two surfaces that must stay synchronised.

**Problem.** Over time, the evaluation surface drifts from the user surface. A bug is fixed on the user path and not the eval path, or vice versa. Tests pass on a surface that no longer represents production.

**Forces.**
- Two surfaces means two codepaths (or one codepath with branches).
- Drift is silent; only parity checks detect it.
- Parity checks are themselves tests that need maintenance.
- The cost of drift is delayed until a production incident.

**Pattern.** Include a **parity check** in the test suite: a small scenario that runs through both the user surface (headless browser, API-level) and the evaluation surface, and asserts behavioural equivalence. Run it in CI.

**Consequences.** Drift is caught early. The cost is explicit: you pay for parity in test time and maintenance, but you pay once, not per-bug.

---

## Pattern 11: Context Injection

**Context.** Agents behave as a function of their context — user profile, history, tenant configuration, retrieval results. Testing without realistic context tests only the degenerate case.

**Problem.** If the test harness supplies no context, the agent behaves as it would for a fresh anonymous user. If the harness pulls context from production, tests contaminate and are contaminated by real data. Neither is acceptable for systematic evaluation.

**Forces.**
- Agents need context to behave realistically.
- Production context has privacy, isolation, and reproducibility problems.
- Ad-hoc fake context drifts and lacks structure; evaluators invent different shapes for the same agent.
- Context has shape — user, history, environment, retrieval — that deserves naming.

**Pattern.** Treat context as an **injected dependency**, not an ambient one. The agent declares its injection points with names and schemas; the test harness supplies context through those points at session start. The agent MUST read context only from injected sources during evaluation, never from production databases, real user accounts, or environment variables.

**Consequences.** Tests are reproducible because context is explicit. Privacy is preserved because no production data is needed. The cost is authoring: agents that were quietly reading ambient state must be refactored to accept injection, and context bundles become required artifacts.

**Related:** *Context Bundle* — the Context Bundle describes what context *kinds* an agent consumes; Context Injection is the mechanism for supplying concrete context at test time.

---

## Pattern 12: Capability Registry

**Context.** Agents claim to do many things; evaluators ask them to do specific things. Without a shared vocabulary, the claim and the ask talk past each other.

**Problem.** Evaluators test in domains the agent was never designed for and generate noise findings. Agent teams receive findings that aren't their concern. Coverage gaps go undetected because nobody knows what "covered" means.

**Forces.**
- Agent scope is often implicit, living only in internal documentation.
- "General-purpose" is rarely true in practice; most agents have real boundaries.
- Coverage-by-domain and coverage-by-action are different questions and both matter.
- Free-text scope descriptions don't support automation.

**Pattern.** Make the agent publish a **capability registry** — a machine-readable declaration of domains (finance, healthcare, customer-service), actions (summarise, draft, refuse), and modalities (text, voice, image) the agent supports. Also declare `outOfScope` — the domains and actions the agent MUST refuse. Evaluators use the registry to scope their suites and detect coverage gaps.

**Consequences.** Evaluators know where to test and where not to. Findings from out-of-scope probes are filtered at submission. The cost is that agent teams must commit to a scope in machine-readable form, which is harder than waving at a PRD.

**Related:** *Context Bundle* carries the human-oriented purpose; the Capability Registry carries the machine-oriented scope. They agree where they overlap; the registry is normative where they disagree.

---

## How the patterns compose

```
Second Door ──┬── needs ──> Context Bundle
              ├── needs ──> Evaluation Mode Spectrum
              ├── needs ──> Context Injection
              └── protects ──> Surface Parity Check

Context Bundle ──┬── references ──> Policy as First-Class Input
                 ├── references ──> Sandboxed Tool Fidelity
                 └── aligns with ──> Capability Registry

Scenario ──┬── produces ──> Finding Over Pass/Fail
           ├── produces ──> Evidence Pointer, Not Evidence Copy
           └── scoped by ──> Capability Registry

Replay as Equal Citizen ──> applies to all scenarios
```

Not every agent needs every pattern. A small internal agent with no tool calls can skip *Sandboxed Tool Fidelity*. A deterministic agent can deprioritise *Replay*. But the patterns are designed to be individually useful — you can adopt three of them and get value; you are not signing up for the whole vocabulary.

## What this document is not

This is a pattern language, not a style guide. The patterns describe recurring shapes; they don't prescribe implementations. The [`spec/`](../spec) directory describes one way to implement the patterns for interoperability. Other implementations are welcome, and the patterns outlive any particular protocol.
