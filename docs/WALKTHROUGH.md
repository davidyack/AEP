# AEP Walkthrough

*A single end-to-end example from discovery to evaluation result.*

---

This document walks one complete evaluation from the first request to the final findings. If you read only one AEP document before implementing, read this one. The specification is the authoritative source; this walkthrough is illustrative.

We'll evaluate a fictional agent (`example.inquiry-assistant`) running one scenario (`quality-probes-assumptions.json`) in `graybox` mode.

Two parallel tracks are shown throughout: the **REST surface** on the left, the **JSON-RPC equivalent** on the right. Both return the same data. Pick whichever matches your tooling.

---

## Step 0 — Authenticate

Every AEP request requires authentication (AEP-REQ-031). The examples use a bearer token; real deployments might use mTLS or workload identity.

```bash
export AEP_TOKEN="eyJhbGciOi..."   # short-lived, ≤1 hour
```

---

## Step 1 — Discover available agents

You want to see what agents this server exposes and which ones you're authorised to test.

**REST**

```bash
curl -H "Authorization: Bearer $AEP_TOKEN" \
     "https://eval.example.com/aep/agents?limit=25"
```

**JSON-RPC**

```bash
curl -H "Authorization: Bearer $AEP_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "jsonrpc": "2.0",
       "id": 1,
       "method": "aep.agent.list",
       "params": {"limit": 25}
     }' \
     https://eval.example.com/aep/rpc
```

**Response** (both surfaces return the same data — summaries, not full contracts):

```json
{
  "agents": [
    {
      "agentId": "example.inquiry-assistant",
      "version": "1.2.0",
      "name": "Inquiry Assistant",
      "description": "Helps PMs sharpen vague requests.",
      "riskTier": { "level": "medium" },
      "lifecycle": { "stage": "staging" },
      "domains": ["product-management", "requirements-elicitation"],
      "modalities": ["text"],
      "supportedModes": ["blackbox", "graybox", "policybox", "toolbox", "replay"],
      "authorizedModes": ["blackbox", "graybox", "policybox"],
      "supportedExtensions": ["policy-profile", "deterministic-replay"],
      "lastUpdatedAt": "2026-04-15T00:00:00Z"
    }
  ],
  "nextCursor": null,
  "total": 1
}
```

Note `authorizedModes` vs `supportedModes`: this agent supports `toolbox` but your credentials don't cover that mode, so you'll see it in the list but cannot create toolbox sessions against it. Unauthorised agents are omitted entirely rather than shown here.

### Filtering and pagination

For servers with many agents, filter the list to what matters:

```bash
# All red-team-capable agents in customer-service domain
curl "https://eval.example.com/aep/agents?domain=customer-service&extension=red-teaming"

# Only high-risk agents in staging
curl "https://eval.example.com/aep/agents?risk=high,critical&lifecycle=staging"

# Agents updated since my last sync (incremental discovery)
curl "https://eval.example.com/aep/agents?updatedSince=2026-04-01T00:00:00Z"
```

Pagination is cursor-based: use `nextCursor` from one response as `?cursor=` on the next.
```

---

## Step 2 — Retrieve the full Agent Contract

Before submitting evaluation, you need to know the agent's input schema, output schema, and context injection points.

**REST**

```bash
curl -H "Authorization: Bearer $AEP_TOKEN" \
     https://eval.example.com/aep/agents/example.inquiry-assistant
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 2, "method": "aep.agent.get",
  "params": {"agentId": "example.inquiry-assistant"} }
```

**Response** (abbreviated):

```json
{
  "aepVersion": "0.1",
  "agentId": "example.inquiry-assistant",
  "version": "1.2.0",
  "descriptor": { "..." : "..." },
  "inputSchema": {
    "type": "object",
    "required": ["message"],
    "properties": {
      "message": {"type": "string", "minLength": 1, "maxLength": 8000}
    }
  },
  "outputSchema": {
    "type": "object",
    "required": ["response"],
    "properties": { "response": {"type": "string"} }
  },
  "contextModel": {
    "injectionPoints": [
      { "name": "user", "type": "user", "required": true, "schema": { "..." : "..." } },
      { "name": "history", "type": "history", "required": false, "schema": { "..." : "..." } },
      { "name": "environment", "type": "environment", "required": false, "schema": { "..." : "..." } }
    ],
    "historyModel": "linear",
    "maxHistoryTurns": 50
  },
  "evaluationModes": ["blackbox", "graybox", "policybox", "toolbox", "replay"],
  "limits": { "turnTimeoutSeconds": 60, "maxTurnsPerSession": 50 }
}
```

You'll validate your inputs against `inputSchema` client-side before submission (AEP-REQ-006).

---

## Step 3 — Start a session

Sessions bind a specific agent version and carry the injected context. You pick the evaluation mode based on what you need to observe — here we use `graybox`, which gives access to the Context Bundle but not tool calls or policy events.

**REST**

```bash
curl -X POST -H "Authorization: Bearer $AEP_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "agentId": "example.inquiry-assistant",
       "testMode": "graybox",
       "injectedContext": {
         "user": { "role": "product-manager", "tenureMonths": 18, "priorInteractions": 5 },
         "history": [],
         "environment": { "locale": "en-US", "timezone": "America/Denver" }
       },
       "ttlSeconds": 900
     }' \
     https://eval.example.com/aep/sessions
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 3, "method": "aep.session.start",
  "params": {
    "agentId": "example.inquiry-assistant",
    "testMode": "graybox",
    "injectedContext": { "...": "..." },
    "ttlSeconds": 900
  }
}
```

**Response**:

```json
{
  "sessionId": "sess_a1b2c3d4",
  "agentId": "example.inquiry-assistant",
  "agentVersion": "1.2.0",
  "testMode": "graybox",
  "state": "active",
  "createdAt": "2026-04-17T12:00:00Z",
  "expiresAt": "2026-04-17T12:15:00Z"
}
```

The session is now `active`. You can run turns or scenarios.

---

## Step 4 — Run a scenario

You could execute turns one at a time via `POST /aep/sessions/{id}/turns`, but scenarios bundle a complete test case — inputs, assertions, failure conditions — in a reusable artifact. We'll run `quality-probes-assumptions.json`.

**REST**

```bash
curl -X POST -H "Authorization: Bearer $AEP_TOKEN" \
     -H "Content-Type: application/json" \
     -d @examples/reference-agent/scenarios/quality-probes-assumptions.json \
     https://eval.example.com/aep/sessions/sess_a1b2c3d4/scenarios
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 4, "method": "aep.scenario.run",
  "params": { "sessionId": "sess_a1b2c3d4", "scenario": { "...": "..." } }
}
```

**Response**:

```json
{
  "scenarioId": "reference.quality.probes_assumptions",
  "runId": "run_e5f6g7",
  "status": "completed",
  "turnsExecuted": 1,
  "turnRefs": ["turn_001"],
  "assertionResults": [
    { "type": "does-not-invoke-tool", "passed": true },
    { "type": "judge", "passed": true, "score": 0.82 },
    { "type": "not-contains", "passed": true }
  ]
}
```

The scenario completed, all assertions passed. The Trace is being built.

---

## Step 5 — Evaluate

Transition the session from `active` to `evaluated`. This triggers the scoring engine to produce an EvaluationResult from the run.

**REST**

```bash
curl -X POST -H "Authorization: Bearer $AEP_TOKEN" \
     https://eval.example.com/aep/sessions/sess_a1b2c3d4/evaluate
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 5, "method": "aep.session.evaluate",
  "params": { "sessionId": "sess_a1b2c3d4" } }
```

**Response** (synchronous scoring):

```json
{
  "sessionId": "sess_a1b2c3d4",
  "state": "evaluated",
  "resultRef": "result_h8i9j0"
}
```

For servers that advertise `asyncScoring: true` in their capabilities, the response returns immediately with `status: "pending"` and you poll `GET /aep/results/{resultRef}` until `status: "ready"`.

---

## Step 6 — Report (seal the result)

The evaluated state means scoring is done; the reported state means the result is sealed and ready for consumption. This step makes the result immutable.

**REST**

```bash
curl -X POST -H "Authorization: Bearer $AEP_TOKEN" \
     https://eval.example.com/aep/sessions/sess_a1b2c3d4/report
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 6, "method": "aep.session.report",
  "params": { "sessionId": "sess_a1b2c3d4" } }
```

---

## Step 7 — Retrieve the result

**REST**

```bash
curl -H "Authorization: Bearer $AEP_TOKEN" \
     https://eval.example.com/aep/results/result_h8i9j0
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 7, "method": "aep.result.get",
  "params": { "resultRef": "result_h8i9j0" } }
```

**Response**:

```json
{
  "resultId": "result_h8i9j0",
  "sessionId": "sess_a1b2c3d4",
  "agentId": "example.inquiry-assistant",
  "agentVersion": "1.2.0",
  "testMode": "graybox",
  "scores": {
    "quality": 0.82,
    "safety": 1.0,
    "grounding": null
  },
  "violations": [],
  "outcome": "passed",
  "metrics": {
    "turnsExecuted": 1,
    "totalLatencyMs": 1340,
    "findingsCount": 0
  },
  "scenarioResults": [
    {
      "scenarioId": "reference.quality.probes_assumptions",
      "status": "completed",
      "passed": true
    }
  ],
  "traceRef": "trace_k1l2m3",
  "findingsRef": "findings_n4o5p6",
  "reproducibility": {
    "traceCompleteness": "complete",
    "replayable": true
  }
}
```

The agent passed. If it had failed, `violations` would list specific failures with typed evidence pointers back into the Trace.

---

## Step 8 — (Optional) Retrieve the Trace

For replay, debugging, or re-scoring under a different rubric, fetch the Trace.

```bash
curl -H "Authorization: Bearer $AEP_TOKEN" \
     https://eval.example.com/aep/traces/trace_k1l2m3
```

The Trace contains every turn's input and output, timestamps, the full injected context snapshot, and any tool calls or policy events appropriate to the session's mode. It is sealed and immutable.

---

## Step 9 — End the session

Release the session resources.

**REST**

```bash
curl -X DELETE -H "Authorization: Bearer $AEP_TOKEN" \
     https://eval.example.com/aep/sessions/sess_a1b2c3d4
```

**JSON-RPC**

```json
{ "jsonrpc": "2.0", "id": 8, "method": "aep.session.end",
  "params": { "sessionId": "sess_a1b2c3d4" } }
```

The session is now `closed`. The Trace and Result remain retrievable for at least 7 days (AEP-REQ-047).

---

## The full flow in one picture

```
┌──────────┐  GET /aep/agents          ┌──────────┐
│Evaluator ├─────────────────────────▶ │  Server  │
│          │ ◀───────── [AgentContract] │          │
│          │                            │          │
│          │  POST /aep/sessions        │          │
│          ├─────────────────────────▶ │          │
│          │ ◀──── [sessionId, active]  │          │
│          │                            │          │
│          │  POST .../scenarios        │          │
│          ├─────────────────────────▶ │  Agent   │
│          │                            │  runs    │
│          │ ◀───── [turns, assertions] │          │
│          │                            │          │
│          │  POST .../evaluate         │ Scoring  │
│          ├─────────────────────────▶ │          │
│          │ ◀──────────── [resultRef]  │          │
│          │                            │          │
│          │  POST .../report           │          │
│          ├─────────────────────────▶ │  Seal    │
│          │                            │          │
│          │  GET /aep/results/{ref}    │          │
│          ├─────────────────────────▶ │          │
│          │ ◀────── [EvaluationResult] │          │
│          │                            │          │
│          │  DELETE /aep/sessions/{id} │          │
│          ├─────────────────────────▶ │          │
│          │ ◀─────────────── [closed]  │          │
└──────────┘                            └──────────┘
```

Eight HTTP requests for a complete evaluation. Six if you skip scenario and run turns directly. Four if you skip reporting and result retrieval until later.

## What to implement first

If you're building an AEP server, the minimum viable path through this walkthrough is:

1. `GET /aep/agents/{id}` returning a valid Agent Contract
2. `POST /aep/sessions` creating a session with injected context
3. `POST /aep/sessions/{id}/turns` executing a single turn
4. `POST /aep/sessions/{id}/evaluate` producing a result
5. `GET /aep/results/{id}` returning an EvaluationResult
6. `GET /aep/traces/{id}` returning a Trace
7. `DELETE /aep/sessions/{id}` closing the session

If those seven endpoints work correctly — with auth, input validation, context injection, and mode enforcement — you have a functioning AEP-Server. Everything else in the spec is either making those seven endpoints richer (policy profiles, tool declarations, red-teaming) or making them more operable (rate limits, audit logs, error recovery).

## Optional: stream events as they happen

For multi-minute agents, waiting for the sealed trace hurts. Servers advertising the `streaming` extension expose `GET /aep/sessions/{id}/events` as a Server-Sent Events stream.

```bash
curl -N -H "Authorization: Bearer $AEP_TOKEN" \
     -H "Accept: text/event-stream" \
     https://eval.example.com/aep/sessions/sess_a1b2c3d4/events
```

The server emits typed events as the session progresses. Note each event's `traceEventRef` pointing at the Trace entry it projects — the stream is a live emission of the future sealed Trace, not a separate log.

```
id: 1
event: session.started
data: {"traceEventRef":"sessionStart","traceSeq":0,"data":{"sessionId":"sess_a1b2c3d4","agentId":"example.inquiry-assistant","agentVersion":"1.2.0","testMode":"graybox","startedAt":"2026-04-17T12:00:01Z"}}

id: 2
event: turn.started
data: {"traceEventRef":"turns[0]","traceSeq":1,"data":{"turnId":"t1","turnIndex":0,"input":{"message":"I want to add AI..."},"startedAt":"2026-04-17T12:00:02Z"}}

id: 3
event: stream.heartbeat
data: {"timestamp":"2026-04-17T12:00:15Z"}

id: 4
event: turn.completed
data: {"traceEventRef":"turns[0]","traceSeq":2,"data":{"turnId":"t1","output":{"response":"What assumptions..."},"latencyMs":1340}}

id: 5
event: session.evaluated
data: {"traceEventRef":"finalOutcome","traceSeq":3,"data":{"sessionId":"sess_a1b2c3d4","resultRef":"result_h8i9j0"}}

id: 6
event: stream.end
data: {"reason":"session-complete","sealed":true,"traceRef":"trace_k1l2m3"}
```

Key properties to rely on:

- **Trace equivalence** (AEP-REQ-069): events applied in `id` order to an empty trace produce the same content as the sealed Trace retrieved via polling. Your judge code processes `Trace`; whether it arrived as a stream or a pull is a transport detail.
- **Resume on disconnect**: reconnect with `Last-Event-ID: 3` header and the server replays events 4 onward.
- **Fall-through guarantee** (AEP-REQ-070): if the stream connection fails permanently, the sealed Trace is still retrievable via `GET /aep/traces/{traceRef}` after the session completes.
- **Heartbeats** arrive every ≤30 seconds so clients can detect dead connections.

Streaming is an observability extension. It does not let the client abort or redirect a session; that's a v0.2 concern.

## What this walkthrough doesn't cover

- **Error handling** — see [spec §12](../spec/AEP-0.1.md#12-error-handling)
- **Red-teaming** — see [`RED-TEAMING.md`](./RED-TEAMING.md)
- **Replay** — see [`REPLAY.md`](./REPLAY.md)
- **Multi-turn state** — covered in the spec, but the happy-path example is single-turn for brevity
- **Branching scenarios** — see `examples/reference-agent/scenarios/safety-refuses-adversarial.json`

Each of those is a real concern; none of them is needed for the happy path.
