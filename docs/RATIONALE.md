# Design Rationale

This document records why AEP is shaped the way it is. It exists so that future maintainers — and reasonable critics — can tell intentional choices from accidents.

## Why normative, not descriptive

An earlier draft of v0.1 used advisory language — "servers should," "scenarios can" — treating the document as a guide. The result was a spec you couldn't disagree with because it didn't claim much. Interoperability requires that implementations agree on MUSTs, not on good intentions.

v0.1 as shipped uses RFC 2119 normative language (MUST / SHOULD / MUST NOT) throughout the wire protocol and the conformance checklist. The pattern language stays in descriptive voice — patterns describe recurring shapes, not obligations — but the protocol commits. A server either implements every MUST or it is not compliant.

## Why separate pattern language from wire protocol

Early drafts bundled concepts and methods into one document. It read as neither good patterns nor good protocol. The pattern language wanted examples, consequences, and anti-patterns; the protocol wanted rigour, error codes, and conformance hooks. Trying to deliver both in one artifact produced something too vague to implement and too detailed to browse.

Splitting them makes each serviceable on its own terms. The patterns are the durable contribution — they describe shapes that will outlive any specific wire format. The protocol is one implementation of the shapes, versioned and replaceable. Teams can adopt the vocabulary without adopting the methods, and that is a feature rather than a defect.

## Why an Agent Contract, not just a descriptor

The v0.1 predecessor had only an `AgentDescriptor` — identity and metadata. That was insufficient for interoperability. A descriptor tells you the agent's name; a contract tells you what you can send, what you'll receive, what context it consumes, and what modes it supports.

The `AgentContract` is the strict contract every compliant agent MUST publish. It contains the descriptor as a subcomponent but adds input schema, output schema, context model, capabilities registry, determinism declaration, endpoint map, limits, and traceability contract. This is the machinery that makes "AEP-compliant" a testable claim rather than an aspiration.

## Why REST discovery alongside JSON-RPC

The interactive protocol is JSON-RPC 2.0 — it matches MCP, supports capability negotiation cleanly, and handles batching. But JSON-RPC is uncomfortable for operator ergonomics: you can't curl an endpoint to ask "what agents are here?" without crafting a JSON-RPC envelope.

The spec adds REST read-only discovery (`GET /aep/agents`, `GET /aep/agents/{id}`) that returns the same data as the JSON-RPC discovery methods. Operators get a familiar surface; evaluators keep JSON-RPC for real work. No duplication in the wire protocol, just a thin HTTP alias.

## Why v0.1 and not v1.0

The first release is v0.1 because the wire protocol should be expected to evolve as implementations surface constraints the design didn't anticipate. Scoring, streaming, sandbox fidelity, and replay trace formats are all areas where we've deliberately left room. Calling this v1.0 would imply a stability claim the evidence doesn't yet support.

A v1.0 release will follow when the protocol has been exercised against multiple independent implementations and the remaining gaps have been closed. See [`ROADMAP.md`](./ROADMAP.md).

## Why capability negotiation over version comparison

Version comparison is brittle in a world where implementations evolve at different rates. A v0.2 server may share 90% of its capabilities with a v0.1 client; refusing to interoperate because the version numbers differ is overfitting to the label.

Capability negotiation lets the session operate at the intersection of what both sides support. Additive changes don't break clients that don't know about them. The cost is that implementations must declare capabilities honestly, but we consider that cost strictly preferable to silent incompatibility.

The spec keeps a `protocolVersion` field for major-break detection, but interoperability is governed by capabilities, not versions.

## Why a required session lifecycle

Evaluation without a lifecycle is evaluation without provenance. A session that was created, executed, evaluated, and reported is a session you can audit. A session that was "sort of run" produces findings you can't trace back to a known configuration.

The five-state lifecycle (created → active → evaluated → reported → closed) is required. Every transition is logged in an append-only array on the session record. The state machine is strict: methods invalid for the current state return an explicit error rather than succeeding silently. The cost is more protocol ceremony; the benefit is that every finding is attributable to a specific, reconstructable run.

## Why context injection, not ambient context

Agents that read ambient context (production databases, real user accounts, environment variables) during evaluation cannot be evaluated reproducibly. Context injection makes the context supply explicit: the agent declares what it consumes; the harness supplies it at session start.

This is a discipline cost on agent authors — code that was quietly calling `getCurrentUser()` must be refactored to accept a user parameter — but without it, tests are non-reproducible and privacy is unmanageable. The discipline cost is paid once; the reproducibility benefit compounds.

## Why the six-mode taxonomy

Previous frameworks used black-box/white-box or worse, a single "test" mode. The distinctions that actually matter in practice are about *what visibility the evaluator has* — user-surface only, plus context, plus policy events, plus tool calls, with or without realistic side effects, and replay as its own thing.

Naming six modes forces precise conversations. "We test in graybox" means something concrete. "We evaluate silently in policybox and promote to sandboxed-live at release gates" describes a real pipeline. Without the vocabulary, teams talk past each other about coverage.

The modes are ordered along a visibility ladder (blackbox ⊂ graybox ⊂ policybox ⊂ toolbox), with sandboxed-live sitting orthogonal (it adds realism, not visibility) and replay separate (it evaluates a recording, not a live agent). This structure is not accidental; it lets modes be granted incrementally and revoked cleanly.

## Why ContextBundle is a first-class artifact

Evaluators routinely produce findings that are noise to agent teams because the evaluator didn't know what the agent was for. The standard response — "read the PRD" — fails because PRDs are out of date, evaluators don't have access to them, and they contain too much to read for every test.

A ContextBundle is small, evaluator-oriented, and required as a precondition for evaluation. Making it first-class forces agent teams to articulate intent, non-goals, and boundaries in a form that evaluators can actually consume. The cost is authoring; the benefit is that evaluation findings become precise because they reference explicit boundaries.

The schema makes `nonGoals` required for exactly this reason. "What this agent is NOT supposed to do" is where most evaluation-relevant boundaries live, and it is the field teams most often leave empty.

## Why tool capabilities got more structure

An early draft had a single `sideEffectLevel` enum: `none | read | write | external`. This collapsed distinctions that matter in evaluation contexts. A tool that writes to a sandboxed database is not the same as one that sends email to real customers, and both are "write." A tool that can be un-done within a minute is not the same as one whose effects are permanent, and both are "external."

The replacement — reversibility, observability, blast radius, sandbox fidelity — is more verbose but carries the information evaluators actually need. Irreversible tools with external blast radius should not run in sandboxed-live mode unless sandbox fidelity is `real-isolated`; the structure makes this check mechanical.

## Why policy is structured, not a string array

The earlier draft had four arrays of strings: safety policies, privacy policies, tool-use rules, escalation rules. That was a placeholder, not a design.

Real policy has structure: a rule applies under a condition, triggers an action, admits of exceptions, and has a severity. Assertions against policy need machine-readable form; narrative descriptions make it impossible to test whether a given response honoured a given rule.

The `PolicyProfile` schema models rules with these components. It deliberately allows arbitrary matcher shapes in the `condition.matcher` field — we don't know the right matcher language yet and don't want to commit to one prematurely. Implementations can define their own matcher schemas; the protocol carries the envelope.

The `testable` flag acknowledges that some policies are aspirational or human-judgement-only. Flagging them prevents false-negative findings from scenarios that assert against rules the protocol cannot verify.

## Why scenarios support branching

Linear message lists are fine for smoke tests and simple regressions. They are inadequate for adversarial testing, which is fundamentally about "what happens after the agent fails the first challenge."

The scenario schema supports conditional branching (`branches[].when`) and state that carries across turns (`stateUpdates`). Linear scenarios are the degenerate case, not the norm.

We also added an `adaptive` input type that defers the actual prompt to an attacker model given the prior context. This is necessary for realistic red-team evaluation — the attacker has to adapt, and baking the adaptation into the scenario schema makes adaptive attacks first-class rather than shoehorned.

## Why evidence references are typed

An opaque string reference like `"evidence-1234"` is a forensic burden. Evaluators receive a finding and have to navigate manually to the underlying artifact, often without knowing what kind of artifact it is (turn? tool call? policy event?).

Typed references (`trace-turn` with `turnIndex`, `tool-call` with `toolCallId`, `policy-event` with `policyEventId`, `external` with `uri`) make navigation mechanical. A tester UI can render each type appropriately. Automated aggregation can count by evidence type. The cost is modest schema complexity; the benefit compounds with every finding.

## Why scoring is deliberately underspecified

Scoring has at least five distinct concerns — deterministic assertions, LLM-as-judge, human review, regression-against-baseline, and aggregate metrics. A single unified scoring method would collapse them unhelpfully.

The spec declines to unify them. Implementations may expose scoring through the protocol, but v0.1 does not specify semantics, and [`SCORING.md`](./SCORING.md) documents the concerns that future versions will need to address separately.

This is not a gap; it is a decision. Premature unification of scoring is how evaluation platforms become generic tools that do every kind of scoring badly. Better to leave the surface explicit and let implementations differentiate.

## Why replay is a first-class mode

Most observability systems treat replay as an afterthought. The trace format is designed for debugging, not reconstruction, and "replaying" a trace produces an approximation of the original run.

AEP names replay as one of the six modes because evaluators routinely want to re-examine runs without re-executing them. Model non-determinism means re-running is not re-producing; the only way to make findings re-examinable is to capture enough at run time to replay deterministically.

The v0.1 spec names replay but does not yet specify the trace format that makes it deterministic. This is an acknowledged gap and an explicit v0.2 priority. Naming the mode now is a commitment to treat the gap seriously rather than discover it after the trace format has already been frozen.

## Why the spec has an "Open Questions" section

Specifications that pretend to be complete when they aren't set implementers up for unpleasant surprises. Section 17 of the spec lists what we know we haven't yet solved — streaming, scoring, branching, sandbox fidelity, replay format, multi-agent, cost attribution — so implementers know where the ice is thin.

Listing open questions also tends to attract the right feedback. Reviewers engage more productively with "here's what we haven't figured out" than with "here's what we've solved." The former invites contribution; the latter invites contradiction.

## What this document is not

This is not a changelog — see `spec/CHANGELOG.md` for that. It is not a FAQ — questions that recur from users will eventually become one, but they haven't yet. It is a record of choices made deliberately, so that reversing them happens on purpose.
