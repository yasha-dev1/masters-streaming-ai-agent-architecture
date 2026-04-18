# Event Envelope Standards for Push-Based AI Agent Delivery

*Research note - push-protocol series, part 03*
*Author: Yasha Boroumand - masters thesis "Streaming AI Agent Architecture"*
*Date: 2026-04-18*

## Executive Summary

The Model Context Protocol (MCP) defines a `resources/subscribe` method and a `notifications/resources/updated` notification sent when the content of a subscribed resource changes [1]. The current schema is thin: a JSON-RPC 2.0 notification whose `params` carries little more than the resource `uri` [1][20]. This note surveys the envelope standards that could fill the gap.

**CloudEvents v1.0.2** (CNCF graduated, Jan 2024) is the only widely adopted envelope standard whose scope matches the problem: required attributes (`id`, `source`, `specversion`, `type`), optional attributes (`subject`, `time`, `datacontenttype`, `dataschema`), and extensions (`partitionkey`, `sequence`, `traceparent`, `dataref`, `recordedtime`) covering identity, routing, ordering, tracing, and payload-by-reference [2][3][4]. Bindings exist for HTTP, Kafka, MQTT, AMQP, NATS, and WebSockets - but **no JSON-RPC binding exists**, which is directly relevant to MCP [2][5].

**AsyncAPI 3.0** is the "OpenAPI for event-driven APIs" [6]. It does not define an envelope; it describes channels, messages, and send/receive operations, with bindings for Kafka, WebSocket, SSE/Mercure, HTTP, and others [7]. It can *reference* CloudEvents as the message schema [8]. AsyncAPI is the right layer for publishing an MCP push channel's contract.

**Standard Webhooks** (backers: Svix, Zapier, Twilio, Lob, Mux, ngrok, Supabase, Kong) focuses on signed, replay-protected HTTP delivery. It defines three headers (`webhook-id`, `webhook-timestamp`, `webhook-signature`), HMAC-SHA256 signing over `msg_id.timestamp.payload`, an optional ed25519 variant (`v1a`), a payload convention of `{type, timestamp, data}`, and a retry schedule [9][10]. It is the HTTP-callback half of what CloudEvents+AsyncAPI leave open.

**Adjacent standards**: AWS EventBridge has its own flatter envelope (`version`, `id`, `detail-type`, `source`, `account`, `time`, `region`, `resources`, `detail`) and does *not* adopt CloudEvents [11]. IETF `Idempotency-Key` draft-07 standardises the deduplication header [12][13]. RFC 9457 (Problem Details) [14] handles error semantics. OpenTelemetry event conventions [15] plus the CloudEvents `distributedtracing` extension [16] cover tracing. Schema registries (Confluent, EventBridge, CNCF xRegistry) sit above the envelope [17][18].

**Recommendation.** A three-layer stack: (1) keep MCP's JSON-RPC 2.0 `notifications/resources/updated` wire frame; (2) inside `params`, place a full CloudEvents v1.0 structured-mode object (a proposed `cloudevents-jsonrpc` binding), using `partitionkey`, `sequence`, `traceparent`, `dataref`, `recordedtime` extensions; (3) for the HTTP-callback variant, wrap the CloudEvent in Standard Webhooks headers with IETF `Idempotency-Key` for deduplication and RFC 9457 for errors. Each channel's contract is published as an AsyncAPI 3.0 document referencing the CloudEvents schema. Sections 5 and 6 develop the composition and enumerate remaining gaps.

---

## Section 1 - CloudEvents v1.0.2

### 1.1 Status and scope

CloudEvents is a CNCF specification for describing event data in a common way [2]. v1.0.2 was released Feb 5, 2022; CloudEvents SQL v1 Jun 13, 2024; the project graduated in CNCF on Jan 25, 2024 [2][19]. Current `main` reads `1.0.3-wip`; on-the-wire `specversion` is `"1.0"` [3]. Adopters: Knative, Argo, Falco, Harbor, Serverless Workflow, Adobe I/O Events, Alibaba EventBridge, Azure Event Grid, Google Eventarc, IBM Cloud Code Engine [19]. **AWS EventBridge does not support CloudEvents** natively [19].

### 1.2 Required attributes [3]

- **`id`** - non-empty; producers MUST ensure `source` + `id` is unique.
- **`source`** - non-empty URI-reference identifying the context (absolute URIs recommended).
- **`specversion`** - `"1.0"`.
- **`type`** - non-empty; reverse-DNS prefix recommended (`com.example.resource.updated`).

### 1.3 Optional attributes

- **`datacontenttype`** - RFC 2046 media type for `data`.
- **`dataschema`** - URI of a schema `data` adheres to.
- **`subject`** - producer-scoped subject (natural place for the MCP resource URI).
- **`time`** - RFC 3339 timestamp of the occurrence.
- **`data`** - the payload.

Intermediaries MUST forward events of 64 KiB or less [3].

### 1.4 Content modes [3][5]

- **Structured** - whole event as one document (typically `application/cloudevents+json`); forwards hop-to-hop without re-encoding.
- **Binary** - attributes as `ce-*` protocol headers; `data` is the raw body with `Content-Type` = `datacontenttype`.
- **Batch** - `application/cloudevents-batch+json`; array of events sharing `specversion`.

### 1.5 Protocol bindings

The `cloudevents/bindings` directory of the spec repo contains [5]:

- `http-protocol-binding.md`
- `kafka-protocol-binding.md`
- `mqtt-protocol-binding.md`
- `amqp-protocol-binding.md`
- `nats-protocol-binding.md`
- `websockets-protocol-binding.md`

**There is no JSON-RPC binding.** For a thesis that extends MCP (which is JSON-RPC 2.0 at the wire level), this is a meaningful gap. Section 5 sketches what a minimal `cloudevents-jsonrpc` binding could look like.

### 1.6 Extensions

Extensions are optional top-level attributes [4]:

- **`partitionkey`** - partitioning / causal grouping (think Kafka partition key); may change or be dropped across hops.
- **`sequence`** - lexicographically-orderable string scoped to `source` (`"002" > "001"`; pad numeric sequences).
- **`traceparent`** (+ optional `tracestate`) - W3C Trace Context embedded in the event so tracing survives multi-hop forwarding independently of per-protocol headers.
- **`dataref`** - URI-reference to externally stored payload (claim-check); `data` and `dataref` MAY coexist but MUST be identical.
- **`recordedtime`** - when the producer recorded the event, distinct from `time` (when the occurrence happened); enables bitemporal modelling.
- Others in the extensions tree: `authcontext`, `bam`, `dataclassification`, `deprecation`, `expirytime`, `opcua`, `sampledrate` / `rate`, `severity` [4].

### 1.7 A CloudEvent in JSON (structured mode)

Taken from the spec and minimally adapted [20]:

```json
{
  "specversion": "1.0",
  "type": "io.modelcontextprotocol.resource.updated",
  "source": "/mcp-servers/calendar-42",
  "subject": "calendar://user/123/events/evt_2024-04-18_0900",
  "id": "01HV9S4M2XW4A4GP7R2F9Q3G5T",
  "time": "2026-04-18T09:00:00.123Z",
  "datacontenttype": "application/json",
  "dataschema": "https://example.org/schemas/calendar-event/v1.json",
  "partitionkey": "user/123",
  "sequence": "0000000000000427",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "recordedtime": "2026-04-18T09:00:00.145Z",
  "data": {
    "title": "Thesis committee meeting",
    "start": "2026-04-18T14:00:00Z",
    "end":   "2026-04-18T15:00:00Z",
    "status": "confirmed"
  }
}
```

Binary-mode over HTTP would put everything except `data` into `ce-*` headers and send the inner `data` object as the raw JSON body with `Content-Type: application/json` [5].

### 1.8 Schema registry integrations

CloudEvents itself is envelope-only; schemas for `data` are external. The CNCF-aligned **xRegistry** project was originally part of CloudEvents and split off in April 2023; it defines a REST-based registry API with a Schema Registry subtype for payload schemas (JSON Schema, Avro, Protobuf), plus Message Definitions and Endpoints Registries [17][18]. AWS EventBridge Schema Registry supports OpenAPI 3 and JSON Schema Draft 4 [21]; Confluent Schema Registry covers Avro, Protobuf, and JSON Schema with BACKWARD as default compatibility [22]. xRegistry is the one aligned with CloudEvents.

---

## Section 2 - AsyncAPI 3.0

### 2.1 What it is

AsyncAPI is a machine-readable spec for message-driven APIs, modelled on OpenAPI [6][8]. The 3.0.0 release (Dec 2023) restructured the document around **Channels**, **Messages**, and **Operations**, with operations declaring `action: send` or `action: receive` and first-class request/reply [6][23].

### 2.2 Core objects

- **Channel** - an addressable component (Kafka topic, MQTT topic, WebSocket connection). For MCP, a channel corresponds to a subscribed resource URI.
- **Message** - has `payload` (a schema), `headers`, `contentType`. `schemaFormat` makes Avro, Protobuf, and **CloudEvents** first-class.
- **Operation** - `send` or `receive`, bound to a channel, referencing messages. `reply` describes request/reply.

### 2.3 Protocol bindings

From `asyncapi/bindings` [7]: Kafka, AMQP, MQTT, NATS, RabbitMQ, IBM MQ, Solace, Google Cloud Pub/Sub, AWS SNS/SQS, WebSockets, Mercure (SSE), HTTP, JMS, Redis, STOMP. The WebSocket binding treats the channel as the connection and has no sub-channels [24].

### 2.4 Referencing CloudEvents

The AsyncAPI community position [8] is that AsyncAPI and CloudEvents are complementary: CloudEvents specifies the event, AsyncAPI specifies the API around it. Two integration modes are documented:

- **Structured-mode CloudEvents** - the message `payload` uses `schemaFormat: 'application/cloudevents+json; version=1.0'` (or references a JSON Schema for the CloudEvent).
- **Binary-mode CloudEvents** - context attributes are described as `ce-*` message `headers` and `payload` only describes `data`. The AsyncAPI `MessageTrait` mechanism fits the CloudEvents header set, so it is declared once and inherited per message.

### 2.5 Describing an MCP push channel

No authoritative AsyncAPI document for MCP is published today [25]. The thesis can propose the first one; a skeleton for a single resource subscription:

```yaml
asyncapi: 3.0.0
info:
  title: MCP Push Channel - calendar-42
  version: 1.0.0
  description: |
    Push updates for resources exposed by an MCP server, delivered as
    JSON-RPC 2.0 notifications/resources/updated whose `params.event` is
    a CloudEvents v1.0 object.

servers:
  mcp-session:
    host: calendar-42.example.org
    protocol: wss
    pathname: /mcp

channels:
  resourceUpdated:
    address: notifications/resources/updated
    messages:
      resourceUpdatedMessage:
        $ref: '#/components/messages/ResourceUpdated'

operations:
  receiveResourceUpdate:
    action: receive
    channel:
      $ref: '#/channels/resourceUpdated'
    messages:
      - $ref: '#/channels/resourceUpdated/messages/resourceUpdatedMessage'

components:
  messages:
    ResourceUpdated:
      name: resourceUpdated
      title: Resource updated notification
      contentType: application/json
      payload:
        type: object
        required: [jsonrpc, method, params]
        properties:
          jsonrpc: { const: "2.0" }
          method:  { const: "notifications/resources/updated" }
          params:
            type: object
            required: [uri, event]
            properties:
              uri:
                type: string
                description: The MCP resource URI that changed.
              event:
                # reference the CloudEvents v1.0 JSON schema
                schemaFormat: 'application/cloudevents+json; version=1.0'
                $ref: 'https://github.com/cloudevents/spec/raw/v1.0.2/cloudevents/formats/cloudevents.json'
```

This is the simplest possible shape: the JSON-RPC frame is kept, the change-describing envelope is a CloudEvent, and the AsyncAPI document is the published contract.

---

## Section 3 - Standard Webhooks

### 3.1 Scope

Standard Webhooks (standardwebhooks.com) is a spec plus open-source libraries for "easy, secure, reliable" webhooks [9]. The technical steering committee draws on Zapier, Twilio, Lob, Mux, ngrok, Supabase, Svix (Tom Hacohen, CEO), and Kong [9]. The spec positions itself as complementary to OpenAPI, AsyncAPI, CloudEvents, and IETF HTTP Message Signatures [10]. Adoption tracks the Svix ecosystem - OpenAI, Brex, and other Svix customers follow these conventions in practice.

### 3.2 Headers

Three mandatory request headers [10]:

- `webhook-id` - unique message identifier (e.g. `msg_2KWPBgLlAfxdpx2AI54pPJ85f4W`).
- `webhook-timestamp` - integer Unix timestamp in seconds.
- `webhook-signature` - space-delimited list of signatures, each prefixed with a version identifier (`v1,<base64>` for symmetric; `v1a,<base64>` for ed25519 asymmetric).

### 3.3 Symmetric signing

The signing input is the concatenation `msg_id.timestamp.payload` where `payload` is the raw request body [10]:

```
msg_2KWPBgLlAfxdpx2AI54pPJ85f4W.1674087231.{"type":"contact.created","timestamp":"2022-11-03T20:26:10.344522Z","data":{"id":"1f81eb52-5198-4599-803e-771906343485"}}
```

The signature algorithm is HMAC-SHA256; the secret is 24-64 bytes, base64-encoded with the prefix `whsec_`. The resulting signature is base64 and goes into `webhook-signature` with the `v1,` prefix.

### 3.4 Asymmetric signing

The `v1a` variant uses ed25519 [10]. Secret keys use `whsk_`, public keys `whpk_`, both base64. The producer signs `msg_id.timestamp.payload` with its private key; consumers verify with the publicly distributed `whpk_...`. This matters for multi-consumer broadcast and compliance scenarios where the consumer must not be able to forge new messages.

### 3.5 Replay protection

Consumers MUST verify `webhook-timestamp` is within tolerance of current time, and SHOULD persist `(webhook-id, webhook-timestamp)` tuples to reject duplicates within the window [10]. The spec does not mandate a tolerance; 5 minutes is the Svix-derived default.

### 3.6 Delivery semantics

Success is any 2xx; recommended timeout 15-30 seconds. The recommended retry schedule is immediate, 5s, 5m, 30m, 2h, 5h, 10h, 14h, 20h, 24h (10 attempts, ~3-day total window) with random jitter [10].

### 3.7 Payload convention

The spec recommends but does not mandate [10]:

```json
{
  "type": "contact.created",
  "timestamp": "2022-11-03T20:26:10.344522Z",
  "data": {
    "id": "1f81eb52-5198-4599-803e-771906343485"
  }
}
```

`type` is full-stop-delimited; `timestamp` is ISO 8601. Payloads should stay under 20 KB so consumers can pre-filter without parsing. **This shape is almost a CloudEvent** (`type`, `time`, `data`); substituting a full CloudEvent is a trivial superset that adds `id`, `source`, `subject`, `specversion`, extensions, and a schema story.

### 3.8 Full Standard Webhooks signed request

```
POST /webhooks/mcp-calendar HTTP/1.1
Host: agent.example.org
Content-Type: application/cloudevents+json
webhook-id: msg_2KWPBgLlAfxdpx2AI54pPJ85f4W
webhook-timestamp: 1713434400
webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4=
Content-Length: 512

{
  "specversion": "1.0",
  "type": "io.modelcontextprotocol.resource.updated",
  "source": "/mcp-servers/calendar-42",
  "subject": "calendar://user/123/events/evt_2024-04-18_0900",
  "id": "01HV9S4M2XW4A4GP7R2F9Q3G5T",
  "time": "2026-04-18T09:00:00.123Z",
  "datacontenttype": "application/json",
  "partitionkey": "user/123",
  "sequence": "0000000000000427",
  "data": { "title": "Thesis committee meeting", "status": "confirmed" }
}
```

`webhook-id` is *distinct* from the CloudEvent `id`: the former identifies the *delivery attempt* for replay protection, the latter the *event*. Multiple retries of the same CloudEvent share `id` but each gets a fresh `webhook-id`.

---

## Section 4 - Adjacent standards

### 4.1 AWS EventBridge event structure

EventBridge uses a custom flatter envelope [11][26]:


```json
{
  "version": "0",
  "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
  "detail-type": "EC2 Instance State-change Notification",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2017-12-22T18:43:48Z",
  "region": "us-east-1",
  "resources": ["arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"],
  "detail": { "instance-id": "i-1234567890abcdef0", "state": "terminated" }
}
```

The `source` + `detail-type` pair mirrors CloudEvents' `source` + `type`; `detail` mirrors `data`. EventBridge does *not* support CloudEvents natively and has no `specversion`-style extensibility [19]. Its Schema Registry supports OpenAPI 3 and JSON Schema Draft 4 with automatic discovery [21]. For the thesis, EventBridge is useful as evidence that the largest production event bus chose *not* to adopt CloudEvents.

### 4.2 OpenTelemetry event conventions

OpenTelemetry models Events as `LogRecord`s with a mandatory `event.name` and an evolving library of namespaced definitions [15]. There is no `event.domain` attribute in current general-events conventions [15]. OTel meets CloudEvents at tracing: the `distributedtracing` extension carries W3C Trace Context (`traceparent`, `tracestate`) inside the envelope so spans survive multi-hop forwarding where per-protocol headers are stripped [16]. The thesis should treat `traceparent` as mandatory-in-practice for agent-to-agent flows.

### 4.3 Idempotency-Key (IETF draft)

`draft-ietf-httpapi-idempotency-key-header` (currently -07) standardises a request header that marks non-idempotent methods (POST, PATCH) as safely retryable [12][13]. The header is an RFC 8941 Item Structured Header whose value is a string (UUIDs recommended). The server fingerprints the key plus optional body checksum; duplicates return the cached response, mid-flight duplicates MUST return 409 [13]. Resources SHOULD publish an expiration policy.

This complements Standard Webhooks: `Idempotency-Key` protects the *consumer* from acting twice on retried POSTs; `webhook-signature` + `webhook-timestamp` guard against *forgery* and *replay*. Orthogonal concerns, both belong in a serious push spec.

### 4.4 RFC 9457 Problem Details

RFC 9457 (April 2023, obsoletes RFC 7807) defines `application/problem+json` for machine-readable HTTP error responses [14]:

```
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "https://example.com/probs/out-of-credit",
  "title": "You do not have enough credit.",
  "detail": "Your current balance is 30, but that costs 50.",
  "instance": "/account/12345/msgs/abc",
  "balance": 30,
  "accounts": ["/account/12345", "/account/67890"]
}
```

For push delivery, Problem Details is the right format for a consumer to return when it rejects a webhook (signature failed, idempotency collision, schema violation) - richer than a bare 4xx and uniform across providers.

### 4.5 Schema registries and wire formats

- **Confluent Schema Registry** - Avro, Protobuf, JSON Schema; default BACKWARD; BACKWARD_TRANSITIVE recommended for Protobuf [22].
- **AWS EventBridge Schema Registry** - OpenAPI 3 + JSON Schema Draft 4 with auto-discovery [21].
- **CNCF xRegistry** - successor to CloudEvents' registry work; separately-specced Schema, Message Definitions, and Endpoints Registries; can be a JSON file, static file server, or REST API [17][18].

Avro and Protobuf *complement* rather than compete - CloudEvents has bindings for both as `data` encodings. Avro/Protobuf add compact wire format and stricter compatibility at the cost of tooling. For agent push, JSON Schema + CloudEvents JSON is the path of least resistance; Avro/Protobuf are defensible for high-volume metric-style streams.

### 4.6 Watermarks

No IETF or CNCF standard defines a watermark envelope. Semantics come from Apache Beam, Flink, and Google Cloud Dataflow: a watermark `W(t)` asserts the stream is probably complete up to event-time `t` [27][28]. Flink propagates watermarks inline with data, preserving order [28]. An agent that needs time-windowed computation needs a watermark; CloudEvents `sequence` gives per-source ordering but not completeness. A `watermark` extension or a separate `notifications/resources/watermark` method are both credible answers; neither is standardised.

---

## Section 5 - Proposed composition for the thesis

The thesis's push protocol can be built entirely from existing standards, composed as follows.

### 5.1 The stack

```
+-----------------------------------------------------------+
|  Contract:  AsyncAPI 3.0 document (per resource channel)   |
+-----------------------------------------------------------+
|  Wire (in-session):   JSON-RPC 2.0 notification            |
|                       method = notifications/resources/updated
|                       params = { uri, event: <CloudEvent> }|
+-----------------------------------------------------------+
|  Wire (callback):     HTTP POST  application/cloudevents+json
|                       + Standard Webhooks headers          |
|                       + Idempotency-Key                    |
+-----------------------------------------------------------+
|  Envelope:  CloudEvents v1.0 structured-mode JSON          |
|             with extensions: partitionkey, sequence,       |
|             traceparent, dataref, recordedtime             |
+-----------------------------------------------------------+
|  Payload:   JSON Schema (or Avro/Protobuf via dataref) +   |
|             schema discoverable in xRegistry or equivalent |
+-----------------------------------------------------------+
```

### 5.2 What each layer buys

| Layer | Standard | Solves | Invented |
|-------|----------|--------|----------|
| Contract | AsyncAPI 3.0 | Machine-readable channel docs, codegen | - |
| In-session wire | MCP + JSON-RPC 2.0 | Reuses MCP transports (stdio, Streamable HTTP, SSE) [29][30] | `cloudevents-jsonrpc` binding |
| Callback wire | Standard Webhooks | Signing, replay protection, retries | - |
| Duplicates | IETF Idempotency-Key | Safe POST retries | - |
| Errors | RFC 9457 | Uniform failure responses | - |
| Envelope | CloudEvents v1.0.2 | id/source/type/time, tracing, ordering, claim-check | - |
| Payload schema | JSON Schema + xRegistry | Evolution, discovery | MCP <-> xRegistry mapping |

### 5.3 Concrete end-to-end example

**In-session push** (MCP client with an open Streamable HTTP session):

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "calendar://user/123/events/evt_2024-04-18_0900",
    "event": {
      "specversion": "1.0",
      "type": "io.modelcontextprotocol.resource.updated",
      "source": "/mcp-servers/calendar-42",
      "subject": "calendar://user/123/events/evt_2024-04-18_0900",
      "id": "01HV9S4M2XW4A4GP7R2F9Q3G5T",
      "time": "2026-04-18T09:00:00.123Z",
      "datacontenttype": "application/json",
      "dataschema": "https://registry.example.org/schemas/calendar-event/v1.json",
      "partitionkey": "user/123",
      "sequence": "0000000000000427",
      "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
      "data": {
        "title": "Thesis committee meeting",
        "start": "2026-04-18T14:00:00Z",
        "end":   "2026-04-18T15:00:00Z",
        "status": "confirmed"
      }
    }
  }
}
```

**Out-of-session push** (agent registered an HTTP webhook):

```
POST /webhooks/mcp HTTP/1.1
Host: agent.example.org
Content-Type: application/cloudevents+json
Idempotency-Key: 01HV9S4M2XW4A4GP7R2F9Q3G5T
webhook-id: msg_2KWPBgLlAfxdpx2AI54pPJ85f4W
webhook-timestamp: 1713434400
webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4=

{
  "specversion": "1.0",
  "type": "io.modelcontextprotocol.resource.updated",
  "source": "/mcp-servers/calendar-42",
  "subject": "calendar://user/123/events/evt_2024-04-18_0900",
  "id": "01HV9S4M2XW4A4GP7R2F9Q3G5T",
  "time": "2026-04-18T09:00:00.123Z",
  "data": { "title": "Thesis committee meeting", "status": "confirmed" }
}
```

The CloudEvent `id` is deliberately reused as `Idempotency-Key`: retries get a new `webhook-id` (so the HMAC differs, as it must since `webhook-timestamp` changes) but carry the same `Idempotency-Key`, so the consumer deduplicates end-to-end. A rejected delivery returns:

```
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://agent.example.org/probs/schema-mismatch",
  "title": "Event payload did not match registered schema.",
  "status": 422,
  "detail": "Required field 'start' missing in data.",
  "instance": "/webhooks/mcp/msg_2KWPBgLlAfxdpx2AI54pPJ85f4W",
  "cloudevent-id": "01HV9S4M2XW4A4GP7R2F9Q3G5T"
}
```

### 5.4 The proposed "cloudevents-jsonrpc" binding

The thesis can propose a minimal CloudEvents binding for JSON-RPC 2.0 mirroring the structured-mode HTTP binding:

- **Structured mode** - the CloudEvent JSON object appears verbatim under a designated key in `params` (here `params.event`). `params` MAY carry JSON-RPC-native fields (here `uri`). No attribute rewriting.
- **Binary mode** - context attributes appear as named members of `params` (e.g. `params.ce_id`, `params.ce_source`), with `params.data` as the raw `data` value. Useful when the consumer wants flat attribute access.
- **Batch mode** - not used; JSON-RPC 2.0 already provides batch at the outer level.

This is a two-page spec and, unlike the WebSocket binding, would be directly useful to the MCP community - exactly the intervention a masters thesis can make.

---

## Section 6 - Gaps and open questions

1. **No CloudEvents JSON-RPC binding** exists today [5]. Either the thesis proposes one (5.4) or keeps to a de-facto convention inside `params`. The former is more valuable.

2. **MCP has no published AsyncAPI document.** The `modelcontextprotocol` org ships TypeScript + generated JSON Schema [1]. AsyncAPI-ising MCP's push channels is straightforward but needs upstream buy-in to become authoritative.

3. **Schema registry binding.** `dataschema` is a URI; nothing prescribes *how* to resolve it. xRegistry is the obvious answer for new work [17][18], but MCP clients need either xRegistry awareness or an in-band schema-publication mechanism.

4. **Watermark semantics are unstandardised.** For time-windowed agent computations, neither CloudEvents, AsyncAPI, nor Standard Webhooks helps - a genuine contribution opportunity [27][28].

5. **Delivery semantics.** Standard Webhooks specifies at-least-once + idempotency *encouragement*; IETF `Idempotency-Key` provides the mechanism. Exactly-once *transport* is not specified by any of these and is arguably impossible; the stack buys effectively-once via at-least-once + consumer idempotency.

6. **Subscription lifecycle.** MCP's `resources/subscribe` returns no subscription object, lease, or resumption cursor. A CloudEvents envelope doesn't change that. Durable multi-day subscriptions need a lease + replay-cursor model that is not in any of the standards reviewed.

7. **Ordering across sources.** CloudEvents `sequence` is explicitly source-scoped [31]. Global ordering in multi-source workflows needs Kafka-style partition-keyed total order or a causal-time model (vector clocks, HLC). The thesis should pick one.

8. **Asymmetric key management.** Standard Webhooks v1a gives the primitive; key rotation, JWKS-like discovery, and per-consumer authorisation are not spec'd. A reference implementation will have to solve these.

9. **Binary wire formats.** Avro/Protobuf via `dataref` handles large data but loses self-description. For low-latency / high-volume pipelines this trade-off needs benchmarking.

10. **AWS EventBridge non-alignment.** The largest production event bus does not speak CloudEvents [19]. Realistic interop requires documenting the lossy translation between EventBridge and CloudEvents envelopes.

---

## References

[1] Model Context Protocol - Specification Schema, `modelcontextprotocol.io/specification/2025-11-25/schema` and `github.com/modelcontextprotocol/modelcontextprotocol`.

[2] CloudEvents - project homepage, `cloudevents.io`.

[3] CloudEvents Specification (main branch / v1.0.3-wip, v1.0 on-wire), `github.com/cloudevents/spec/blob/main/cloudevents/spec.md`.

[4] CloudEvents Extensions directory, `github.com/cloudevents/spec/tree/main/cloudevents/extensions`.

[5] CloudEvents Protocol Bindings directory (HTTP, Kafka, MQTT, AMQP, NATS, WebSockets), `github.com/cloudevents/spec/tree/main/cloudevents/bindings`.

[6] AsyncAPI 3.0.0 Specification, `asyncapi.com/docs/reference/specification/v3.0.0`.

[7] AsyncAPI Bindings repository, `github.com/asyncapi/bindings`.

[8] "AsyncAPI and CloudEvents", AsyncAPI Initiative blog, `asyncapi.com/blog/asyncapi-cloud-events`.

[9] Standard Webhooks homepage and steering committee, `standardwebhooks.com`.

[10] Standard Webhooks Specification (symmetric + asymmetric signatures, retry schedule, payload convention), `github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md`.

[11] AWS EventBridge event structure, `docs.aws.amazon.com/eventbridge/latest/userguide/eb-events-structure.html`.

[12] IETF HTTPAPI Working Group - Idempotency-Key draft index, `datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/`.

[13] `draft-ietf-httpapi-idempotency-key-header-07`, `datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header-07`.

[14] RFC 9457 - Problem Details for HTTP APIs, `rfc-editor.org/rfc/rfc9457`.

[15] OpenTelemetry - Semantic conventions for events, `opentelemetry.io/docs/specs/semconv/general/events/`.

[16] CloudEvents Distributed Tracing Extension, `github.com/cloudevents/spec/blob/main/cloudevents/extensions/distributed-tracing.md`.

[17] xRegistry specifications, `github.com/xregistry/spec` and `xregistry.io`.

[18] xRegistry Schema Registry specification, `xregistry.io/xreg/xregistryspecs/schema-v1/docs/spec.html` (and sibling Message Definitions, Endpoints).

[19] "Cloud Native Computing Foundation Announces the Graduation of CloudEvents", CNCF announcement, 25 January 2024, `cncf.io/announcements/2024/01/25/cloud-native-computing-foundation-announces-the-graduation-of-cloudevents/`.

[20] CloudEvents JSON Event Format, `github.com/cloudevents/spec/blob/main/cloudevents/formats/json-format.md`.

[21] AWS EventBridge Schema Registry, `docs.aws.amazon.com/eventbridge/latest/userguide/eb-schema-registry.html`.

[22] Confluent Schema Registry documentation, `docs.confluent.io/platform/current/schema-registry/index.html`.

[23] AsyncAPI 3.0.0 Release Notes, `asyncapi.com/blog/release-notes-3.0.0`.

[24] AsyncAPI WebSockets binding, `github.com/asyncapi/bindings/blob/master/websockets/README.md`.

[25] Background: "AsyncAPI, CloudEvents, OpenTelemetry: Which Event-Driven Specs Should Your DevOps Include?", AsyncAPI Initiative, `asyncapi.com/blog/async_standards_compare`.

[26] AWS EventBridge events reference, `docs.aws.amazon.com/eventbridge/latest/userguide/event-reference.html`.

[27] Begoli et al., "Watermarks in Stream Processing Systems: Semantics and Comparative Analysis of Apache Flink and Google Cloud Dataflow", PVLDB 14(12), 2021, `vldb.org/pvldb/vol14/p3135-begoli.pdf`.

[28] Apache Flink - Timely Stream Processing, `nightlies.apache.org/flink/flink-docs-stable/docs/concepts/time/`.

[29] MCP Transports (Streamable HTTP + SSE), `modelcontextprotocol.io/specification/2025-03-26/basic/transports`.

[30] "Why MCP Deprecated SSE and Went with Streamable HTTP", fka.dev blog, 6 June 2025, `blog.fka.dev/blog/2025-06-06-why-mcp-deprecated-sse-and-go-with-streamable-http/`.

[31] CloudEvents Sequence Extension, `github.com/cloudevents/spec/blob/main/cloudevents/extensions/sequence.md`.
