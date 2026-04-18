# 02 - A2A and the Non-MCP Agent Push Landscape

*Research note for: Streaming context for AI agents — push-to-agent communication standards.*
*Scope: Everything except MCP extensions (covered separately). Focus on whether any existing agent protocol natively supports "upstream event source pushes to long-running agent."*

---

## Executive Summary

The non-MCP agent-communication landscape in 2025/2026 is dominated by a handful of proposals, of which exactly one is a true standard-in-progress: **Google's Agent2Agent (A2A) protocol**, donated to the Linux Foundation in June 2025 and now backed by 150+ organizations. Secondary efforts include IBM's **ACP** (folded into A2A under LF in 2025), Cisco-led **AGNTCY / SLIM / Agent Connect Protocol**, and vendor-platform-specific mechanisms in LangGraph Platform, OpenAI's Realtime/Responses APIs, Letta, LlamaIndex Workflows, Temporal, and Inngest.

The key finding for the thesis question — *can any of these handle "upstream data system pushes an event to a long-running agent"?* — is **no, not directly, and not as a first-class concept.** The patterns that come closest are:

1. **A2A `tasks/pushNotificationConfig/set`** — but this is *agent-to-client* push (status/artifact updates flowing back to whoever initiated the task), not *external-system-to-agent* push. It assumes an agent has already been invoked; it does not solve the "wake up an idle agent when Kafka has a new event" problem.
2. **A2A SSE `message/stream` / `tasks/resubscribe`** — streams *from* the agent to the client, again backward from what we want.
3. **LangGraph `interrupt()` + `Command(resume=...)`** — pauses a graph and resumes on external input, but resumption is strictly *synchronous* (you call `.invoke()` again). No native webhook trigger for resume.
4. **Temporal Signals, Inngest events** — genuinely bidirectional external-push, but these are *workflow orchestrators* that people build agents on top of, not agent-communication standards.
5. **OpenAI Realtime API, Gemini Live API** — persistent WebSocket sessions, but scoped to audio/voice turn-taking and capped at ~10-minute connection lifetimes (with session-resumption tokens). Not designed for "agent subscribed to Kafka topic for a week."

What is **standardized**: agent-to-agent task invocation (A2A), agent discovery (Agent Cards, OASF), webhook delivery semantics from *agent back to caller*. What is **ad hoc**: the entire upstream-system-to-agent direction. In production, everyone glues together webhook → queue → worker-that-invokes-agent. There is no de-facto standard envelope, no standard subscription primitive, no standard "agent listens on this topic" contract.

This is the gap that MCP-extension work (the other thesis track) could plausibly fill.

---

## Section 1: A2A Protocol Deep Dive

### 1.1 History and Governance

Google announced A2A in April 2025 alongside ~50 partners, positioning it as the complement to Anthropic's MCP: where MCP standardizes *agent-to-tool* communication, A2A standardizes *agent-to-agent* communication. On **June 23, 2025**, Google Cloud donated the spec, SDKs, and developer tooling to the Linux Foundation, forming the Agent2Agent Protocol Project with AWS, Cisco, Google, Microsoft, Salesforce, SAP, and ServiceNow as founding members ([Linux Foundation press release](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents); [Google Developers Blog](https://developers.googleblog.com/en/google-cloud-donates-a2a-to-linux-foundation/)). IBM's ACP was folded into A2A under the same LF umbrella later in 2025 ([agentcommunicationprotocol.dev](https://agentcommunicationprotocol.dev/introduction/welcome) notes "ACP is now part of A2A under the Linux Foundation"). As of the v0.3 release (mid-2025), A2A has 150+ supporting organizations ([Google Cloud blog](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade)).

### 1.2 Core Model: Client Agent, Remote Agent, Task, Message, Artifact

A2A is a JSON-RPC 2.0 protocol (with an added gRPC binding in v0.3) between a **Client Agent** (requester) and a **Remote Agent / A2A Server** (provider) ([A2A spec](https://a2a-protocol.org/latest/specification/)). Key objects:

- **Agent Card** — a JSON metadata document (served at a well-known URL) advertising the agent's identity, skills, endpoint, security schemes, and capability flags including `streaming: true` and `pushNotifications: true`. Authenticated clients can fetch an *Extended Agent Card* via the `agent/authenticatedExtendedCard` HTTP endpoint for additional skills.
- **Task** — the stateful unit of work. States: `SUBMITTED`, `WORKING`, `INPUT_REQUIRED`, `COMPLETED`, `FAILED`, `CANCELED`, `REJECTED`, `AUTH_REQUIRED`. Each Task has a server-generated `id` and an optional `contextId` that logically groups related tasks and messages into a conversational session.
- **Message** — a single turn, with `role` (USER or AGENT), a `messageId`, and a `parts[]` array containing text, bytes, URLs, or structured data.
- **Artifact** — a Task output, again composed of parts.

The critical framing: **the "client" in A2A is explicitly another agent or agent-like application.** The [IBM overview](https://www.ibm.com/think/topics/agent2agent-protocol) is emphatic: "The A2A client, also known as the client agent, can be an app, service or other AI agent that delegates requests to remote agents." Nothing in the spec forbids a non-agent client — JSON-RPC is just JSON-RPC — but the entire conceptual model is *one agentic entity asks another agentic entity to do work*. There is no "event source" or "topic" primitive.

### 1.3 The Three Interaction Modalities

A2A's [README on the project repo](https://github.com/a2aproject/A2A) explicitly enumerates three modalities: "Flexible Interaction: Supports synchronous request/response, streaming (SSE), and asynchronous push notifications." These map to distinct JSON-RPC methods ([v0.2.5 spec](https://a2a-protocol.org/v0.2.5/specification/)):

| Method | Direction | Transport | Use |
|---|---|---|---|
| `message/send` | client → server | HTTP POST, sync | Fire-and-wait |
| `message/stream` | client → server, events server → client | SSE over HTTP | Real-time incremental |
| `tasks/get` | client → server | HTTP POST, sync | Poll status |
| `tasks/cancel` | client → server | HTTP POST, sync | Abort |
| `tasks/resubscribe` | client → server, resumes SSE | SSE over HTTP | Reconnect after drop |
| `tasks/pushNotificationConfig/set` | client → server | HTTP POST | Register webhook |
| `tasks/pushNotificationConfig/get` | client → server | HTTP POST | Read config |
| `tasks/pushNotificationConfig/list` | client → server | HTTP POST | Enumerate configs |
| `tasks/pushNotificationConfig/delete` | client → server | HTTP POST | Remove webhook |

All method names use forward slashes by convention ("tasks/pushNotificationConfig/set"), not the dotted style common elsewhere.

### 1.4 Push Notifications: The Mechanism in Detail

This is the mechanism closest to the thesis question, so it merits careful reading. From the [A2A streaming & async docs](https://a2a-protocol.org/latest/topics/streaming-and-async/):

> "Push Notifications for Disconnected Scenarios. Purpose: Enable asynchronous updates for very long-running tasks or clients unable to maintain persistent connections."

**The flow is:**

1. The *client agent* calls `message/send` (or `message/stream`) to initiate a Task. The request may embed a `pushNotificationConfig`, or the client can call `tasks/pushNotificationConfig/set` separately.
2. The `PushNotificationConfig` object contains:
   - `url` — the client's HTTPS webhook endpoint
   - `token` — optional opaque value the client can use to validate delivery
   - `authentication` — a `PushNotificationAuthenticationInfo` structure describing how the *A2A Server* should authenticate *to the client's webhook* (Bearer, Basic, API key, or a JWT/JWKS flow)
3. As the Task progresses, the A2A Server POSTs `TaskStatusUpdateEvent` or `TaskArtifactUpdateEvent` payloads (inside a `StreamResponse` envelope) to the client's webhook URL — but only on significant transitions (terminal state, input-required, auth-required), not every artifact chunk.
4. On receiving a notification, the client typically calls `tasks/get` to retrieve the full updated Task.

**JWT + JWKS flow** (per the spec's security guidance):

1. Client registers `authentication.scheme: "Bearer"` with an expected issuer/audience.
2. A2A Server signs a JWT (claims: `iss`, `aud`, `iat`, `exp`, `jti`, `taskId`) with its private key.
3. Client's webhook validates the signature against the server's JWKS endpoint and checks claims.
4. Webhook must implement replay prevention (timestamps, nonce tracking via `jti`).

**The direction this serves.** This is asymmetric and very specific: it is the *server agent* pushing *task-progress updates* back to the *client agent* that originated the task. It's a callback for an in-flight job, not a subscription to an event source. The client agent must have already invoked the server agent and registered interest in *that specific taskId*. There is no way in A2A for, say, a Kafka topic or a Stripe webhook to push a new event into an otherwise-idle agent. Such an external system would have to act as a *client agent* and call `message/send` itself — turning every upstream event source into an A2A client, which is a significant architectural imposition (it would need to speak JSON-RPC, authenticate with a security scheme from the AgentCard, manage Task lifecycles, etc.).

### 1.5 SSE Streaming and `tasks/resubscribe`

`message/stream` opens an HTTP connection that the server holds open with `Content-Type: text/event-stream`. Each SSE event's data is a JSON-RPC 2.0 Response object containing a `Task`, `TaskStatusUpdateEvent`, or `TaskArtifactUpdateEvent`. The stream terminates when the task hits a terminal state. If the connection drops, the client calls `tasks/resubscribe` with the `taskId` to reconnect. The spec mandates events MUST be delivered in generation order, and multiple concurrent subscribers for the same task each receive the identical event sequence (enabling team collaboration and client reconnection).

Again: **this flows *from* the agent *to* the client.** SSE in A2A is not a mechanism for an agent to subscribe to an external event feed.

### 1.6 v0.3 Additions

The Google Cloud blog post [Agent2Agent protocol is getting an upgrade](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade) documents v0.3 improvements (mid-2025):

- **gRPC transport binding** added alongside JSON-RPC/HTTP.
- **Signed Agent Cards** for authenticity.
- **Python SDK** expansion and native integration into Google's Agent Development Kit (ADK).
- Deployment paths on Cloud Run, GKE, and the forthcoming Agent Engine.

None of these changes alter the fundamental direction: A2A remains agent-to-agent task delegation. There is no new "subscribe to external event stream" primitive.

### 1.7 Community Work: Kafka + A2A

Kai Waehner's widely-cited blog [Agentic AI with A2A and MCP using Apache Kafka as Event Broker](https://www.kai-waehner.de/blog/2025/05/26/agentic-ai-with-the-agent2agent-protocol-a2a-and-mcp-using-apache-kafka-as-event-broker/) proposes exactly what my earlier analysis implied: Kafka is used as the *underlying event broker*, and agents publish/consume via Kafka topics while using A2A/MCP for the *semantic structure* of those messages. Critically: **A2A itself does not define how an agent subscribes to a Kafka topic.** The Kafka-A2A integration is entirely out-of-band; A2A is just the wire format for messages that happen to be transported over Kafka. This is a pattern, not a standard.

### 1.8 Verdict on A2A for Upstream Push

A2A solves agent-to-agent delegation with real care for asynchrony (push callbacks, SSE streaming, task lifecycle). It does **not** address upstream-system-to-agent push. To shoehorn that use case into A2A today, you would:

- Wrap each upstream system in a synthetic "client agent" that translates events into `message/send` calls — heavy and impedance-mismatched (every Stripe webhook event becomes a new Task? a new Message on an existing context?).
- Or build an out-of-band transport (Kafka, NATS) and layer A2A message envelopes on top, which is what Waehner proposes — but then A2A is just a payload schema, not a protocol.

Neither is a satisfying standard. This is a real gap.

---

## Section 2: Other Agent Protocols and Platforms

### 2.1 AGNTCY / Agent Connect Protocol / SLIM (Cisco-led, LF)

AGNTCY launched March 2025 from Cisco Outshift with LangChain and Galileo; donated to the Linux Foundation July 2025 ([outshift.cisco.com](https://outshift.cisco.com/blog/building-the-internet-of-agents-introducing-the-agntcy); [LF press](https://www.linuxfoundation.org/press/linux-foundation-welcomes-the-agntcy-project-to-standardize-open-multi-agent-system-infrastructure-and-break-down-ai-agent-silos)). The stack has four pieces:

- **OASF** (Open Agent Schema Framework) — agent metadata schema
- **ADS** (Agent Directory Service) — discovery
- **Agent Connect Protocol (AConP / ACP)** — message-level protocol (spec at [spec.acp.agntcy.org](https://spec.acp.agntcy.org/))
- **SLIM** (Secure Low-latency Interactive Messaging) — transport layer

**SLIM is the most interesting part for push.** Per the [IETF draft](https://datatracker.ietf.org/doc/draft-mpsb-agntcy-slim/) and the [SLIM GitHub repo](https://github.com/agntcy/slim), SLIM extends gRPC over HTTP/2/3 with:

- Request/reply, fire-and-forget, **publisher/subscriber**, and group communication patterns
- End-to-end encryption via MLS (Message Layer Security)
- NATS transport support for pub/sub messaging
- Stream multiplexing and flow control

The **pub/sub pattern in SLIM is the closest thing in the agent-protocol space to an upstream-event-source-to-agent primitive.** An agent can subscribe to a logical topic; publishers can be anything that speaks SLIM. However: SLIM is network-transport level (think "gRPC++ with MLS"), not an agent-semantics layer. You get secure messaging and multicast, but nothing about *what* an agent should do with an incoming event (there's no equivalent of A2A's Task lifecycle for subscribed events). And SLIM is very new (v0.6.0 as of late 2025), adoption is nascent, and IETF drafts at the `-01` revision are far from being finalized standards.

**Verdict:** Promising direction (pub/sub is the right shape), but not a usable standard yet; no semantic agent model on top of it.

### 2.2 IBM ACP (BeeAI)

IBM Research announced ACP in March 2025 alongside the BeeAI project ([research.ibm.com/projects/agent-communication-protocol](https://research.ibm.com/projects/agent-communication-protocol)). ACP is REST/HTTP-based, supports sync/async/streaming, and the [WorkOS overview](https://workos.com/blog/ibm-agent-communication-protocol-acp) describes it as JSON-RPC over HTTP/WebSockets with "persistent contexts so a long-running planner agent can survive restarts." **ACP was folded into A2A under the LF umbrella in 2025** per [agentcommunicationprotocol.dev](https://agentcommunicationprotocol.dev/introduction/welcome): "ACP is now part of A2A under the Linux Foundation." The [4sysops comparison article](https://4sysops.com/archives/comparing-ai-protocols-mcp-a2a-agp-agntcy-ibm-acp-zed-acp/) confirms ACP-specific dev has largely ceased. It is no longer a live competitor.

### 2.3 LangGraph Platform

LangGraph Platform is LangChain's managed runtime for LangGraph agents. [Platform docs on webhooks](https://docs.langchain.com/langgraph-platform/use-webhooks) describe a two-directional picture:

- **Outbound webhooks** — when a background Run completes, LangGraph Platform POSTs the Run payload to a configured webhook URL. This is platform-to-external, not upstream-to-agent.
- **Triggers** — [per the Medium writeup](https://medium.com/@_Ankit_Malviya/building-event-driven-multi-agent-workflows-with-triggers-in-langgraph-48386c0aac5d), external triggers "respond to signals from outside the workflow—API calls, webhooks, message queues, or database changes." This is the closest LangGraph gets to push-to-agent, but it's a platform feature, not a protocol standard.
- **Background runs** — `/runs` with `webhook` param lets you kick off a long-running agent and be notified on completion.

LangGraph's `interrupt()` primitive (see Section 3) is *adjacent* to push but requires synchronous `.invoke(Command(resume=...))` — no wire-protocol standard for an external system to inject the resume.

### 2.4 OpenAI Assistants / Responses / Realtime APIs

- **Assistants API**: The community forum thread [Assistants API run needs webhook](https://community.openai.com/t/assistants-api-run-needs-webhook/484418) documents that as of late 2025, Assistants API runs are poll-only — you `create_run`, then repeatedly `retrieve_run` until status changes. No webhook for run completion.
- **Responses API / Batch / Fine-tuning**: OpenAI's [webhooks guide](https://platform.openai.com/docs/guides/webhooks) does offer webhooks for *completion events* on Batch jobs, Background Responses, and Fine-tuning jobs. This is analogous to A2A push — the agent (OpenAI-hosted) pushes back to you when a job finishes. Not a mechanism for you to push external events *into* an OpenAI-hosted agent.
- **Realtime API**: [Persistent WebSocket](https://developers.openai.com/api/docs/guides/realtime-websocket) with true bidirectional event streaming. Both sides can send at any time. This **is** a genuine push channel — but scoped to audio-first voice interaction, with connection lifetimes typically bounded by the conversation (minutes, not hours/days).

**Verdict:** OpenAI's agent surfaces have no general-purpose upstream-push primitive. Realtime is the closest thing but is domain-specific to voice.

### 2.5 Gemini Live API

[ai.google.dev/gemini-api/docs/live-api](https://ai.google.dev/api/live) — WebSocket endpoint (`wss://generativelanguage.googleapis.com/...BidiGenerateContent`), bidirectional, truly event-driven. Same bounded-session story as OpenAI Realtime: ~10-minute connection ceiling, with `SessionResumptionUpdate` tokens valid for 2 hours to reconnect into the same logical session. ADK automates the resumption glue. Good for "persistent voice conversation," structurally unsuitable for "agent subscribed to Kafka for days" without application-level orchestration.

### 2.6 Letta (formerly MemGPT)

Letta ([letta.com](https://letta.com/)) is about persistent stateful agents with memory hierarchies, not push protocols. Its contribution is the *agent-as-process* model: an agent has durable memory and can be reawakened. But Letta's [API docs](https://docs.letta.com/concepts/memgpt/) show invocation is RPC-style — external signals arrive via explicit message sends to the agent's endpoint. There's no documented standard for external event *subscription*; the platform exposes HTTP APIs that callers invoke.

### 2.7 LlamaIndex Workflows

LlamaIndex Workflows ([developers.llamaindex.ai/python/workflows-api-reference/workflow/](https://developers.llamaindex.ai/python/workflows-api-reference/workflow/)) is an *internal* event-driven framework: steps emit and consume typed events within the same workflow instance. The Workflow Context exposes a streaming queue that can be written to from outside the workflow — so in principle, external code can push events in via `ctx.send_event()`. But this is an in-process API, not a network protocol. No wire standard for remote event push.

### 2.8 Temporal and Inngest

These are durable-execution workflow engines, not agent protocols, but they are increasingly used *as the backbone* for production AI agents:

- **Temporal Signals** — [temporal.io/blog/orchestrating-ambient-agents-with-temporal](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal) — external systems inject events into a running Workflow via typed `Signal` methods. Workflows survive restarts, run for days/months. This is *exactly* the upstream-push semantic we want, but it is a workflow-engine feature not an agent standard: the Signal API is Temporal's, not portable across runtimes.
- **Inngest** — [inngest.com/docs/features/events-triggers](https://www.inngest.com/docs/features/events-triggers) — functions triggered by events (SDK send, webhook, cron, external system). "Any trigger, any code" is the pitch. Again, this is an orchestration-platform capability, not a cross-vendor agent protocol.

**Verdict:** These are the production answer for "upstream event → long-running agent" today — but they solve it at the orchestrator layer, not at the agent-protocol layer. An agent built on Temporal speaks "Signals"; an agent built on Inngest speaks "Events"; neither has a standard wire envelope for agents on the *other* platform.

---

## Section 3: Agent Lifecycle Models and External Signals

### 3.1 Task-Scoped vs Persistent

Agent lifecycle divides roughly into two camps:

| Model | Example | External signal mechanism |
|---|---|---|
| **Task-scoped / ephemeral** | LangChain AgentExecutor, OpenAI Assistants run, A2A Task, MCP tool call | Input provided at invocation; no mechanism mid-flight beyond HITL prompts |
| **Persistent / daemon** | Letta, Temporal workflow, LangGraph durable run, Slack bot with long-running listener | Polling, webhooks from source system, message queue consumer, or framework-specific signal API |

Task-scoped agents don't need a push-to-agent protocol — the task was kicked off with all needed context, and if new info arrives, the platform just starts a new task. Persistent agents are where the gap lives.

### 3.2 How Persistent Agents Receive External Signals Today

In practice, **persistent agents receive external signals via one of four mechanisms**, none of which is standardized across frameworks:

1. **Polling.** Agent periodically calls out to external systems (email, Jira, DB) and looks for new items. Simple, wasteful, laggy. No protocol needed but also no standardization.
2. **Webhook-to-queue-to-agent.** External system POSTs a webhook to a tiny HTTP receiver; receiver enqueues onto SQS/Kafka/Pub/Sub; a worker dequeues and invokes the agent. This is the overwhelmingly common production pattern (see Section 4). Every org rolls their own; the agent just sees a normal invocation.
3. **Framework-specific signal APIs.** Temporal Signals, Inngest events, LangGraph triggers. Standardized within one platform, not across.
4. **Persistent bidirectional connection.** OpenAI Realtime / Gemini Live WebSockets. Structurally capable of upstream push but bounded in duration and scope.

### 3.3 LangGraph `interrupt()`

Per the [LangChain interrupts docs](https://docs.langchain.com/oss/python/langgraph/interrupts):

```python
from langgraph.types import interrupt, Command
# Inside a node:
value = interrupt({"question": "Approve deployment?"})
# Graph pauses, state is checkpointed. Caller sees __interrupt__ event.
# Resume with:
graph.invoke(Command(resume=user_input), config={"thread_id": "..."})
```

Key properties:

- Pauses mid-node; the entire node re-executes on resume (important to know for side effects).
- State is checkpointed via the configured persistence backend (memory, Postgres, etc.); a paused graph survives process restarts.
- Resumption requires the **same `thread_id`** and an explicit `.invoke(Command(resume=...))` call.
- Input can be anything JSON-serializable; nothing mandates a human originates it — a webhook handler, another agent, or a message-queue consumer can trigger `.invoke()`.
- **But the resume mechanism itself is a synchronous Python call, not a protocol.** There is no standard wire format for "external system pushes resume value to a paused graph." LangSmith/LangGraph Platform adds a REST endpoint for this (POST to the run with a resume body), but it's platform-specific.

`interrupt()` is the right *shape* for push-to-agent (pause-and-wait-for-external), but it is not a protocol, and the resume trigger has no standard wire format across platforms.

---

## Section 4: Webhook-to-Agent Patterns in Production

Because none of the agent protocols solve this, production teams glue webhooks to agents using low-code platforms or roll-your-own backends. The dominant pattern, across n8n, Zapier, Make, Pipedream, and custom stacks:

```
[External System] ---webhook POST---> [Lightweight HTTP receiver]
                                             |
                                             v
                              [Queue: SQS / Kafka / Redis / etc.]
                                             |
                                             v
                              [Worker: dequeues, invokes LLM agent]
                                             |
                                             v
                                    [Agent response / action]
```

### 4.1 n8n / Zapier / Make

- **n8n** ([docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)) — Webhook Trigger node starts a workflow; an AI Agent node (LangChain-backed) processes it. Supports chaining, memory, tool nodes. No execution time limit. Most agentic flexibility.
- **Make / Zapier** — Simpler linear or router-based flows. Make supports multi-step AI workflows via HTTP/JSON modules. Zapier is simplest, limited branching.

**These platforms invent no protocol.** They're ETL-for-events with LLM nodes bolted on. Each one has its own webhook format, its own LLM-invocation convention, its own state model.

### 4.2 The Slack Production Pattern

The Slack case is worth calling out because Slack's webhooks require responses within **3 seconds**, which hard-forces the queue pattern. A canonical AWS writeup ([integrating-amazon-bedrock-agentcore-with-slack](https://aws.amazon.com/blogs/machine-learning/integrating-amazon-bedrock-agentcore-with-slack/)) splits this into three Lambdas:

1. Verification Lambda — validates Slack signature, returns 200 instantly.
2. SQS integration Lambda — filters and enqueues.
3. Agent Lambda — dequeues, invokes the agent, posts result back to Slack.

This pattern generalizes: every webhook-to-agent production system looks like this, with variations in queue technology and agent runtime.

### 4.3 Is There an Emerging Standard?

**No.** CloudEvents ([cloudevents.io](https://cloudevents.io/)) is a CNCF standard for event envelope format, and you see it used *between* queue hops, but it says nothing about agent invocation semantics. There is no "CloudEvents for AI agents" proposal as of April 2026. The webhook→queue→agent architecture is everyone's house pattern; nothing has canonicalized it.

---

## Section 5: Gaps, and Is Any of This a Viable Alternative to Extending MCP?

### 5.1 What's Actually Standardized

| Concern | Standardized? | Where |
|---|---|---|
| Agent-to-agent task invocation | Yes | A2A `message/send` |
| Agent-to-client progress callbacks | Yes | A2A `tasks/pushNotificationConfig/set` + SSE |
| Agent discovery | Yes | A2A Agent Cards, AGNTCY OASF |
| Agent pub/sub transport | Partially | AGNTCY SLIM (early, transport-only) |
| Upstream system → agent push | **No** | — |
| Durable agent state across events | Partially | Platform-specific (Temporal, LangGraph, Letta) |
| Cross-framework "subscribe to event" primitive | **No** | — |

### 5.2 The Core Gap

Every protocol reviewed assumes the **agent is the service** and **some client** (another agent, a user, an app) invokes it. The direction upstream → agent requires either:

1. The upstream system masquerades as a client agent (heavy, wrong abstraction for a Kafka topic).
2. Out-of-band transport (Kafka, NATS) with an application-level bridge (what Waehner proposes with Kafka + A2A payloads, what everyone actually does in production with webhook → queue → worker).

There is no protocol that says: *"I am agent X; here is my subscription to event type E from source S; please deliver events to me on this long-lived channel with these delivery guarantees, and I will remain subscribed across restarts."* That object — the **agent-side event subscription** — does not exist as a standardized primitive anywhere in the non-MCP landscape.

### 5.3 Skeptical Assessment of the Claims

Vendors frequently describe their agent protocols as "async" or "event-driven." Auditing each:

- **A2A "async push notifications"** — Asynchronous, yes; but directionally callbacks for in-flight tasks, not subscription to external events. Marketing glosses this.
- **LangGraph "event-driven agents"** — True of the internal graph execution model; false as a cross-system protocol statement.
- **Inngest / Temporal "event-driven AI agents"** — True within the platform; not a portable standard.
- **AGNTCY SLIM pub/sub** — Legitimate pub/sub at the transport layer, but no agent-semantic layer on top and not yet broadly deployed.
- **OpenAI Realtime / Gemini Live "persistent sessions"** — True but bounded (~10-minute connections with token-based resumption) and voice-scoped.

The phrase "agent-to-agent" is doing a lot of work in A2A. It means exactly what it says: agent-to-agent. Not event-source-to-agent.

### 5.4 Is MCP-extension the right path?

Given:

- A2A's pushNotificationConfig is the wrong direction (server→client, not event-source→agent);
- AGNTCY SLIM is transport-level and lacks an agent-subscription primitive;
- LangGraph interrupt()/Temporal Signals are platform APIs without wire standards;
- No other contender has a subscription primitive at all;

...**there is a genuine open standardization slot** for "upstream event → agent subscription." MCP's transport (stdio/HTTP+SSE in the current spec, streamable HTTP in newer versions) already has server-to-client notifications (`notifications/*`), which MCP uses today for *server* pushing to *client* (e.g., `notifications/resources/updated`). Extending MCP to support the *inverse* flow — or defining a new complementary protocol where the "upstream system" is an MCP-server-like entity pushing events into an agent-as-client — seems architecturally cleaner than trying to bend A2A into this shape, because:

1. MCP already has the server-push-to-client notification pattern.
2. MCP already has a subscription concept (`subscribe`/`unsubscribe` on resources).
3. MCP already has a well-defined client (the agent) that is long-lived relative to a tool-server.

The strongest non-MCP candidate to compete would be **AGNTCY SLIM + a new subscription-semantic layer on top** — but that's a much longer road than extending MCP, which already ships in every major agent framework.

---

## References

### Primary specs
- [A2A Protocol latest spec](https://a2a-protocol.org/latest/specification/) and [v0.2.5](https://a2a-protocol.org/v0.2.5/specification/)
- [A2A streaming & async docs](https://a2a-protocol.org/latest/topics/streaming-and-async/)
- [A2A GitHub repo](https://github.com/a2aproject/A2A)
- [AGNTCY docs](https://docs.agntcy.org/)
- [SLIM IETF draft](https://datatracker.ietf.org/doc/draft-mpsb-agntcy-slim/) and [GitHub](https://github.com/agntcy/slim)
- [Agent Connect Protocol spec](https://spec.acp.agntcy.org/)
- [IBM Agent Communication Protocol](https://agentcommunicationprotocol.dev/introduction/welcome)
- [Gemini Live API reference](https://ai.google.dev/api/live) and [session management](https://ai.google.dev/gemini-api/docs/live-api/session-management)
- [OpenAI Realtime WebSocket docs](https://developers.openai.com/api/docs/guides/realtime-websocket)
- [OpenAI Webhooks guide](https://platform.openai.com/docs/guides/webhooks)
- [LangChain interrupts docs](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph Platform use webhooks](https://docs.langchain.com/langgraph-platform/use-webhooks)
- [Inngest Events & Triggers](https://www.inngest.com/docs/features/events-triggers)
- [LlamaIndex Workflows](https://developers.llamaindex.ai/python/workflows-api-reference/workflow/)
- [Letta GitHub](https://github.com/letta-ai/letta) and [docs](https://docs.letta.com/concepts/memgpt/)

### Vendor announcements and analysis
- [Google Cloud donates A2A to Linux Foundation](https://developers.googleblog.com/en/google-cloud-donates-a2a-to-linux-foundation/)
- [LF press release on A2A](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents)
- [Agent2Agent is getting an upgrade (v0.3)](https://cloud.google.com/blog/products/ai-machine-learning/agent2agent-protocol-is-getting-an-upgrade)
- [LF press release on AGNTCY](https://www.linuxfoundation.org/press/linux-foundation-welcomes-the-agntcy-project-to-standardize-open-multi-agent-system-infrastructure-and-break-down-ai-agent-silos)
- [Outshift by Cisco: Introducing AGNTCY](https://outshift.cisco.com/blog/building-the-internet-of-agents-introducing-the-agntcy)
- [IBM: What is A2A](https://www.ibm.com/think/topics/agent2agent-protocol) and [What is ACP](https://www.ibm.com/think/topics/agent-communication-protocol)
- [WorkOS: IBM ACP technical overview](https://workos.com/blog/ibm-agent-communication-protocol-acp)
- [Temporal: Orchestrating ambient agents](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)
- [Kai Waehner: Agentic AI with A2A and MCP using Apache Kafka](https://www.kai-waehner.de/blog/2025/05/26/agentic-ai-with-the-agent2agent-protocol-a2a-and-mcp-using-apache-kafka-as-event-broker/)
- [4sysops: Comparing AI protocols (MCP, A2A, AGP, AGNTCY, ACP, Zed ACP)](https://4sysops.com/archives/comparing-ai-protocols-mcp-a2a-agp-agntcy-ibm-acp-zed-acp/)
- [AWS: Integrating Bedrock AgentCore with Slack](https://aws.amazon.com/blogs/machine-learning/integrating-amazon-bedrock-agentcore-with-slack/)
- [Building event-driven multi-agent workflows with LangGraph triggers](https://medium.com/@_Ankit_Malviya/building-event-driven-multi-agent-workflows-with-triggers-in-langgraph-48386c0aac5d)
