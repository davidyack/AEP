# Composite Session Evaluation (CSE) Extension

*Evaluating multi-agent systems as a single session.*

**Extension ID:** `cse` · **Core hooks:** spec §11.7 · **Status:** v0.1 draft companion

---

This is the companion document that spec §11.7 points to. The core spec defines five hooks (graph on sessions, nodeId on trace turns, compositeCapable on contracts, oneOf constraint on session creation, and the four requirements AEP-REQ-109 through 112). This document defines the full extension semantics that sit on top of those hooks: graph structure, handoff events, evaluation targets, and the scoring dimensions that make multi-agent evaluation work.

## Why CSE exists

Real agent systems are increasingly composed: a router dispatches to specialists, an orchestrator sequences tool-using agents, a supervisor escalates to a human on edge cases. Evaluating such systems one-agent-at-a-time misses the interesting failure modes:

- **Wrong routing.** Router picks Specialist B when Specialist C was right.
- **Context loss at handoff.** Agent A learned the user is a premium customer; Agent B doesn't know.
- **Correct locally, wrong globally.** Every agent does its job individually, but the joint outcome is wrong.
- **Compounding delays.** Each hop adds latency; the user-visible SLA breaks despite every individual agent meeting its own.

Single-agent evaluation catches none of these. CSE does.

## Core concepts

### Session graph

A composite session is a directed graph:

- **Nodes** — spans of execution where one specific agent is the active responder
- **Edges** — handoffs between agents, carrying an explicit payload and reason
- **Root** — the first node to receive user or system input
- **Terminal** — a node that produces a final response with no outgoing edges (or an explicit `end` marker)

```
    root
      │
      ▼
  ┌─ router ─┐
  │          │
  ▼          ▼
billing    support
  │          │
  └─── human ─┘   (terminal)
```

### Graph constraints (v0.1)

- **Rooted DAGs, no cycles.** Agent A may hand to B; B may not hand back to A on the same session. Covers nearly all real orchestration; keeps semantics simple.
- **Single active node at a time.** Parallel fan-out is deferred to CSE v2.
- **Explicit termination.** A composite session completes when execution reaches a terminal node (no outgoing edges) or when an agent emits an explicit `end` handoff to a synthetic terminal.
- **Human handoff terminates the graph.** Escalation to a human is a terminal edge; CSE does not model the human side.

### Granularity: node = sub-session

A **node** represents a period where one named agent is the active responder. Internally, a node contains one or more turns (request/response pairs). Handoffs happen at node boundaries, not turn boundaries within a node.

This means:
- Agent B handing to Agent C is a node boundary (edge emitted)
- Agent B continuing its own conversation across multiple turns stays within one node
- A sub-agent invocation that returns control immediately is modelled as a tool call within a node, not a new node

The test: did the active-agent identity change? If yes, it's a new node.

### Handoff as a first-class event

Every edge in the graph corresponds to a **handoff event** with an explicit payload:

```json
{
  "handoffId": "ho_001",
  "fromNode": "router",
  "toNode": "billing-specialist",
  "reason": "classified as billing question",
  "contextTransferred": {
    "userIntent": "refund request for order #123",
    "userProfile": { "tier": "premium" },
    "conversationSummary": "..."
  },
  "at": "2026-04-17T12:01:30Z"
}
```

Handoffs are evaluatable artifacts. Routing quality, handoff fidelity, and context preservation all score against them.

## The graph schema

Session creation with a composite graph:

```json
POST /aep/sessions
{
  "testMode": "graybox",
  "graph": {
    "nodes": [
      { "id": "router", "agentId": "example.router", "role": "router" },
      { "id": "billing", "agentId": "example.billing-specialist", "role": "specialist" },
      { "id": "support", "agentId": "example.support-specialist", "role": "specialist" },
      { "id": "human", "role": "terminal", "terminal": true }
    ],
    "edges": [
      { "id": "e1", "from": "router", "to": "billing", "condition": "classification == 'billing'" },
      { "id": "e2", "from": "router", "to": "support", "condition": "classification == 'support'" },
      { "id": "e3", "from": "billing", "to": "human", "condition": "escalate == true" },
      { "id": "e4", "from": "support", "to": "human", "condition": "escalate == true" }
    ],
    "root": "router"
  },
  "injectedContext": { "user": { ... } },
  "ttlSeconds": 900
}
```

Each node references an AEP-compliant agent by `agentId`. All referenced agents MUST have `compositeCapable: true` (core AEP-REQ-112).

### Node fields

| Field | Purpose |
|-------|---------|
| `id` | Unique within the graph. Used in handoff events and trace turns. |
| `agentId` | The AEP agent active at this node. Required except for terminal/synthetic nodes. |
| `agentVersion` | Optional version pin. If omitted, uses the agent's current version. |
| `role` | Free-form label (`router`, `specialist`, `supervisor`, `terminal`). Used for scoring targets and diagnostics. |
| `terminal` | Boolean; true if this node has no outgoing edges and represents completion. |

### Edge fields

| Field | Purpose |
|-------|---------|
| `id` | Unique within the graph. |
| `from`, `to` | Node IDs. |
| `condition` | Human-readable description of when the edge fires. Not evaluated by the protocol — informational. |
| `expectedPayloadSchema` | Optional JSON Schema the handoff payload MUST match when this edge fires. Used for validating handoff correctness. |

## Handoff authority

**Default: only the explicit handoff payload is visible to the receiving node.** Agent B sees what Agent A explicitly transferred. It does NOT see:

- Agent A's internal reasoning
- Agent A's tool call arguments or results
- Agent A's raw turn inputs and outputs
- The user's original message unless Agent A passed it through

This matches real orchestration patterns and keeps node boundaries clean. Nodes that need more context need it made explicit.

Servers MAY support a non-default authority level via node configuration (e.g. `authority: "full-history"`), but this is opt-in per deployment and not the spec default.

## Evaluation targets

CSE evaluation scores against four target levels:

### Node target
*"Did this specific agent do its job?"*

Identical to single-agent evaluation, scoped to the turns within one node. Scoring rubrics written for single-agent use of that agent port unchanged.

```yaml
target: node
nodeId: billing-specialist
rubric: billing_accuracy
```

### Edge target
*"Was this handoff correct?"*

Scores the edge event. Two sub-dimensions:

- **Routing correctness** — given the node's output, was this the right edge to fire?
- **Payload fidelity** — did the handoff transfer the necessary context?

```yaml
target: edge
fromNode: router
toNode: billing-specialist
rubrics:
  - routing_correctness
  - payload_fidelity
```

### Path target
*"Was this a reasonable path through the graph?"*

Scores a sequence of nodes and edges. Catches cases where every individual hop looked fine but the overall journey was wrong (e.g. "user asked a simple question and got bounced through four specialists").

```yaml
target: path
startNode: router
endNode: human
rubric: path_efficiency
```

### Graph target
*"Was the overall outcome correct?"*

End-to-end evaluation from root to terminal. The composite system's user-visible result is evaluated as a whole. Typically the most important target for customer-impact scoring.

```yaml
target: graph
rubric: end_to_end_user_satisfaction
```

## Scoring dimensions

CSE populates the existing `EvaluationResult.scores` map with CSE-specific keys. These join (not replace) any single-agent score keys.

| Key | Target | What it measures |
|-----|--------|------------------|
| `routing_quality` | edge | Fraction of routing decisions that match the target routing for the input |
| `handoff_fidelity` | edge | Quality of context preservation across handoffs |
| `node_quality.{nodeId}` | node | Per-node score, using the node agent's normal quality rubric |
| `path_efficiency` | path | Whether the traversed path was reasonable given alternatives |
| `end_to_end_outcome` | graph | Whether the final response met user intent |
| `compounding_latency` | graph | Total wall-clock latency across all hops |

Servers populate these as numeric scores or categorical outcomes; consumers ignore keys they don't recognise.

## CSE stream events

When the Streaming extension (§11.5) is also supported, CSE adds these event types to the registered set:

| Event | Emitted when |
|-------|--------------|
| `node.entered` | Execution enters a node (the node becomes the active agent) |
| `node.exited` | Execution leaves a node (immediately before or after an edge) |
| `handoff.initiated` | Active node starts a handoff (payload prepared, not yet transferred) |
| `handoff.completed` | Handoff payload delivered; target node is now active |
| `graph.terminal-reached` | Execution reaches a terminal node |

Payload shapes follow the same conventions as the base streaming events: each event carries its sequence `id`, the relevant `nodeId` or `handoffId`, and type-specific details.

## CSE error codes

CSE reserves the range `-32080` through `-32089`:

| Code | Name | Meaning |
|------|------|---------|
| -32080 | `graph_invalid` | Graph structure violates CSE rules (e.g. cycle detected, unreachable node) |
| -32081 | `node_agent_incompatible` | A graph node references an agent with `compositeCapable: false` |
| -32082 | `handoff_payload_invalid` | Handoff payload failed the edge's `expectedPayloadSchema` |
| -32083 | `no_terminal_reached` | Graph completed without reaching a terminal (stuck state) |
| -32084 | `unauthorized_edge` | Attempted handoff along an edge the current node did not declare |

## Termination semantics

A composite session completes when:

1. Execution reaches a node with `terminal: true`, OR
2. A node explicitly emits an `end` handoff to a synthetic terminal, OR
3. The session times out (normal AEP TTL behavior)

Termination condition (3) produces a `graph_invalid` finding if no terminal was reachable — this usually indicates a broken graph definition.

## Partial evaluation during streaming

With both CSE and Streaming active, the partial-scoring rules from AEP-REQ-017 apply. Node-target scoring can run as each `node.exited` event fires. Edge-target scoring can run on `handoff.completed`. Graph-target scoring MUST wait for `graph.terminal-reached`.

## What's NOT in CSE v0.1

Explicitly deferred to CSE v2:

- **Cycles.** Agent A ⇄ Agent B conversational loops.
- **Parallel fan-out.** Router dispatching to three specialists whose results are joined.
- **Counterfactual scoring.** "Would the other routing choice have been better?"
- **Cross-session graphs.** Long-running graphs that span multiple user sessions.
- **Dynamic graph construction.** Graphs whose structure is determined at runtime based on node outputs.

These are real needs but require design work beyond v0.1. The v0.1 shape is deliberately the subset that covers roughly 80% of production orchestration patterns.

## Relationship to the core

CSE is a strict extension: no CSE feature breaks a single-agent session. The five core hooks (AEP-REQ-109–112) are additive — agentId-only sessions behave identically whether or not a server advertises CSE support. Implementers can build single-agent AEP first and add CSE later without breaking anything.

## Implementation checklist

For servers adding CSE support:

- [ ] Advertise `"cse"` in `capabilities.extensions` at `aep.initialize`
- [ ] Accept `graph` field on `POST /aep/sessions` (mutually exclusive with `agentId`)
- [ ] Validate graph structure (DAG, connected, has root, reachable terminals)
- [ ] Enforce `compositeCapable: true` on all referenced agents
- [ ] Populate `nodeId` on every Trace turn
- [ ] Emit handoff events in the trace (and via streaming if supported)
- [ ] Implement node/edge/path/graph evaluation targets
- [ ] Populate CSE score dimensions in EvaluationResult
- [ ] Return CSE error codes appropriately

## Relationship to other extensions

- **Policy Profile (§11.1)** — policies apply per-node using the node agent's policy profile. No cross-node policy semantics in v0.1.
- **Tool Capability (§11.2)** — tool capabilities are per-agent; in a composite session each node operates under its agent's tool permissions.
- **Red-teaming (§11.3)** — composite sessions can run in red-team mode; adversarial scenarios can target handoffs as attack surfaces (e.g. context-poisoning during handoff is a recognised attack category).
- **Deterministic Replay (§11.4)** — composite sessions replay node-by-node. A composite session is replayable only if every participating agent is replayable.
- **Streaming (§11.5)** — CSE adds the five events listed above to the streaming registry.
- **Dataset (§11.6)** — datasets MAY declare `"compositeAgentGraph": "..."` in metadata for datasets targeting composite systems; each example exercises the whole graph rather than one agent.
