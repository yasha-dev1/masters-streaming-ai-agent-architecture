# Standardising Push-Based Event Delivery to Long-Running AI Agents

*Synthesis report. Follow-up to `confluent-rtce-deep-dive.md` and RQ4. Date: 2026-04-18.*

*Feeds on four research notes in this directory:*
- `01-mcp-subscribe-deep-dive.md` — what MCP already specifies for push
- `02-a2a-and-agent-push-landscape.md` — non-MCP agent protocols
- `03-event-envelope-standards.md` — CloudEvents, AsyncAPI, Standard Webhooks
- `04-push-delivery-semantics.md` — transports, guarantees, back-pressure, priority

---

## 1. Executive summary

1. **MCP already has the push primitive.** `resources/subscribe` + `notifications/resources/updated` + server-initiated messages on Streamable HTTP cover the wire-level mechanics. No new protocol is needed for *transport*.
2. **The gap is semantic, not syntactic.** Today's `notifications/resources/updated` carries only a URI — it is a doorbell, not a payload. There is no standardised event body, no offset/resume, no priority, no watermark, no back-pressure, no fan-out model, no long-lived-auth story.
3. **No viable non-MCP standard exists for "upstream event source → long-running agent."** A2A's push (`tasks/pushNotificationConfig`) is directionally wrong (agent→client task callbacks, not external→agent subscriptions). AGNTCY SLIM is transport-only at IETF draft-01. LangGraph `interrupt()` is a Python in-process primitive, not a wire protocol.
4. **The right artefact is an MCP profile**, not a fork: *MCP Streaming Resources*. It combines (a) CloudEvents v1.0.2 as the notification body, (b) Standard-Webhooks-style signing at HTTP ingress edges, (c) explicit resume/offset/back-pressure/priority semantics layered onto MCP JSON-RPC, and (d) a small number of new control messages (`flow/grant`, `flow/ack`).
5. **The open MCP SEP landscape gives this a legitimate landing zone.** SEP-2495 (April 2026) explicitly names the gap "server-push that re-enters the LLM loop." SEP-2532 (resource streaming), SEP-2567/2575 (stateless MCP), and the 2026 community-interest roadmap item *"triggers and event-driven updates"* are all adjacent. A thesis-scale spec proposal is not swimming against the current.

**Answer to the user's framing question:** no new protocol. Define a profile on MCP that pins down the semantics MCP leaves underspecified, plus a CloudEvents-JSON-RPC binding that doesn't yet exist in the CNCF spec tree. That is a reproducible, citable, open-source contribution.

---

## 2. The gap, stated precisely

The thesis needs a protocol that supports:

| Requirement | Why it's needed | Current MCP status |
|---|---|---|
| Server push to long-running agent | Business events (Slack DM, Stripe dispute) can't wait for next poll | Primitive exists (`notifications/resources/updated`) |
| Event *content* in the push | Avoid round-trip-per-event tax; let fast-path push tiny payloads | **Missing** — URI-only doorbell |
| Offsets / resume-from-sequence | Agent reconnects after network blip without losing events | **MAY-level** resumption on Streamable HTTP; no offset semantics |
| Priority / urgency tier | "Notify RIGHT AWAY" vs "deliver within a minute" vs "batch overnight" | **Missing** |
| Back-pressure / flow control | Agent too slow to consume must not OOM the stream | **Missing** (transport has TCP/HTTP-2 windows; app layer doesn't surface them) |
| At-least-once + idempotency | Standard reliability bar; exactly-once-into-LLM is a myth | **Missing** normative language |
| Watermark / completeness | Agent knows "all events up to T have arrived" before reasoning | **Missing** |
| Fan-out to multiple agents | Shared subscription across a consumer group | **Missing** |
| Long-lived auth refresh | Subscriptions that live days need token rotation | Partial (mention in spec, not specified) |
| Standard payload envelope | Cross-vendor tooling (schema registry, tracing, replay) | **Missing** (no CloudEvents binding) |

None of these are invented requirements — every row has a precedent in MQTT 5, AMQP 1.0, Kafka, NATS JetStream, or Standard Webhooks. The gap is that nobody has composed them into a single MCP profile.

---

## 3. Why extend MCP rather than invent something new

### 3.1 The wire-level primitive is already there

Research note 01 confirms: MCP 2025-11-25 specifies both `resources/subscribe` and `notifications/resources/updated`, and Streamable HTTP (the 2025-06-18 replacement for HTTP+SSE) supports server-initiated messages with `Mcp-Session-Id` + `Last-Event-ID` resumption. The scaffolding is present. What is missing is the *semantic contract*.

Building a new protocol from scratch would re-derive JSON-RPC framing, SSE transport, session management, and auth — all things MCP already settles. That is waste.

### 3.2 No viable non-MCP standard fits

Research note 02 surveyed the landscape. The only standard-in-progress in the agent space is **A2A** (Linux Foundation, 150+ orgs, v0.3 adds gRPC). A2A's push mechanism — `tasks/pushNotificationConfig/set` with JWT/JWKS-signed webhook callbacks — is aimed at the wrong direction: *the agent* pushes task-progress updates to *its client* (which A2A assumes is another agent). Inverting A2A to accept "external event source notifies agent" works against A2A's conceptual model.

Other candidates:

- **AGNTCY SLIM** (Cisco/AGNTCY, IETF draft-01) is a pub/sub transport layer with MLS group encryption. Real push primitive, but no agent-semantic layer on top, and "IETF draft-01" is a pre-adoption state.
- **LangGraph `interrupt()`** has the correct shape (pause-and-wait-for-external-input) but `Command(resume=...)` is a Python function call, not a wire format.
- **OpenAI Realtime API / Gemini Live** offer real bidirectional WebSocket push but are voice-optimised, session-bounded to ~10 minutes, and structurally wrong for "agent subscribed for days."
- **Temporal Signals, Inngest events** solve the problem at the orchestrator layer; they are runtime primitives, not cross-vendor wire protocols.
- **Webhook → queue → worker** (n8n, Zapier, Make, Pipedream, Lambda fan-in) is the house pattern for wiring Slack/GitHub/Stripe to agents. No standard, no ordering, no replay. CloudEvents appears at envelope level between hops but says nothing about agent invocation.

The conclusion from note 02 is load-bearing: *there is a genuine open standardisation slot at the agent layer for "upstream event → agent subscription," and MCP is the only protocol positioned to fill it without conceptual contortion.*

### 3.3 The SEP pipeline is open and aligned

Research note 01 surfaced **SEP-2495** (*Event-Driven Tool Invocation / Server-Push to LLM Re-entry*, open draft, April 2026) which names the gap verbatim: *"MCP supports server-to-client notifications, but these only update metadata — they never re-enter the LLM loop."* SEP-2532 (Resource Streaming for binary, open), SEP-2567/2575 (stateless MCP + explicit state handles, open), and the 2026 roadmap item *"Triggers and event-driven updates"* (explicitly flagged as community-interest) give the thesis a legitimate target: contribute the profile as an SEP, or at minimum publish it alongside a reference implementation that satisfies SEP-2495's ask.

Prior rejected SEPs — SEP-1006 (Bidirectional Tool Calls), SEP-593 (Webhooks as server capability) — are instructive: both lost because they over-extended the protocol surface. A profile that reuses `subscribe`/`notify` and adds narrow control messages is less likely to face the same objection.

---

## 4. Proposed profile: MCP Streaming Resources

The sections below are a sketch, not a finished spec. Each subsection names what changes normatively vs what stays a MAY.

### 4.1 Three-layer stack

```
┌──────────────────────────────────────────────────────────────┐
│  Domain event (Slack msg, JIRA transition, price change)     │  ← your business model
├──────────────────────────────────────────────────────────────┤
│  CloudEvents v1.0.2 envelope                                  │  ← identity, routing, tracing
│  (id, source, type, time, subject, traceparent, partitionkey,│
│   sequence, dataref, rate, recordedtime) + data              │
├──────────────────────────────────────────────────────────────┤
│  MCP JSON-RPC frame over Streamable HTTP                      │  ← transport, session, auth
│  notifications/resources/updated with CloudEvent in params    │
└──────────────────────────────────────────────────────────────┘
```

This stack reuses two CNCF/W3C standards the ecosystem already has tooling for (CloudEvents, Streamable HTTP) and adds one binding that does not yet exist: **CloudEvents-over-JSON-RPC** (note 03 verified no such binding in `cloudevents/spec/cloudevents/bindings`).

### 4.2 Content-bearing `notifications/resources/updated`

Today:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": { "uri": "ctx://unit/slack/thread/C123/T456" }
}
```

Profile-upgraded:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "ctx://unit/slack/thread/C123/T456",
    "event": {
      "specversion": "1.0",
      "id": "01HS7Z...QK",
      "source": "ctx-engine/slack-ingest",
      "type": "com.example.ctx.unit.updated",
      "time": "2026-04-18T14:02:11Z",
      "subject": "thread:C123/T456",
      "partitionkey": "C123",
      "sequence": "187432",
      "priority": "p1",
      "datacontenttype": "application/json",
      "data": { /* delta or self-contained CU snapshot */ },
      "dataref": "ctx://unit/slack/thread/C123/T456?rev=187432",
      "traceparent": "00-abc...-00-01"
    }
  }
}
```

Two modes, both MUST-supported by profile-conformant servers:

- **Structured (event carries data)** — client gets what it needs without follow-up. Required for small events in the hot path.
- **Claim-check (event carries `dataref`, empty `data`)** — client calls `resources/read?uri=dataref` to fetch. Required when `data` exceeds a profile-configurable threshold (e.g. 16 KB). Preserves the "notify signal, pull payload" pattern recommended in RQ4.

### 4.3 Offsets, resume, at-least-once

Streamable HTTP has `Last-Event-ID` replay but base MCP only says `MAY`. The profile upgrades this to:

- Server MUST advertise a replay window in `initialize` response: `{streamingResources: {replay: {events: 10000, seconds: 600}}}`.
- Each notification MUST carry `sequence` (monotonic per resource URI, per-session).
- On reconnect, client sends `Last-Event-ID` OR an MCP-level `resources/resume` with `{uri, afterSequence}` — the latter is durable across sessions, the former is best-effort within a stream.
- Idempotency contract: `(event.source, event.id)` is the dedup key. Clients MUST be idempotent consumers. Delivery is at-least-once.
- Effectively-once end-to-end into the LLM is **not promised** — note 04's honest finding is that exactly-once-into-LLM is unachievable; agents must tolerate duplicates.

### 4.4 Back-pressure: `flow/grant` and `flow/ack`

Two new control notifications, both client→server:

```json
{"method": "flow/grant", "params": {"uri": "ctx://...", "credits": 100}}
{"method": "flow/ack",   "params": {"uri": "ctx://...", "throughSequence": "187450"}}
```

Semantics lifted from Reactive Streams `request(n)` + AMQP settlement. Server MUST NOT send more than `credits` outstanding notifications per URI. `flow/ack` releases credits and signals consumer-side durability. This closes the OOM-on-fast-producer hole that SSE and raw WebSockets leave open (note 04 §4).

### 4.5 Priority

Three tiers — aligned with AMQP 1.0 priority semantics (the only surveyed protocol with native priority):

- `p0` — urgent, interrupt-class. Agent SHOULD preempt current turn (if runtime supports it; LangGraph `interrupt()` is the reference).
- `p1` — high, deliver-ahead. Server MUST emit before queued p2 events for the same subscription.
- `p2` — normal, best-effort.

Priority is advisory: the client runtime decides how to react. The profile only specifies the signal and the ordering obligation at the server.

This is the direct answer to the user's observation *"there will always be events that the agents should be notified RIGHT AWAY."* `p0` with a client that supports agent interruption is the standardised path.

### 4.6 Watermarks and completeness

Borrowed from Dataflow Model / Beam / Flink (note 03 §4, note 04 §3.5). A separate notification:

```json
{
  "method": "notifications/resources/watermark",
  "params": { "uri": "ctx://...", "eventTime": "2026-04-18T14:00:00Z", "sequence": "187432" }
}
```

Tells the agent "all events up to this event-time/sequence have been delivered." Critical for RQ3's batch-path correctness: the agent can reason over a complete window rather than a partial one.

No IETF/CNCF watermark envelope exists — this is an original contribution the thesis can ship as a CloudEvents extension or an MCP-specific notification. Cite Akidau et al. (VLDB 2015), Begoli et al. (PVLDB 2021).

### 4.7 Fan-out / consumer groups

Lifted from MQTT 5 shared subscriptions and Kafka consumer groups. Extra param on subscribe:

```json
{
  "method": "resources/subscribe",
  "params": {
    "uri": "ctx://unit/slack/thread/*",
    "group": "triage-agents",
    "startFrom": { "mode": "latest" | "earliest" | {"afterSequence": "..."} }
  }
}
```

Multiple agents joining the same `group` get partitioned delivery by `event.partitionkey`. Without `group`, each subscriber gets every event (pub/sub fan-out, default).

### 4.8 Long-lived auth

MCP spec allows OAuth but does not specify refresh for subscriptions that outlive an access token. Profile normative language: server MUST accept a `notifications/auth/refreshed` notification carrying the new token, scoped to the session. Closes the "agent subscribed for days" gap.

### 4.9 External-origin webhooks

When the event originates *outside* MCP — a Stripe dispute POSTed to a webhook URL — the edge adapter signs it per **Standard Webhooks** (`webhook-id`, `webhook-timestamp`, `webhook-signature`), unwraps it into a CloudEvent, and emits it as a profile-conformant MCP notification on the other side. The profile spec SHOULD reference standardwebhooks.com for ingress, CloudEvents HTTP binding for cross-system hops, and the CloudEvents-JSON-RPC binding (to be authored) for the MCP leg.

---

## 5. Delivery guarantees: provable vs aspirational

From note 04 §8, adapted:

| Claim | Provable? | Mechanism |
|---|---|---|
| At-least-once delivery to subscriber | Yes | Replay window (W seconds, N events) + `Last-Event-ID`/`resources/resume` |
| Exactly-once delivery to subscriber | No | Not achievable over HTTP with retries; reduce to at-least-once + idempotent consumer |
| Exactly-once *into the LLM turn* | No | Agent loops, tool calls, and retries break this everywhere |
| FIFO per partition key | Yes | Server MUST preserve per-`partitionkey` ordering; no global order promise |
| Gap detection | Yes | Monotonic `sequence` per URI; consumer detects holes |
| Priority ordering within a subscription | Yes | Server MUST emit higher-priority events ahead of queued lower-priority ones |
| Back-pressure bound | Yes | `flow/grant` credits |
| Cross-agent consistency | **Open** | RQ5 territory; profile can propagate watermarks but multi-agent consistency is unspecified across systems (identified gap) |

Being honest about what's achievable matters for the thesis's credibility. Don't promise exactly-once.

---

## 6. What this is not

- **Not a replacement for Kafka/NATS/Pulsar.** Those remain the backbone; MCP Streaming Resources is the *agent-facing* edge. Internally, RQ1's Kappa+/Streamhouse recommendation still holds.
- **Not a new protocol.** It is a profile: a set of normative upgrades and conventions that any MCP server can opt into, signalled in `initialize` via a `streamingResources` capability block.
- **Not a drop-in replacement for A2A.** A2A is agent-to-agent task delegation. MCP Streaming Resources is data-source-to-agent push subscription. They are orthogonal and composable.

---

## 7. Relationship to the thesis contributions

From `report.md` §6, the thesis had identified three contribution opportunities. This profile map to them as follows:

1. **Empirical streaming-architecture benchmark on agent workloads** — the profile is the API the benchmark measures against. Without it, there is no standard "agent-facing push interface" to benchmark.
2. **Evaluation methodology for optimal Context Units** — unchanged, but the profile provides the delivery vehicle for CUs, and watermarks tell the evaluator when a window is complete.
3. **Multi-agent consistency model** — the profile's `group` + `partitionkey` + watermark primitives are the building blocks for a formal consistency spec. This is the most open piece; the profile documents the primitives, the thesis specifies the guarantees atop them.

A **fourth contribution** is now explicit: **the MCP Streaming Resources profile specification and reference implementation**. This is the single most shippable OSS artefact from the thesis, and it plugs directly into the open SEP-2495.

---

## 8. Concrete next steps

1. **Write the profile spec** (`docs/spec.md` in a thesis-owned repo). Structure it like an IETF/MCP-SEP draft: abstract, normative language, JSON-RPC examples, interaction diagrams, security considerations, IANA considerations (event-type naming).
2. **Propose the CloudEvents-JSON-RPC binding** to the CNCF CloudEvents working group. It's a ~2-page binding document; the precedent is the existing Kafka/AMQP/MQTT/HTTP bindings.
3. **Build a reference MCP server** that fronts a Flink/Paimon or Redpanda-backed materialised view and speaks the profile. This is the OSS "equivalent of RTCE with subscribe support" identified in `confluent-rtce-deep-dive.md` §6.2.
4. **Build a reference client** (or extend an existing MCP client — Claude Code has open issues #2722, #4094, #44283 for not consuming `list_changed` notifications) to prove the loop closes.
5. **Engage with SEP-2495 authors.** The profile is a concrete proposal for how to satisfy that SEP's motivation. Worth cross-referencing.
6. **Benchmark** against RTCE (pull-only) and against Streaming Agents (in-Flink push) on the same workload — this is the RQ1+RQ4 empirical study.

---

## 9. Answer to the user's exact question, restated

> *"what is the best way to standardize this? should this be a new protocol compatible with MCP that lets upstream event-based system push events to long-running agents? is it even possible as protocol or should there be anything else?"*

It is possible as a protocol. It should **not** be a new protocol — MCP already owns this interface surface area, its spec has the primitives, and the 2026 roadmap has explicitly flagged "event-driven updates" as an open community-interest slot. The right artefact is a **profile on MCP** that pins down the semantics MCP leaves underspecified: CloudEvents envelope, at-least-once + idempotency, offsets/resume, back-pressure credits, priority tiers, watermarks, and consumer-group fan-out. A CloudEvents-JSON-RPC binding (which doesn't exist in the CNCF spec tree today) is a clean companion contribution to the CloudEvents working group.

> *"even if we crack tableflow and make an event-stream based system, there will always be events that the agents should be notified RIGHT AWAY!"*

Correct, and this is exactly the case the `p0` priority tier plus a client-side `interrupt()` pattern (à la LangGraph) is designed for. The standard specifies the signal and the server-side ordering obligation; the client runtime decides how to react (preempt current turn, or queue ahead). AMQP 1.0's 0–9 priority byte is the only comparable precedent in the transport surveys, so the space is genuinely underspecified and the thesis has room to contribute.

---

## 10. Consolidated reference list

Primary sources used across the four research notes; duplicates collapsed. Numbering is local to this synthesis — the sub-notes carry their own numbering.

**MCP spec and ecosystem**
1. Model Context Protocol Specification, version 2025-11-25. <https://modelcontextprotocol.io/specification/2025-11-25>
2. MCP Streamable HTTP transport. <https://modelcontextprotocol.io/specification/2025-06-18/basic/transports>
3. `modelcontextprotocol/specification` — SEP index and discussions. <https://github.com/modelcontextprotocol/specification>
4. SEP-2495 — Event-Driven Tool Invocation / Server-Push to LLM Re-entry (open, draft, April 2026).
5. SEP-2532 — Resource Streaming for binary payloads (open, draft, April 2026).
6. SEP-2567 / SEP-2575 — Stateless MCP + Explicit State Handles (open, April 2026).
7. SEP-1686 — Long-Running Operations / Tasks (merged into 2025-11-25).
8. SEP-593 — Webhooks as server capability (closed, rejected).
9. SEP-1006 — Bidirectional Tool Calls (closed, rejected).
10. MCP 2026 Roadmap. <https://modelcontextprotocol.io/blog/2026-roadmap>
11. `modelcontextprotocol/servers` — reference server implementations. <https://github.com/modelcontextprotocol/servers>

**Agent-to-agent and adjacent protocols**
12. Agent2Agent (A2A) Protocol v0.3. Linux Foundation. <https://a2a-protocol.org>, <https://github.com/a2aproject/A2A>
13. AGNTCY SLIM (IETF draft-01). <https://datatracker.ietf.org/doc/draft-agntcy-slim/>
14. LangGraph `interrupt()` documentation. <https://langchain-ai.github.io/langgraph/>
15. OpenAI Realtime API. <https://platform.openai.com/docs/guides/realtime>
16. Gemini Live API. <https://ai.google.dev/gemini-api/docs/live>
17. Temporal Signals. <https://docs.temporal.io/encyclopedia/application-design-patterns#signals>
18. Inngest events. <https://www.inngest.com/docs/events>

**Event envelope and webhook standards**
19. CloudEvents v1.0.2 core. <https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md>
20. CloudEvents HTTP binding. <https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/http-protocol-binding.md>
21. CloudEvents extensions: `partitionkey`, `sequence`, `dataref`, `rate`, `recordedtime`, `traceparent`. <https://github.com/cloudevents/spec/tree/main/cloudevents/extensions>
22. AsyncAPI 3.0. <https://www.asyncapi.com/docs/reference/specification/v3.0.0>
23. Standard Webhooks specification. <https://www.standardwebhooks.com/>, <https://github.com/standard-webhooks/standard-webhooks>
24. xRegistry (CNCF). <https://github.com/xregistry/spec>
25. AWS EventBridge Schema Registry. <https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-schema-registry.html>
26. IETF `Idempotency-Key` header, draft-07. <https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/>
27. RFC 9457 — Problem Details for HTTP APIs. <https://datatracker.ietf.org/doc/html/rfc9457>

**Transports and delivery semantics**
28. WHATWG HTML — Server-Sent Events. <https://html.spec.whatwg.org/multipage/server-sent-events.html>
29. RFC 6455 — The WebSocket Protocol. <https://datatracker.ietf.org/doc/html/rfc6455>
30. RFC 9113 — HTTP/2. <https://datatracker.ietf.org/doc/html/rfc9113>
31. W3C WebTransport. <https://www.w3.org/TR/webtransport/>
32. OASIS MQTT 5.0. <https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html>
33. OASIS AMQP 1.0. <https://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-overview-v1.0-os.html>
34. Apache Kafka consumer protocol. <https://kafka.apache.org/documentation/#consumerapi>
35. NATS JetStream. <https://docs.nats.io/nats-concepts/jetstream>
36. Reactive Streams specification. <https://www.reactive-streams.org/>

**Streaming foundations**
37. Akidau et al. *The Dataflow Model.* VLDB 2015. <https://research.google/pubs/pub43864/>
38. Begoli et al. *One SQL to Rule Them All.* PVLDB 2021.

**Thesis internal**
39. `research/RQ1-lambda-vs-kappa.md`
40. `research/RQ4-context-distribution.md`
41. `research/RQ5-shared-memory.md`
42. `research/confluent-rtce-deep-dive.md`
43. `research/push-protocol/01-mcp-subscribe-deep-dive.md`
44. `research/push-protocol/02-a2a-and-agent-push-landscape.md`
45. `research/push-protocol/03-event-envelope-standards.md`
46. `research/push-protocol/04-push-delivery-semantics.md`
