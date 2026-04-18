# Six Modes of Agent Evaluation

*A taxonomy for describing what your test suite can actually see.*

---

Agent evaluation is not black-box versus white-box. Between those poles are a handful of useful intermediate stops, each corresponding to a different set of questions you can meaningfully ask. Giving these stops names lets teams have precise conversations about coverage, security posture, and reproducibility.

This document names six modes. They are not exhaustive and they are not orthogonal — a given evaluation may combine modes — but in our experience they cover the territory worth covering.

---

## 1. Blackbox

**What's visible:** only what a user would see. Inputs go in, outputs come out.

**Good for:** user-experience testing, surface-level quality checks, sanity tests, acceptance gates.

**Limitations:** you can't tell *why* something happened. A correct output from a broken reasoning process looks identical to a correct output from a sound one.

**Typical question:** "Does the agent produce the right answer to this question?"

---

## 2. Graybox

**What's visible:** blackbox plus the Context Bundle — intended use, non-goals, behavioural boundaries, tool inventory, policy summary.

**Good for:** the default mode for most quality and behaviour testing. Evaluators have enough context to judge whether an output is appropriate, not just whether it's correct.

**Limitations:** no visibility into internal events. Policy violations that don't surface to the user are invisible.

**Typical question:** "Is this output appropriate given what the agent is supposed to be for?"

---

## 3. Policybox

**What's visible:** graybox plus structured policy events — which rules were evaluated, which triggered, what actions were taken.

**Good for:** safety testing, RAI validation, compliance checks, policy-regression testing.

**Limitations:** still doesn't expose tool-call details or intermediate reasoning. Policy events that silently pass leave no trace.

**Typical question:** "Did the agent's behaviour honour the policies it's supposed to be bound by?"

---

## 4. Toolbox

**What's visible:** policybox plus tool call visibility — what tools were invoked, with what arguments, returning what values.

**Good for:** debugging tool-use failures, testing tool-selection quality, validating that agents call the right tools with the right arguments.

**Limitations:** tool calls may include real side effects unless combined with sandboxing. Model-internal reasoning still opaque.

**Typical question:** "Did the agent use its tools correctly, and did the tools behave as expected?"

---

## 5. Sandboxed-Live

**What's visible:** toolbox, but with real tool integrations that operate in a safe environment (test databases, mock external APIs with realistic behaviour, rate-limited real calls where appropriate).

**Good for:** end-to-end testing with production-like fidelity. The agent exercises its real tool integrations without real-world consequences.

**Limitations:** sandbox fidelity is always imperfect. Some failures only manifest against real production systems.

**Typical question:** "Does the agent work when the plumbing actually runs?"

---

## 6. Replay

**What's visible:** whatever was captured in a prior trace. The agent is not executed; its recorded behaviour is re-examined.

**Good for:** post-hoc analysis, bisection across agent versions, re-scoring against new rubrics, cheap regression checks.

**Limitations:** you are evaluating a recording, not the current agent. If the agent has changed, replay tells you what *used to* happen. Replay also cannot catch issues that depend on live state.

**Typical question:** "What exactly happened on this run, and would a different rubric catch anything we missed?"

---

## How to think about the modes

Two useful framings:

**As a visibility ladder.** Each mode strictly adds to the one before it (blackbox ⊂ graybox ⊂ policybox ⊂ toolbox). Sandboxed-live sits orthogonal to the ladder (it adds realism rather than visibility). Replay is its own thing.

**As a trust-boundary spectrum.** Blackbox asks the least of the agent team and grants the evaluator the least. Toolbox asks the most and grants the most. Evaluation environments should grant the minimum necessary mode.

## Using the modes in practice

- **Default to graybox** for most testing. Pure blackbox produces findings that are hard to interpret; anything more than graybox should be granted deliberately.
- **Use policybox for RAI and safety suites.** Silent policy violations are exactly the thing you need to catch, and only policybox surfaces them.
- **Use toolbox for tool-use regression.** Tool-selection quality is a real failure mode and hard to debug without visibility.
- **Use sandboxed-live for release gates.** Pre-production confidence benefits from real plumbing, not just mocks.
- **Use replay for post-incident analysis.** Don't re-run the scenario to reproduce a bug; replay the trace.

## What the modes are not

These are not security classifications. "Blackbox" does not mean "safe to expose to anyone"; all evaluation modes should require authentication and isolation. The modes describe *what you can see*, not *who should be allowed to see it*.

They are also not roles. An evaluator is not a "graybox evaluator"; they run sessions in the mode appropriate to the question at hand.

---

*This taxonomy was extracted from the broader Agent Evaluation Protocol (AEP) work because it is useful on its own terms. You can adopt the vocabulary without adopting any protocol. If you do use it, we'd appreciate attribution — but more than that, we'd appreciate hearing what modes we missed.*
