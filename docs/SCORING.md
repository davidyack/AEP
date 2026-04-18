# On Scoring

*Why the AEP spec does not prescribe a scoring method, and what future versions will have to address.*

---

Scoring is where evaluation platforms earn or lose their reputation. A spec that unifies it prematurely makes every implementer's work harder; a spec that ignores it entirely leaves the most important question unanswered. This document takes a middle path: it describes the concerns that scoring must address, without committing the protocol to a single answer.

There are at least five distinct scoring concerns. They interact but they do not collapse.

## 1. Deterministic assertions

**What it is.** Mechanical checks: did the output contain X, avoid Y, match regex Z, invoke tool T, trigger policy P? Boolean or countable.

**What it's good for.** CI gates, regression prevention, smoke tests. Fast, reproducible, cheap to evaluate, easy to explain.

**What it isn't good for.** Anything involving judgement. "Is this response appropriate" cannot be deterministically asserted.

**How AEP accommodates it.** The `Scenario.turns[].assertions` field supports deterministic types (`contains`, `not-contains`, `matches-regex`, `invokes-tool`, `does-not-invoke-tool`, `triggers-policy`, `does-not-trigger-policy`). The `judge` type is reserved for concern #2.

## 2. LLM-as-judge

**What it is.** A second model (the "judge") evaluates the agent's output against a rubric. Produces a graded score, a categorical judgement, or both.

**What it's good for.** Subjective quality, appropriateness, tone, reasoning quality. Scales better than human review.

**What it isn't good for.** Anything where the judge and the agent share the same failure modes — bias, blind spots, hallucination tendencies. Judges calibrated once and never again drift along with the underlying models.

**What makes it hard.** Prompt sensitivity (small rubric changes produce large score changes), non-determinism (the judge isn't deterministic either), and calibration (you need judge-human agreement data to know if the judge is trustworthy).

**How AEP accommodates it.** The `judge` assertion type carries a rubric pointer; implementations supply the judge. The protocol does not mandate a judge model, prompt, or scoring scale. Calibration is an implementation concern that the protocol should not prematurely dictate.

## 3. Human review

**What it is.** A person reads the trace and produces findings.

**What it's good for.** Cases where the judgement is genuinely hard, where the stakes are high enough to justify the cost, or where the population being reviewed is small.

**What it isn't good for.** Scale. Also: two reviewers often disagree, and without inter-rater agreement data, "human review" produces findings of unknown quality.

**What makes it hard.** Queueing, triage, assignment, agreement tracking, reviewer fatigue. These are product concerns, not protocol concerns.

**How AEP accommodates it.** Findings carry a `createdBy` field and a `triage` substructure. Human-generated findings flow through the same schema as automated ones, so aggregation and tracking work uniformly. The queueing and assignment machinery is explicitly out of scope.

## 4. Regression against baseline

**What it is.** Comparing current behavior against a prior known-good state. Did anything get worse? What specifically?

**What it's good for.** Release gates, post-deployment monitoring, bisecting which change broke which behavior.

**What it isn't good for.** Validating that the baseline itself was good. You can have a stable regression of mediocre behavior.

**What makes it hard.** Choosing a baseline (which version? which date? which configuration?), deciding what "worse" means (fewer passes? more severe findings? both weighted how?), and handling the cases where the baseline's behavior itself was wrong.

**How AEP accommodates it.** Traces are immutable and version-attributed (`agentVersion`), which is the minimum needed to make regression possible. The `replay` mode allows old traces to be re-examined under new rubrics. Baseline management and comparison semantics are implementation concerns.

## 5. Aggregate metrics

**What it is.** Summary statistics over many runs. Win rates, pass rates, severity distributions, latency percentiles, cost per scenario.

**What it's good for.** Dashboard consumption, trend detection, executive reporting. Compressing thousands of findings into a tractable picture.

**What it isn't good for.** Triage. An aggregate metric with no drill-down is useless for fixing anything.

**What makes it hard.** Weighting (does a critical finding count as 1 or 100 findings?), normalization (are percentages over runs, over scenarios, or over turns?), and survivor bias (aggregates over only the scenarios you chose to run).

**How AEP accommodates it.** Aggregation is derived from findings; the Finding schema supports the fields needed (severity, category, agent version, scenario identity). The protocol does not prescribe aggregation methods.

## How the concerns compose

These concerns overlap in real evaluation pipelines. A typical CI run might combine:

- Deterministic assertions as the base layer (fast, cheap, required to pass)
- LLM-as-judge over anything marked as quality-relevant (slower, calibrated against a sample)
- Regression against baseline applied to both (halt if anything got worse)
- Aggregate metrics published to a dashboard
- Human review sampled on, say, 5% of findings, for ongoing judge calibration

A unified "scoring method" in the protocol would have to abstract over all of these. The abstraction would be either so thin it's useless (passing a rubric through to an implementation) or so thick it constrains the implementations unhelpfully. We have chosen to make the surface explicit instead.

## Implementation guidance

If you are implementing AEP and choosing a scoring stack, the following are opinions drawn from the field, not requirements:

**Default to deterministic for anything you can express deterministically.** It's faster, cheaper, and easier to debug. Reach for LLM-as-judge only where deterministic assertions can't express the question.

**Calibrate your judges.** Before relying on LLM-as-judge scores, get a sample of human judgements on the same inputs and measure agreement. A judge with 60% human agreement is worse than no judge — it gives false confidence.

**Version your rubrics.** A rubric is software. Regression-against-baseline assumes the rubric is stable; if it isn't, you can't tell whether the agent changed or the yardstick did. Track rubric versions alongside agent versions.

**Separate pass/fail from severity.** Traditional test frameworks conflate "did the assertion fail" with "how bad is it." Agent evaluation wants these separate. Use Finding severity.

**Aggregate from findings, not alongside them.** Don't run parallel aggregation pipelines. Every dashboard number should be derivable from the underlying findings. If it isn't, you've lost traceability.

## When v0.2 might specify scoring

The protocol will specify scoring if and only if we see concrete evidence that cross-implementation interoperability is being held back by the absence. Signs to watch for:

- Implementers building incompatible ad-hoc scoring APIs that users want to bridge
- A de facto community convention emerging that's worth codifying
- A specific scoring concern where the lack of a common format is causing findings to be non-portable

Until we see those signs, premature specification is the greater risk.

## Further reading

The concerns and counterexamples above are informed by — but not dependent on — published work in LLM evaluation, classical software testing, and human-reviewer methodology. We have resisted citing specific frameworks to avoid this document becoming obsolete when they do. The concerns themselves should remain stable even as the implementations churn.
