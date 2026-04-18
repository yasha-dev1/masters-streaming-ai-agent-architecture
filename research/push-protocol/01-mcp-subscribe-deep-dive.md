# MCP Subscribe / Push Deep Dive

**Scope.** Determine whether MCP, as it stands in April 2026, can serve as a *push* protocol between a materialised streaming-context server and an AI agent — i.e. whether the gap identified in the Confluent RTCE analysis (MCP as pull-only) can be closed by *extending* MCP rather than inventing a new protocol. This note is a primary-source audit of MCP's existing subscribe/notify surface, its transports, its SEP pipeline, and its ecosystem adoption.

---

## Executive Summary

| Question | Answer |
|---|---|
| Does MCP specify a `subscribe` primitive today? | **Yes**, but only for `resources` (since 2024-11-05). A client calls `resources/subscribe {uri}` and the server **MAY** send `notifications/resources/updated {uri}`. [1][2] |
| Does the update notification carry the new content (delta or full)? | **No.** The notification carries *only* the URI; it is a "re-read me" hint. The client must issue a second `resources/read` round-trip to fetch content. [1][2] |
| Can a server push spontaneously on stdio? | **Yes**, any time, as a newline-delimited JSON-RPC notification on stdout. [3][4] |
| Can a server push spontaneously on Streamable HTTP? | **Yes**, but only once an SSE stream is open — either opportunistically after a client POST, or on a client-initiated `GET /mcp` that the server upgrades to `text/event-stream`. [3][4] |
| Is there replay / at-least-once / ordering? | **Partial / weak.** SSE `id:` + `Last-Event-ID` gives per-stream resumption *if* the server chooses to implement replay. No global ordering, no dedup, no offset model, no persistent log. Servers **MUST NOT** replay messages from a different stream. [4] |
| Is there back-pressure, priority, or batch semantics? | **No.** Nothing in the spec. |
| Do popular clients actually consume `notifications/resources/updated`? | **Largely no.** Claude Desktop advertises `"resources": {"subscribe": false, "listChanged": false}` in its initialize result. Claude Code has had open issues for over a year on not handling `notifications/prompts/list_changed`. [5][6] |
| Are there open SEPs closing this gap? | **Partially.** SEP-2495 (Event-Driven Tool Invocation), SEP-1006 (Bidirectional Tool Calls, closed), SEP-593 (Webhooks, closed), SEP-1391 (Long-Running Operations, superseded by Tasks/SEP-1686), SEP-2532 (Resource Streaming for binary, draft). None of them standardise the *payload* of a materialised-context push. [7][8][9][10][11] |
| What is genuinely missing for a streaming-context server? | Standardised **content-bearing** update notification; **delivery semantics** (offset, at-least-once, replay from offset, dedup); **back-pressure**; **priority / urgency**; **batch vs single-event**; **multi-subscriber fan-out semantics**; **watermarks / completeness**; **auth lifecycle for long-lived subscriptions**. |

**Bottom line.** MCP has the *syntactic* scaffolding for push — `resources/subscribe`, `notifications/resources/updated`, and a bidirectional SSE stream — but no *semantic* model for delivering materialised context. The notification is a doorbell, not a payload. Streamable HTTP's resumption story is per-stream and best-effort. The 2026 roadmap [12] explicitly lists "Triggers and event-driven updates" and "streamed / reference-based result types" as community-interest areas (not core-maintainer-driven), meaning a thesis-scale protocol extension has space to land.

---

## Section 1. Existing push primitives in the MCP spec

The current published spec is **2025-11-25** [13]. The prior version 2025-06-18 [1] is the one most blog/tooling content references. Changes to the `resources/subscribe` surface between these two versions: **none of substance** (only `icons[]` were added to the `Resource` type). The push primitives below are stable across 2024-11-05 → 2025-03-26 → 2025-06-18 → 2025-11-25.

### 1.1 `resources/subscribe` and `notifications/resources/updated`

**Capability declaration** (server's `InitializeResult`) [1][2]:

```json
{
  "capabilities": {
    "resources": {
      "subscribe": true,
      "listChanged": true
    }
  }
}
```

Both flags are independently optional.

**Subscribe request (client → server)** [1][2]:

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/subscribe",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}
```

Semantics:
- `params.uri` is the **single** resource URI the client wants to be notified about.
- The spec does **not** define:
  - A subscription ID in the response (the response is an empty `{}` result).
  - Multi-URI, glob, or pattern subscribe.
  - A filter / predicate / query on the subscription.
  - A "since offset" or "from timestamp" parameter.
  - Backpressure / QoS flags.
- Unsubscribe is symmetric: `resources/unsubscribe {uri}`.

**Update notification (server → client)** [1][2]:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}
```

This is the critical finding for the thesis:

> **The notification carries only the URI.** The spec's own sequence diagram [1] shows `notifications/resources/updated` followed by a fresh `resources/read` call. There is no delta, no patch, no version counter, no watermark, no new-content field. It is a "re-read me" doorbell.

Consequence for a streaming-context use case: every push triggers at least one extra JSON-RPC round trip to actually fetch the payload. This multiplies latency and complicates delivery semantics (the state observed at read time may already have drifted past the state at notification time).

### 1.2 `list_changed` family

Three symmetric notifications [1][2][14][15], each gated by a `listChanged` capability flag:

```json
{"jsonrpc":"2.0","method":"notifications/resources/list_changed"}
{"jsonrpc":"2.0","method":"notifications/tools/list_changed"}
{"jsonrpc":"2.0","method":"notifications/prompts/list_changed"}
```

All three are empty — no params, no delta, no per-item subscribe. Client must re-issue `resources/list` / `tools/list` / `prompts/list` to learn what changed. Broadcast to every connected client.

### 1.4 `notifications/progress`

Scoped to a *specific in-flight request* via a `progressToken` in request `_meta` [16]:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "abc123",
    "progress": 50,
    "total": 100,
    "message": "Reticulating splines..."
  }
}
```

`progress` MUST increase monotonically. This is the closest thing in MCP to a **payload-carrying** server-initiated notification — but it is explicitly tied to a *single long-running request* and its lifecycle ends when the response is returned. It is not a subscribe/notify channel for arbitrary resources.

### 1.5 `notifications/message` (logging)

Gated by `capabilities.logging: {}`. Carries `{level, logger, data}` with syslog-style levels (`debug`/`info`/`notice`/`warning`/`error`/`critical`/`alert`/`emergency`) [17]:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/message",
  "params": {
    "level": "error",
    "logger": "database",
    "data": {"error": "Connection failed", "details": {"host": "localhost", "port": 5432}}
  }
}
```

Clients set the minimum level with `logging/setLevel`. This is **payload-bearing**, **server-initiated**, and **not tied to a specific request** — conceptually closest to a push channel. But it is explicitly scoped to *log messages*, and the spec explicitly forbids using it for credentials, PII, or business data [17]. It cannot legitimately carry materialised context.

### 1.6 `notifications/tasks/status` (new in 2025-11-25)

Tasks (SEP-1686 [18]) introduce durable request state machines with `taskId`, `status ∈ {working, input_required, completed, failed, cancelled}`, `ttl`, `pollInterval`. The notification [19]:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tasks/status",
  "params": {
    "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
    "status": "completed",
    "createdAt": "2025-11-25T10:30:00Z",
    "lastUpdatedAt": "2025-11-25T10:50:00Z",
    "ttl": 60000,
    "pollInterval": 5000
  }
}
```

Crucially, the spec states: **"Requestors MUST NOT rely on receiving this notification, as it is optional."** [19]. Tasks are explicitly *poll-first, notify-optional*. The protocol doubles down on pull — servers give a `pollInterval`, clients are expected to poll `tasks/get`. Tasks are not a streaming push primitive; they are an async job token.

### 1.7 Sampling and Elicitation (server-initiated *requests*, not notifications)

- **`sampling/createMessage`** [20] — server asks the *client* to run an LLM completion. Flow is server→client *request* with a matching response. Not a push of context into the agent; a pull of inference from the client.
- **`elicitation/create`** [21] — server asks the *client* to prompt the user for structured input. Same direction (server→client request). New in 2025-06-18. Cannot carry arbitrary content — restricted to flat JSON Schema with primitive fields only.

Both are server-initiated but are *request/response* pairs, not fire-and-forget notifications. They are semantically "the server needs something" not "the server has new data."

### 1.8 Summary

| Method | Payload-bearing? | Scoped to a subscription? |
|---|---|---|
| `notifications/resources/updated` | **No** (URI only) | Yes — `resources/subscribe` |
| `notifications/*/list_changed` (resources/tools/prompts) | No | No (broadcast) |
| `notifications/progress` | **Yes** | No — tied to a request via `progressToken` |
| `notifications/message` (logging) | **Yes** (but log only) | No (capability flag) |
| `notifications/tasks/status` | **Yes** | No — per-task, optional |
| `sampling/createMessage`, `elicitation/create` | Yes (request/response) | No |

**The only payload-bearing channel a subscriber could try to abuse for streaming context today is `notifications/progress` — but it is scoped to a single request and dies with its response. There is no standardised mechanism for a server to push *content* tied to a resource subscription.**

---

## Section 2. Transport layer — how pushes actually travel

### 2.1 stdio

Spec [3][4]: server writes newline-delimited JSON-RPC to `stdout`, including notifications at any time (logging to `stderr`). No buffering model beyond the OS pipe (~64KB on Linux; `write()` blocks when full — crude back-pressure with no acks). No replay on crash. 1:1 by construction. Fine for local IDE integration, **unsuitable for materialised streaming context at any scale**.

### 2.2 HTTP+SSE (deprecated)

The 2024-11-05 two-endpoint transport (`POST /messages` + `GET /sse`) was deprecated in 2025-03-26. Retained only for backwards compatibility [3][4].

### 2.3 Streamable HTTP (current, since 2025-03-26)

Single endpoint `/mcp` supports both `POST` and `GET` [4]. Behavior:

**POST flow (client → server request):**
1. Client `POST /mcp` with a JSON-RPC request body and `Accept: application/json, text/event-stream`.
2. Server decides: return a single JSON response (`Content-Type: application/json`) or upgrade to SSE (`Content-Type: text/event-stream`).
3. If SSE: server **MAY** send additional JSON-RPC *requests* and *notifications* on the stream "related to the originating client request" before the final response. After the final response, server **SHOULD** terminate the stream.

**GET flow (client opens a pure listen stream):**
1. Client `GET /mcp` with `Accept: text/event-stream`.
2. Server returns `text/event-stream` or 405.
3. Server **MAY** send requests and notifications that are **"unrelated to any concurrently-running JSON-RPC request"**. This is the channel for spontaneous push — `notifications/resources/updated` for a subscribed URI lives here.
4. Server **MUST NOT** send a response on this stream unless resuming a disconnected stream.

**Session management** [4]:
- Server **MAY** mint an `MCP-Session-Id` on `InitializeResult`. Client echoes it on subsequent requests.
- Session is a stickiness hint, not a durable log — server **MAY** terminate at any time with HTTP 404, forcing the client to re-initialize.

**Resumability** [4]:
- Server **MAY** attach SSE `id:` field to events.
- Client can reconnect via `GET /mcp` with `Last-Event-ID: <id>`.
- Server **MAY** replay messages that would have been delivered on *that same stream* after that ID. Server **MUST NOT** replay messages from a different stream.
- Consequence: there is **no session-wide ordered log**. Each SSE stream has its own event ID space (the spec requires IDs unique *within the session*, but replay is scoped to the originating stream only).

**New in 2025-11-25 (SEP-1699 [22]):** the server **SHOULD** immediately send an empty SSE event with an ID to prime the client for reconnect, **MAY** close the connection without terminating the stream to avoid holding long-lived connections, and **SHOULD** send a `retry:` field before closing. This is explicitly a **polling-over-SSE** pattern — the stream logically persists while the TCP connection is transient.

**Multi-connection rule** [4]:
> "The server MUST send each of its JSON-RPC messages on only one of the connected streams; that is, it MUST NOT broadcast the same message across multiple streams."

This matters for the thesis: if the agent has two SSE streams open (one from POST, one from GET), the server has to pick one. There is no multi-channel delivery, and no per-channel ordering guarantee across streams.

### 2.4 What happens when the client disconnects?

Per 2025-11-25 [4]:
- The server may or may not retain messages.
- Replay is *best effort* and *per-stream*.
- Session may be terminated at any time; on HTTP 404 the client re-initializes — any missed notifications are **gone**.
- There is no analogue of Kafka consumer offsets, NATS JetStream durable subscriptions, or AMQP durable queues.

### 2.5 Future transport direction (Dec 2025 transport working group post [23])

The WG is explicitly moving toward **stateless** transport and **explicit subscription streams**: "Replacing general-purpose GET streams with explicit subscription streams for monitoring specific items." SEPs due Q1 2026, spec update targeted June 2026. No WebSocket. No push/webhook. The direction is "HTTP evolves, sessions become data-plane state handles" — SEP-2575 (Make MCP Stateless [24]) and SEP-2567 (Sessionless MCP via Explicit State Handles) are the relevant in-flight proposals.

---

## Section 3. Spec evolution — SEPs relevant to push/subscribe

I searched the spec repo `github.com/modelcontextprotocol/modelcontextprotocol` for issues/PRs with titles matching `subscribe|stream|push|notif|SEP|event|trigger|watch|webhook`. Key results:

### 3.1 Open / draft SEPs that would bear on push

**SEP-2495 — Event-Driven Tool Invocation (Server-Push to LLM Re-entry)** [7]
- Opened April 2026, **open / draft**.
- *"Currently, MCP follows a strict request-response pattern: the LLM client can call tools on the server, but the server cannot trigger a new LLM turn in response to an event. While MCP supports server-to-client notifications (e.g., `notifications/tools/list_changed`), these only update metadata — they never re-enter the LLM loop."*
- Proposes three options: notification-based, request-based, subscription-based. **This SEP explicitly names the thesis's gap.**

**SEP-2532 — Resource Streaming for Binary Content Delivery** [11]
- Opened April 2026, **open / draft**. Not merged.
- Proposes a `resources/stream` method that returns a `downloadUrl` to be GETed out-of-band as raw bytes.
- Scoped to binary content, not streaming *updates*. Solves the base64 inflation problem, not the push problem. Relevant as evidence that the spec does not today support streaming content at all (even pull-streaming).

**SEP-2575 — Make MCP Stateless** [24] and **SEP-2567 — Sessionless MCP via Explicit State Handles** (April 2026, open)
- Proposes removing the initialize handshake, per-request protocol versioning, and `messages/listen` as a dedicated client-initiated streaming endpoint. **Subscribe semantics would migrate onto `messages/listen`** if this lands.

**SEP-2571 — Resource Submission (client-to-server resource creation)** [25]
- April 2026, open. Adds `resources/create` / `resources/delete`. Orthogonal to push but relevant: it makes the resource abstraction bidirectional, which a materialised-context server would need (e.g., agent deposits a context snapshot).

### 3.2 Closed / rejected / superseded SEPs

**SEP-1006 — Bidirectional Tool Calls** [9] — *closed, not merged*. Proposed letting servers call tools on the agent. Rejected in favor of sampling + elicitation + tasks. The discussion thread is the clearest articulation that the core team sees server-initiated action as a first-class design tension.

**SEP-593 — Webhooks as a server capability** [10] — *closed*. Proposed a `webhook` field on `tools/call` so the server could POST the async tool result to a client-supplied URL. Rejected in favor of SEP-1686 Tasks.

**SEP-1391 — Long-Running Operations** [8] — *closed*, superseded by SEP-1686 Tasks.

**SEP-1686 — Tasks** [18] — *merged into 2025-11-25*. The current answer to long-running async. Poll-first, notify-optional. Does **not** address streaming context push.

**SEP-992 — Notification Configuration for Tool Call Result** — *closed*. Would have let clients configure per-tool notification preferences.

### 3.3 Discussion of delivery semantics

I searched for explicit discussion of at-least-once / ordering / dedup / replay in the spec repo. **The spec is silent on all four.** The only relevant text is the SSE-`Last-Event-ID` paragraph [4], which is:
- Per-stream (not per-subscription, not per-session).
- Best-effort (`MAY replay`, not `MUST`).
- Non-durable (no persistence requirement on the server).

This is effectively **best-effort at-most-once with optional server-side replay** — i.e. no stronger semantics than raw SSE.

---

## Section 4. Ecosystem adoption of subscribe/notify

### 4.1 Official reference servers (`github.com/modelcontextprotocol/servers`)

As of April 2026, the official `servers` repo contains seven reference implementations: `everything`, `fetch`, `filesystem`, `git`, `memory`, `sequentialthinking`, `time`.

Of these, **only `everything` implements `resources/subscribe`**, and it does so explicitly as a *demonstration harness*. From its `features.md` [26]:

> "Simulated update notifications are opt-in and off by default. Clients may subscribe/unsubscribe to resource URIs using the MCP `resources/subscribe` and `resources/unsubscribe` requests. Use the `toggle-subscriber-updates` tool to start/stop a per-session interval that emits `notifications/resources/updated { uri }` only for URIs that session has subscribed to."

None of the production-oriented servers (`filesystem`, `git`, `memory`, `fetch`) implement subscribe. The `filesystem` server is the obvious candidate (watch a file, push on change) and does not.

### 4.2 Client support

**Claude Desktop.** Community reports and GitHub issues [5] show Claude Desktop advertises `"resources": {"subscribe": false, "listChanged": false}` in its initialize capabilities. It does not consume resource update notifications.

**Claude Code.** Long-standing open issue anthropics/claude-code#2722 [6]: *"Claude Code doesn't respond to MCP notifications/prompts/list_changed for dynamic command updates."* Issue #4094 requests the same feature for prompts list change. Issue #44283 reports that channel notifications are not handled. The harness does not subscribe to or consume `notifications/resources/updated`.

**Cursor, Cline, Continue, Zed.** Tool-focused clients. No evidence of `resources/subscribe` support in public documentation or issue trackers. MCP adoption in these clients centers on `tools/call`.

**Net:** The subscribe primitive exists in the spec, is implemented in a demo server, and is consumed by **no mainstream client**. This is consistent with the thesis framing: the *protocol syntax* is there, the *ecosystem semantics* are not.

### 4.3 Streaming-oriented MCP servers in the wild

Searching GitHub for Kafka/NATS/Redis-backed MCP servers [27] returns a meaningful cluster of *Kafka MCP servers* (`tuannvm/kafka-mcp-server`, `aswinayyolath/kafka-mcp-server`, `kanapuli/mcp-kafka`, `shivamxtech/kafka-mcp`, `wklee610/kafka-mcp`, `Joel-hanson/kafka-mcp-server`, etc.). Inspection of their tool surfaces (per their READMEs) shows:

- They expose Kafka operations as **`tools/call`** (`produce_message`, `consume_messages`, `list_topics`, `describe_topic`, `consumer_group_*`).
- Consumption is **batch-pull**: `consume_messages` takes a `max_messages` and `timeout`, creates an ephemeral consumer, pulls a batch, returns. No continuous push.
- **None advertise `resources/subscribe`**. None use `notifications/resources/updated`.

Confluent's RTCE is the closest public example of a materialised-streaming-context MCP server, and the thesis's prior note confirmed it is pull-only [see `research/confluent-rtce-deep-dive.md`].

StreamNative's blog post [28] introduces their MCP server as another tool-based exposure of Pulsar/Kafka, same pattern.

**Conclusion:** the entire streaming-backend-over-MCP ecosystem wraps its source in `tools/call`. The `resources/subscribe` primitive is unused in production.

---

## Section 5. Concrete gaps the thesis could close

Given the inventory above, the thesis has room to propose either (a) a profile/extension to MCP that fills in semantics the current spec lacks, or (b) a compatible SEP that standardises the missing payload/delivery layer. Either way the gaps are:

### 5.1 Content-bearing update notification

Today `notifications/resources/updated {uri}` → client does `resources/read {uri}`: two round-trips per push, and server-observed state may have drifted between notify and read. There is no standard field for `contents`, `delta`, `patch`, `version`, or `revision`. Extension: a `notifications/resources/pushed` (or optional `contents` / `revision` fields on the existing notification) carrying the new content inline for small payloads and a resource-link for large ones (following SEP-2532's `downloadUrl` pattern).

### 5.2 Delivery guarantees (offsets, at-least-once, replay)

Today: best-effort, per-stream replay only, no offset, no dedup. A materialised streaming context needs at-least-once with offsets so an agent can reason about "have I seen this?" and a Kafka-backed server can surface native offsets. Extension: add `revision` (monotonic per-URI) and `sequence` (server-wide) on the update notification; add a `since` parameter to `resources/subscribe` for resume-from-revision; define a `lastSeenRevision` semantic on reconnect that is session-scoped rather than SSE-stream-scoped.

### 5.3 Back-pressure / flow control

Today: none. stdio gets OS pipe back-pressure non-normatively; SSE is fire-and-forget. A server emitting 1000 events/sec into a 10/sec client either drops silently or OOMs. Extension: per-subscription `maxRate` / `maxInFlight`, a `batchWindowMs`, or credit-based flow (`resources/subscribe` returns `credits`; client tops up with `resources/credit {uri, n}`).

### 5.4 Priority / urgency

Resources support a `priority` annotation (0.0–1.0) for display ranking [1], but the update notification itself is un-tiered. A streaming feed mixes breaking events and trickle updates; the protocol cannot mark this on the wire. Extension: a `priority` / `urgency` field on the notification.

### 5.5 Batching

Only single events today. Streaming sources produce correlated bursts (micro-batches, window closures) where N×1 messages is wasteful. Extension: `notifications/resources/updated_batch` with `updates: [{uri, revision, contents?}, ...]`.

### 5.6 Multi-subscriber semantics

The spec is 1:1 per session. No `subscriberId`, no `subscriberGroup` (Kafka-style). A materialised-context server serving thousands of agents has no protocol handle for fan-out, coalescing, or dedup across subscribers. Extension: subscriber groups and consumer-group-style offset commit.

### 5.7 Watermarks and completeness

No watermark notion today. Streaming systems need "you've seen everything up to T" so the agent knows whether context is current. Extension: `notifications/resources/watermark {uri, watermark}` or a `watermark` field on keepalives.

### 5.8 Auth lifecycle for long-lived subscriptions

OAuth is negotiated at initialize. A subscription lasting hours outlives most access tokens; token refresh during an open SSE stream is underspecified (force reconnect? inline refresh?). SEP-835/1046/1299 [29] address credential flows but not mid-stream refresh.

### 5.9 Summary matrix

| Gap | Spec today | Open SEP addressing it? | Thesis contribution potential |
|---|---|---|---|
| Content-bearing update | URI only | Partial — SEP-2495, SEP-2532 | **High** — no standardised payload notification exists |
| At-least-once / offsets | Best-effort | No | **High** — entirely absent |
| Back-pressure | None | No | **High** — entirely absent |
| Priority / urgency | `priority` annotation on resource only | No | Medium |
| Batching | Single events only | No | Medium |
| Multi-subscriber fan-out | 1:1 only | No | **High** — entirely absent |
| Watermarks | None | No | **High** — entirely absent |
| Long-lived auth | Initialize-time only | Partial | Low (other WGs working) |
| Event-driven LLM re-entry | None | **SEP-2495 (draft)** | Already in community pipeline — thesis can inform |

---

## References

[1] MCP Specification 2025-06-18 — Server Features: Resources. <https://modelcontextprotocol.io/specification/2025-06-18/server/resources>

[2] MCP Specification 2025-11-25 — Server Features: Resources. <https://modelcontextprotocol.io/specification/2025-11-25/server/resources>

[3] MCP Specification 2025-06-18 — Base Protocol: Transports. <https://modelcontextprotocol.io/specification/2025-06-18/basic/transports>

[4] MCP Specification 2025-11-25 — Base Protocol: Transports. <https://modelcontextprotocol.io/specification/2025-11-25/basic/transports>

[5] modelcontextprotocol/python-sdk Issue #1016 — "Claude Desktop never uses resources from my MCP server". <https://github.com/modelcontextprotocol/python-sdk/issues/1016>

[6] anthropics/claude-code Issue #2722 — "Claude Code doesn't respond to MCP notifications/prompts/list_changed for dynamic command updates". <https://github.com/anthropics/claude-code/issues/2722>. See also #4094, #44283, #1604.

[7] SEP-2495 — Event-Driven Tool Invocation (Server-Push to LLM Re-entry). <https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2495>

[8] SEP-1391 — Long-Running Operations (closed, superseded by Tasks). <https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1391>

[9] SEP-1006 — Bidirectional Tool Calls in Model Context Protocol (closed). <https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1006>

[10] SEP-593 — Feat: Add Webhooks as a server capability (closed). <https://github.com/modelcontextprotocol/modelcontextprotocol/issues/593>

[11] SEP-2532 — Resource Streaming for Binary Content Delivery (draft, April 2026). <https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2532>

[12] The 2026 MCP Roadmap, MCP Blog. <https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/>

[13] MCP Specification 2025-11-25 — Overview. <https://modelcontextprotocol.io/specification/2025-11-25>

[14] MCP Specification 2025-11-25 — Server Features: Tools. <https://modelcontextprotocol.io/specification/2025-11-25/server/tools>

[15] MCP Specification 2025-11-25 — Server Features: Prompts. <https://modelcontextprotocol.io/specification/2025-11-25/server/prompts>

[16] MCP Specification 2025-11-25 — Basic Utilities: Progress. <https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/progress>

[17] MCP Specification 2025-11-25 — Server Utilities: Logging. <https://modelcontextprotocol.io/specification/2025-11-25/server/utilities/logging>

[18] SEP-1686 — Tasks. <https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1686>

[19] MCP Specification 2025-11-25 — Basic Utilities: Tasks. <https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/tasks>

[20] MCP Specification 2025-11-25 — Client Features: Sampling. <https://modelcontextprotocol.io/specification/2025-11-25/client/sampling>

[21] MCP Specification 2025-11-25 — Client Features: Elicitation. <https://modelcontextprotocol.io/specification/2025-11-25/client/elicitation>

[22] MCP 2025-11-25 Spec Update (WorkOS blog) — notes SEP-1699 "Improved SSE polling and disconnect behavior". <https://workos.com/blog/mcp-2025-11-25-spec-update>

[23] Exploring the Future of MCP Transports (MCP Blog, Dec 2025). <https://blog.modelcontextprotocol.io/posts/2025-12-19-mcp-transport-future/>

[24] SEP-2575 — Make MCP Stateless. <https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2575>. See also SEP-2567 (Sessionless MCP via Explicit State Handles).

[25] SEP-2571 — Resource Submission: client-to-server resource creation for agent coordination. <https://github.com/modelcontextprotocol/modelcontextprotocol/issues/2571>

[26] modelcontextprotocol/servers — `src/everything/docs/features.md`: "Resource Subscriptions and Notifications" section. <https://github.com/modelcontextprotocol/servers/blob/main/src/everything/docs/features.md>

[27] Sample of Kafka MCP servers on GitHub:
- tuannvm/kafka-mcp-server <https://github.com/tuannvm/kafka-mcp-server>
- aswinayyolath/kafka-mcp-server <https://github.com/aswinayyolath/kafka-mcp-server>
- kanapuli/mcp-kafka <https://github.com/kanapuli/mcp-kafka>
- Joel-hanson/kafka-mcp-server <https://github.com/Joel-hanson/kafka-mcp-server>

[28] Introducing the StreamNative MCP Server. <https://streamnative.io/blog/introducing-the-streamnative-mcp-server-connecting-streaming-data-to-ai-agents>

[29] November 2025 spec merged SEPs (per WorkOS summary [22]): SEP-1024, SEP-835, SEP-1046, SEP-990, SEP-1036, SEP-1577, SEP-986, SEP-1319, SEP-1699, SEP-1686.
