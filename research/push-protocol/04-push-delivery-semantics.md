# 04 — Push Delivery Semantics for Streaming Events to AI Agents

> **Focus.** Survey how existing push protocols handle reliability, ordering,
> replay, backpressure, and priority; map the design space; and derive a
> concrete specification for an *MCP Streaming Resources* push profile
> (or, failing that, a thin new protocol) that can deliver events to
> long-running AI agents with stated guarantees rather than hand-waving.

---

## 1. Executive Summary

The thesis's working architecture (RQ4 [R-RQ4]) is *notify-then-pull*: a
thin push channel carries change signals, and agents pull full state
from a Context Engine (an MCP server). This note closes: **what delivery
guarantee does the push channel actually provide, and what is the
minimal wire-level contract that makes that guarantee implementable and
observable?**

1. **Provable: at-least-once, per-key FIFO, resumable within a bounded
   replay window.** What every production system (Kafka [R-Kafka],
   JetStream [R-NATS], MQTT 5 QoS 1 [R-MQTT5], AMQP 1.0 unsettled-at-
   receiver [R-AMQP]) delivers. Exactly-once is either single-hop
   marketing (MQTT QoS 2 [R-MQTT5 §4.3.3]) or requires a transactional
   consumer — which an LLM agent is not.
2. **Aspirational: effectively-once.** Agent-side dedup on a stable
   `event_id`. The protocol enables dedup; it cannot enforce it.
3. **Minimal field set.** `event_id`, `source`, `sequence`,
   `partitionkey`, `priority`, `dataref`, plus MCP's `Mcp-Session-Id`
   and SSE event `id` for resumption.
4. **MCP suffices — with a profile.** Streamable HTTP already carries
   session identity and SSE event IDs; the other fields slot into the
   JSON-RPC notification payload as CloudEvents-style attributes
   [R-CE-Core]. No new protocol is required.
5. **Out of scope.** Global total order, causal order across sources,
   end-to-end exactly-once. Per-key FIFO + idempotent consumers is
   what the design commits to.

---

## 2. Transport Mechanisms Compared

The table normalises ten widely-deployed server-to-client push
mechanisms against the dimensions that matter for an agent fast path.

| Protocol | Bidirectional | Ordering | Delivery | Resumable | Replay window | Backpressure | Native priority |
|---|---|---|---|---|---|---|---|
| **SSE** (HTML Living Std.) [R-SSE] | No (server→client only) | FIFO per stream | Best-effort (TCP) | Yes via `Last-Event-ID` | Server-defined | TCP only | No |
| **WebSocket** (RFC 6455) [R-WS] | Yes | FIFO per connection | Best-effort (TCP) | No (spec), app-level only | None (spec) | TCP + ping/pong | No |
| **HTTP/2 Server Push** (RFC 9113) [R-H2] | Server-initiated only | FIFO per stream | Best-effort | No | None | WINDOW_UPDATE [R-H2 §5.2] | No |
| **HTTP/3 / WebTransport** [R-WT] | Yes | Per-stream FIFO; datagrams unordered | Streams reliable; datagrams unreliable-by-choice | No (connection-level) | None | QUIC flow control | No |
| **gRPC streaming** [R-gRPC] | Yes | FIFO per stream | Best-effort (HTTP/2) | No (spec); deadlines + cancellation | None | HTTP/2 flow control | No (headers-only) |
| **Webhooks** [R-StdWebhook] | No (POST-per-event) | None | At-least-once via retries (pattern) | No | Provider-defined (Stripe: 3 days) | HTTP 429, provider-defined | No |
| **MQTT 5** [R-MQTT5] | Yes | FIFO per topic/session | QoS 0/1/2 (provable at broker hop) | Yes via Session Expiry | Retained msg (last) | Receive Maximum | No |
| **AMQP 1.0** [R-AMQP] | Yes | FIFO per link | At-most/at-least/exactly-once via settlement | Yes via link recovery | Broker-defined | Credit-based flow control | Yes (msg priority 0-9) |
| **NATS JetStream** [R-NATS] | Yes | FIFO per subject/stream | At-least-once via ack; at-most/all | Yes via durable consumers | By sequence / time | Pull batches | No |
| **Kafka consumer** [R-Kafka] | Yes | FIFO per partition | At-least/at-most/exactly-once (txn) | Yes via committed offsets | Full retention window | Consumer poll | No (pattern only) |
| **MCP Streamable HTTP** [R-MCP-Transport] | Yes (GET+POST) | Per-stream FIFO | Inherits SSE (best-effort TCP) | Yes via `Mcp-Session-Id` + `Last-Event-ID` | "MAY replay" — unspecified | None (spec) | No |

Two things to note. First, every transport that claims "exactly-once"
means it only on a *single hop* — MQTT QoS 2 [R-MQTT5 §4.3.3] dedups
within the broker; Kafka's EoS [R-Kafka] requires a transactional sink,
and an LLM is not one. Second, every resumable transport does so via an
identifier the client presents at reconnect (`Last-Event-ID`, Session
Expiry, committed offsets, consumer name); the agent side of the MCP
profile must present an equivalent.

### 2.1 SSE (HTML Living Standard)

The SSE spec [R-SSE §9.2.4] defines the resumption contract directly:
each event *may* carry an `id:` field; the user agent remembers the most
recent one; on reconnect it sends it back in the `Last-Event-ID` HTTP
header. That is the extent of the guarantee — the server decides whether
to replay from that cursor; SSE does not persist anything. Streams are
UTF-8 only [R-SSE §9.2.5] ("Event streams in this specification must
always be encoded as UTF-8"), so binary payloads must be base64'd.
Ordering is trivially FIFO because the stream is text over one TCP
connection.

### 2.2 WebSocket (RFC 6455)

RFC 6455 guarantees message-fragment ordering ("Message fragments MUST
be delivered to the recipient in the order sent by the sender"
[R-WS §5.4]), but only TCP-level reliability. There is no replay, no
session concept, no idempotency. Any of those must be layered in the
application protocol on top — which is what Socket.IO, STOMP, and
MQTT-over-WS do. For an agent fast path, WebSocket alone is not enough.

### 2.3 HTTP/2 Server Push vs. HTTP/3 WebTransport

HTTP/2 server push [R-H2 §8.4] is still in the spec but effectively
dead. Chrome disabled it by default in Chrome 106 (2022) citing low
adoption and mixed performance [R-ChromePush]; Firefox 132 removed it
in October 2024. It never worked as an application-level push channel —
it pushes *resources*, not events, and cannot resume.

HTTP/3 WebTransport [R-WT] is the modern shape: bidirectional streams,
unreliable datagrams (useful for "loss is fine if it's old"), and
stream IDs that survive packet loss. It still provides no
application-level replay; resumption is a higher-level concern.

### 2.4 gRPC streaming

gRPC server-streaming and bidirectional streaming [R-gRPC] inherit
HTTP/2 flow control automatically: per-stream credit via WINDOW_UPDATE
frames [R-H2 §6.9]. Deadlines and cancellation are explicit in the
protocol. gRPC does *not* provide resumption: if the HTTP/2 stream
dies, the RPC is over. Structurally isomorphic to MCP's Streamable
HTTP, and the same reconnect obligation falls on the application.

### 2.5 Webhooks

Webhooks are the lowest-common-denominator push: an HTTP POST per
event. There is no formal W3C spec; the Standard Webhooks project
[R-StdWebhook] is the closest normative reference. Every production
implementation (Stripe, GitHub, Shopify) converges on the same recipe:
at-least-once via producer retries with exponential backoff
(Stripe retries for three days); signing with HMAC-SHA256 over
`timestamp.body` (e.g. GitHub's `X-Hub-Signature-256`); idempotency key
in a header (GitHub's `X-GitHub-Delivery` UUID; CloudEvents `id`
[R-CE-Core]). Webhooks have no ordering, no streaming, no cursor — a
poor fit for low-latency fan-out to resident agents.

### 2.6 MQTT 5

MQTT 5 [R-MQTT5] is the most self-contained reliability story in the
survey. Three QoS levels: QoS 0, at-most-once (§4.3.1); QoS 1,
at-least-once via PUBLISH/PUBACK (§4.3.2); QoS 2, exactly-once via the
PUBLISH → PUBREC → PUBREL → PUBCOMP handshake (§4.3.3). Sessions
persist across disconnects via `Session Expiry Interval`
[R-MQTT5 §3.1.2.11.2] (up to 0xFFFFFFFF = never expires). Shared
subscriptions [R-MQTT5 §4.8.2] (`$share/` prefix) give competing-
consumer semantics. MQTT 5 has no message priority.

### 2.7 AMQP 1.0

AMQP 1.0 [R-AMQP §2.6.12] exposes the raw settlement machinery that
MQTT's QoS levels are a convenience over. A delivery is **settled** or
**unsettled** on each end of the link; by choosing where each side
settles, any of the three canonical guarantees is directly expressible.
AMQP 1.0 is the only widely-deployed protocol in the survey with a
native **message priority** field (0–9).

### 2.8 NATS JetStream

JetStream [R-NATS] layers persistent, replayable streams on top of
NATS's pub/sub. Consumers are **durable** (named, state persisted) or
**ephemeral** (anonymous). Ack policy is `explicit | all | none`;
`MaxDeliver` caps redelivery; `AckWait` is the redelivery timeout. The
**DeliverPolicy** enumerates replay origins: `DeliverAll`,
`DeliverLast`, `DeliverByStartSequence`, `DeliverByStartTime`,
`DeliverLastPerSubject`. This catalogue is the design vocabulary the
MCP profile wants.

### 2.9 Kafka consumer protocol

Kafka's delivery guarantees [R-Kafka] are the cleanest reference: at-
most-once (commit before processing), at-least-once (process then
commit), exactly-once (atomic transactional commit across processed
data + offset). Ordering is **per-partition FIFO**; global order is not
provided and is an anti-pattern at scale. Consumer groups provide
competing-consumer semantics; replay is full retention.

### 2.10 MCP Streamable HTTP

MCP's current transport [R-MCP-Transport] uses HTTP POST for client→
server messages, with optional SSE streams initiated by GET (server-
initiated) or POST (streaming response). Two headers carry the
reliability story:

- **`Mcp-Session-Id`** — assigned on initialization, echoed on every
  request; server MAY terminate (HTTP 404) at any time.
- **`Last-Event-ID`** — client echoes the last SSE event ID on reconnect;
  server MAY replay. Spec is explicit that IDs are per-stream.

The normative language is weak: "the server **MAY** use this header to
replay messages that would have been sent after the last event ID, on
the stream that was disconnected, and to resume the stream from that
point" [R-MCP-Transport §Resumability]. MCP gives the mechanics; the
profile in §7 makes them normative.

---

## 3. Delivery Semantics Taxonomy

### 3.1 The three canonical guarantees

- **At-most-once** — no duplicates, potential loss. Fire-and-forget
  (MQTT QoS 0, Kafka with pre-process commit). Unusable for anything
  an agent acts on.
- **At-least-once** — no loss, potential duplicates. The default for
  every durable system (Kafka, JetStream, MQTT QoS 1, AMQP unsettled-
  at-receiver, webhook retry). The realistic target for the agent fast
  path.
- **Exactly-once** — only achievable within a transactional boundary
  (MQTT QoS 2 *at the broker*, Kafka EoS across producer + consumer +
  sink). End-to-end into an LLM agent is not achievable: the LLM and
  its tool calls are not transactional, and their side effects cannot
  be rolled back.

### 3.2 Effectively-once via idempotent consumer

The operational stand-in is **effectively-once**: at-least-once
delivery plus a consumer-side dedup set keyed on a stable `event_id`.
CloudEvents [R-CE-Core] is explicit:

> "Producers MUST ensure that `source` + `id` is unique for each
> distinct event. [...] Consumers MAY assume that Events with
> identical `source` and `id` are duplicates." [R-CE-Core §Attributes]

For the agent runtime, the idempotency set is a bounded LRU keyed on
`(source, id)` sized to exceed the broker's replay window.

### 3.3 Ordering, gap detection, replay windows

Three ordering models appear in practice: **global total order** (one
partition / one writer; wrong choice at scale), **FIFO per partition
/ per key** (Kafka partitions, JetStream subjects, AMQP links — the
sweet spot), and **causal order** (Lamport / vector clocks; not native
in any mainstream push protocol). The RQ4 design uses per-key FIFO
keyed on subscription scope (customer ID, conversation ID, resource
URI).

Per-source monotonic **sequence** numbers let consumers detect loss
without a timeout; CloudEvents' sequence extension [R-CE-Seq]
formalises this as a lexicographically orderable string per source. A
**watermark** is the dual: "no events below this sequence/timestamp
will arrive", as in Flink [Flink].

Replay windows come in three shapes: *last-N* (MQTT retained,
retention-config), *last-T* (JetStream `MaxAge`, Stripe's three-day
retry horizon), *from-offset* (Kafka offsets, JetStream
`DeliverByStartSequence`). The MCP profile needs a finite window —
minutes to hours — large enough to cover a routine restart.

---

## 4. Backpressure and Flow Control

Backpressure is what distinguishes a push protocol from an
out-of-control pipe. Four tiers appear across the survey.

**TCP-level only.** Raw SSE over HTTP/1.1 and raw WebSocket get
backpressure from TCP's receive window — free, but invisible to the
application. The server cannot drop, prioritise, or substitute stale
data when the window closes.

**HTTP/2 / HTTP/3 flow-control windows.** WINDOW_UPDATE
[R-H2 §5.2, §6.9] implements per-stream and per-connection credit
(default 65,535 octets); "flow control cannot be disabled"
[R-H2 §5.2.1]. gRPC, SSE-over-HTTP/2, and MCP's Streamable HTTP
inherit this automatically, but it is not exposed to application code
— the agent cannot easily say "I'm falling behind."

**Reactive Streams `request(n)`.** The Reactive Streams spec [R-RS]
codifies application-level backpressure as a subscriber-to-publisher
credit signal: the subscriber requests at most N more elements and the
publisher MUST NOT exceed that. This is the right abstraction for an
agent; the MCP profile surfaces it as a JSON-RPC notification
(§8.4).

**Broker-native credit.** AMQP 1.0 credit-based flow control
[R-AMQP §2.6.7] and JetStream pull-batch size [R-NATS] implement the
same pattern at the protocol level and are the most robust
backpressure mechanism in the survey.

For agents, the signal flowing *back* to the Context Engine should
include `credit` (Reactive-Streams-style), `last_processed_sequence`
(advances the replay cursor), and `lag_ms` (observability).

---

## 5. Subscription Lifecycle & Resumption

**Durable vs. ephemeral.** Every mature push protocol distinguishes
these: MQTT 5's `Clean Start` flag [R-MQTT5 §3.1.2.4], JetStream's
durable-name presence [R-NATS], Kafka's named consumer groups, AMQP
link terminus durability. Ephemeral subscriptions exist only while
connected; durable ones survive process restarts and carry a cursor.
Agents need both: durable for long-running background agents,
ephemeral for session-scoped ones.

**Shared / competing-consumer subscriptions.** MQTT 5's
`$share/<group>/<topic>` [R-MQTT5 §4.8.2], Kafka consumer groups
[R-Kafka], JetStream queue subscriptions [R-NATS], and AMQP shared-
distribution links all express the same idea: each event is delivered
to exactly one member of the group. The MCP profile must name this
explicitly — horizontally-scaled agent worker pools require it.

**Reconnect state.** What each protocol preserves:

| Protocol | Session ID | Cursor | Subs | Queued events |
|---|---|---|---|---|
| SSE | No | Yes (`Last-Event-ID`) | N/A | Up to server |
| WebSocket | No | No | No | No |
| MQTT 5 | ClientID | Yes (packet IDs) | Yes | Yes (Session Expiry) |
| AMQP 1.0 | Container ID | Yes (link state) | Yes | Yes |
| Kafka | Group + member ID | Yes (offsets) | Yes | Log-based |
| JetStream | Durable name | Yes | Yes | Yes (stream retention) |
| MCP Streamable HTTP | `Mcp-Session-Id` | Per-stream `Last-Event-ID` | Server choice | "MAY" |

MCP is strictly weakest — it specifies the mechanism but leaves the
guarantee up to the server. §8 upgrades this from MAY to SHOULD on a
bounded window.

**Auth for long-lived subscriptions.** A subscription that lives for
hours outlives most bearer tokens. The simplest pattern — and what
MCP's Streamable HTTP already encourages — is **reconnect on expiry**:
server returns `unauthorized`, client reconnects with fresh creds and
resumes via `Last-Event-ID`.

---

## 6. Priority and Urgency

The agent case — "a fraud alert must interrupt the reasoning in
progress" — has no clean protocol-level expression in most of the
surveyed stack. MQTT 5 has no priority field; Kafka has no native
priority (priority-topic patterns are community recipes, not spec);
SSE / WebSocket / gRPC have no payload priority (HTTP/2 has per-stream
priority [R-H2 §5.3] but that's link-layer). **AMQP 1.0 is the
exception**: message priority is a native `header.priority` byte 0–9
[R-AMQP]. MCP has nothing today.

Priority shows up in two distinct places for an agent fast path:

1. **Transport priority** — should this event skip queued events? This
   is what AMQP's priority header does at the broker, within a link.
2. **Application urgency** — should this event *interrupt* the agent's
   current LLM call or tool execution? This is the layer where
   LangGraph's `interrupt()` [R-LG] and OpenAI Realtime interruption
   live. LangGraph's primitive pauses a graph at a node until the
   caller resumes with `Command(resume=...)`; Realtime streams accept
   a cancellation event over the same WebSocket. Neither is triggered
   by a transport bit — they are triggered by an application policy
   acting on an event's declared priority.

The profile in §8 carries a `priority` enum (`low | normal | high |
interrupt`). `interrupt` priority demands a durable, at-least-once
channel with the lowest-latency transport available — no batching, no
coalescing.

---

## 7. Reliability Composition Patterns

None of these are new; what is useful is how they compose with push.

**Transactional outbox.** The producer writes the domain mutation and
the outbound event to the same database transaction; a relay drains
the outbox to the push channel [Kleppmann]. Already adopted in the
thesis RQ4 design [R-RQ4 §6]. Without it, the producer can crash
after writing state and before publishing, and no downstream protocol
can recover what never got sent.

**Claim-check / dataref.** When payloads are large, the event carries
a reference, not the payload. CloudEvents `dataref` [R-CE-DataRef] is
the canonical expression. The MCP profile uses an `mcp://...` resource
URI as `dataref`, matching the notify-then-pull pattern.

**CDC + event-carried state** admits three points on a spectrum:
signal-only (cheapest, always requires pull), full event-carried state
(expensive fan-out), and **hybrid** — signal + optional `dataref` with
small payloads inlined. The hybrid is what the profile recommends.

**Exactly-once illusion.** The producer stamps `event_id`, the
consumer dedups (§3.2). The guarantee is "no double-effect on the
consumer, assuming the consumer remembers seen IDs for the replay
window."

**End-to-end vs. hop-by-hop ack.** MQTT QoS 2 acks are hop-by-hop
("the broker has this"). End-to-end means "the agent has processed
this", and only the agent can send it. The profile must expose an
application-level ack (§8.4) — JetStream's explicit ack [R-NATS] is
the right model; MQTT QoS 1's PUBACK is not.

---

## 8. Thesis Recommendation: The MCP Streaming-Resources Push Profile

### 8.1 Does MCP suffice?

Yes, with a profile. MCP's Streamable HTTP transport
[R-MCP-Transport] provides session identity (`Mcp-Session-Id`),
resumption scaffolding (`Last-Event-ID`), bidirectional messaging
(GET+POST), and an extensible JSON-RPC notification shape. The gaps
are *semantic, not transport* — MCP does not prescribe event shape,
priority, or gap detection. A new protocol is not needed; a *profile*
— additional fields and normative upgrades — is.

### 8.2 Normative upgrades over base MCP

The profile turns the following base-MCP MAYs into SHOULDs:

- Servers **SHOULD** assign SSE event IDs to every event on the
  session's server-initiated stream. (Base: MAY.)
- Servers **SHOULD** replay events on `Last-Event-ID` reconnect for at
  least *W* seconds / *N* events past the last ack, where *W* and *N*
  are advertised at initialization. (Base: MAY, unspecified window.)
- Servers **SHOULD** accept a client-sent `flow/grant` notification
  (Reactive Streams–style credit) and **MUST NOT** exceed granted
  credit. (Base: absent.)
- Servers **SHOULD** accept a client-sent `flow/ack` notification with
  a `last_processed_sequence` that advances the server's replay
  cursor. (Base: absent.)

### 8.3 Event envelope

Each fast-path notification carries these fields, drawn from
CloudEvents [R-CE-Core] and the extensions reviewed in §2–§4. Binding
is JSON on top of MCP's JSON-RPC `notifications/streaming/event`
method:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/streaming/event",
  "params": {
    "id":            "9f1b...c3",      // dedup key; unique per source
    "source":        "ctx://orders",   // ordering scope
    "type":          "order.updated",  // routing / predicate matching
    "time":          "2026-04-18T10:30:00Z",
    "sequence":      "0000001234",     // lex-ordered per source
    "partitionkey":  "customer-42",    // FIFO unit
    "priority":      "high",           // low|normal|high|interrupt
    "dataref":       "mcp://ctx/resources/orders/42",
    "data":          { "...": "..." }  // optional inline payload
  }
}
```

Every field has a principled source: `id`, `source`, `type`, `time`,
`data` from CloudEvents core [R-CE-Core]; `sequence` from the
sequence extension [R-CE-Seq] for gap detection; `partitionkey` from
the partitioning extension [R-CE-Part] for per-key FIFO; `priority`
modelled on AMQP 1.0 [R-AMQP] (no CloudEvents equivalent); `dataref`
from the claim-check extension [R-CE-DataRef], pointing into the MCP
Context Engine's resource surface.

### 8.4 Control messages

Two additional JSON-RPC notifications from client to server:

```json
{"method": "flow/grant", "params": {"n": 50}}
{"method": "flow/ack",   "params": {"last_processed_sequence": "0000001234"}}
```

Both are unidirectional (client → server); both are idempotent modulo
`last_processed_sequence` monotonicity. `flow/grant` implements
Reactive-Streams credit [R-RS]; `flow/ack` implements
JetStream-style explicit ack [R-NATS] and lets the server advance the
replay cursor.

### 8.5 Provable vs. aspirational guarantees

| Property | Status | Basis |
|---|---|---|
| At-least-once within replay window | **Provable** | Server persists until `flow/ack ≥ sequence`; client resumes via `Last-Event-ID`. |
| Per-`partitionkey` FIFO | **Provable** | Single server-authored stream per session; sequence monotonic per source. |
| Gap detection | **Provable** | Client compares `sequence`; missing = gap. |
| Bounded replay | **Provable** | *W* seconds / *N* events advertised at initialization. |
| Effectively-once at the agent | **Aspirational** | Requires agent-side dedup on `(source, id)` — profile mandates it but cannot enforce. |
| Interrupt latency bound | **Aspirational** | `priority: interrupt` is a hint; actual interruption is an application policy (LangGraph `interrupt()` [R-LG], Realtime API cancel). |
| Causal order across sources | **Not provided** | Out of scope. |
| Global total order | **Not provided** | Out of scope. |
| Exactly-once end-to-end | **Not provided** | Not achievable with a non-transactional LLM consumer. |

### 8.6 Where "push signal, then pull full payload" fits

The RQ4 two-channel pattern [R-RQ4] falls out cleanly. The **push
channel** carries the envelope with `data` omitted (or summarised) and
`dataref` pointing at an `mcp://…/resources/…` URI. The **pull
channel** is MCP `resources/read` on that URI. Agents that need only
the change signal ignore the pull; agents that need the payload
follow `dataref`. Cross-path deduplication is via the shared `(source,
id)` — the pulled state carries the ID of the most recent event, and
the agent's idempotency set covers both. The reliability budget is
spent where it matters.

### 8.7 Observability obligations

Three metrics must be exported for the guarantees to be verifiable:
`replay_buffer_depth_{events,seconds}` per session (to check *W* and
*N*), `outstanding_credit` per session (visible backpressure), and
`last_processed_sequence_lag` per session (the key health metric).
Without these, §8.5's guarantees are claims, not contracts.

---

## 9. References

[R-SSE] WHATWG. *HTML Living Standard — Server-Sent Events (§9.2).*
<https://html.spec.whatwg.org/multipage/server-sent-events.html>

[R-WS] Fette, I., Melnikov, A. (2011). *The WebSocket Protocol.*
RFC 6455. <https://datatracker.ietf.org/doc/html/rfc6455>

[R-H2] Thomson, M., Benfield, C. (2022). *HTTP/2.* RFC 9113.
<https://www.rfc-editor.org/rfc/rfc9113.html>

[R-WT] W3C. *WebTransport.* W3C Working Draft.
<https://www.w3.org/TR/webtransport/>

[R-gRPC] gRPC Authors. *gRPC Core Concepts, Architecture and Lifecycle.*
<https://grpc.io/docs/what-is-grpc/core-concepts/>

[R-StdWebhook] Standard Webhooks Project. *Standard Webhooks
specification.* <https://github.com/standard-webhooks/standard-webhooks>

[R-MQTT5] OASIS. *MQTT Version 5.0.* OASIS Standard, 2019.
<https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html>

[R-AMQP] OASIS. *Advanced Message Queuing Protocol (AMQP) Version
1.0 — Part 2: Transport.* OASIS Standard, 2012.
<https://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-transport-v1.0-os.html>

[R-NATS] Synadia / NATS.io. *NATS JetStream Consumers documentation.*
<https://docs.nats.io/nats-concepts/jetstream/consumers>

[R-Kafka] Apache Kafka. *Message Delivery Semantics, Design
documentation.* <https://kafka.apache.org/documentation/#semantics>
and Confluent, *Kafka Delivery Semantics.*
<https://docs.confluent.io/kafka/design/delivery-semantics.html>

[R-MCP-Transport] Anthropic / MCP. *Model Context Protocol
Specification 2025-06-18 — Basic: Transports.*
<https://modelcontextprotocol.io/specification/2025-06-18/basic/transports>

[R-RS] Reactive Streams Initiative. *Reactive Streams specification.*
<https://www.reactive-streams.org/>

[R-CE-Core] CNCF. *CloudEvents — Version 1.0 Specification.*
<https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md>

[R-CE-DataRef] CNCF. *CloudEvents — `dataref` (Claim Check Pattern)
extension.*
<https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/dataref.md>

[R-CE-Part] CNCF. *CloudEvents — Partitioning extension.*
<https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/partitioning.md>

[R-CE-Seq] CNCF. *CloudEvents — Sequence extension.*
<https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/sequence.md>

[R-ChromePush] Weiss, Y. (2022). *Removing HTTP/2 Server Push from
Chrome.* Chrome for Developers blog.
<https://developer.chrome.com/blog/removing-push>

[R-LG] LangChain. *LangGraph Interrupts — Human-in-the-Loop.*
<https://docs.langchain.com/oss/python/langgraph/interrupts>

[R-RQ4] This thesis. *RQ4 — Context Distribution to Agents: Pull,
Push, or Hybrid?* See `/research/RQ4-context-distribution.md`,
especially §6 (Delivery semantics) and §6.3 (Notification-then-pull).

[Kleppmann] Kleppmann, M. (2017). *Designing Data-Intensive
Applications.* O'Reilly. (Transactional outbox discussion, Chapter 11.)

[Flink] Apache Flink. *Event Time and Watermarks.*
<https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/time/>
