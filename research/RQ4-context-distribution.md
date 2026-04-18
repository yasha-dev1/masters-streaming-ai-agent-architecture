# RQ4 — Context Distribution to Agents: Pull, Push, or Hybrid?

> **Research question.** What is the right interaction model between the Context
> Engine and the agents — pull/retrieve, push/pub-sub, or hybrid? How should
> subscriptions be expressed?

---

## 1. Executive Summary

After surveying the pull-based RAG literature [1,2,19], the classical pub/sub
canon [3,6,7,12], agent-specific protocols (MCP [4,16], A2A [5]), agent
framework memory models (LangGraph [8], LlamaIndex [20], Letta/MemGPT [9],
AutoGen/CrewAI [14], Swarm [15]), and event-driven-RAG hybrids [18,21,22], the
recommendation is a **hybrid push-notify / pull-retrieve model** closely
aligned with the thesis's working hypothesis.

Concretely, the Context Engine should expose:

1. a **push/pub-sub notification channel** on the fast path, where agents
   register subscriptions expressed as *typed predicates over a common event
   envelope* (topic + content filter, à la EventBridge [17] / NATS subjects
   [13]), plus optionally a semantic/embedding predicate for future extension;
2. a **pull/retrieve API** on the batch path, in the form of an MCP-style
   resource surface [4,16] that agents query when they need aggregated
   context, shared memory, or archival recall [9,20].

Notifications carry only a lightweight *change signal* (as MCP's
`notifications/resources/updated` does [16]); the full context is *pulled*
afterwards. This matches the CQRS split [10] and the transactional-outbox
delivery discipline [11], and it bounds fan-out cost without collapsing
latency on the urgent path. The remainder of this note derives this
recommendation from the literature and discusses the open questions it leaves
behind.

---

## 2. Background: Push, Pull, and Pub/Sub Variants

Across the distributed-systems literature the interaction models are usually
framed as follows [3,6,7]:

- **Pull / request-response.** The consumer initiates; the producer is
  passive. Consumer chooses when and what to read. Kafka consumers are the
  canonical modern example: the broker stores, the consumer polls [12]. RAG
  is pull on a different axis: the *retriever* is invoked inside the model's
  generation step, synchronously [1,2].

- **Push / notification.** The producer initiates. The consumer must be
  reachable (or have a durable mailbox). Webhooks, SSE streams, MQTT pushes
  are examples. Classical drawback: consumer backpressure is awkward; either
  messages buffer on the producer side, or the consumer must rate-limit the
  producer out-of-band.

- **Publish/subscribe.** A decoupled variant of push: producers emit into an
  anonymous channel, consumers subscribe. Eugster et al. [3] identify
  **three decouplings** it provides: (i) *space* — parties need not know each
  other, (ii) *time* — parties need not be up simultaneously, (iii)
  *synchronisation* — event delivery is asynchronous. They distinguish:

    - **Topic-based** pub/sub (named channels; subscribe by name). Simple,
      compose-able hierarchically (NATS subjects [13]; Kafka topics [12]).
    - **Content-based** pub/sub (subscribe by predicate over the event
      payload; SIENA [7], Gryphon [7], EventBridge rules [17]). More
      expressive, but routing is harder.
    - **Type-based** pub/sub (subscribe by event type in a typed object
      graph). Conceptually clean; uncommon in practice.

- **Hybrid / CQRS-style.** Reads and writes are separated; writes emit
  events, reads hit a projected view [10]. The event stream *notifies*; the
  projection is *pulled*. This naturally models "signal vs. detail," which
  is the core observation for RQ4.

- **Semantic subscription.** Predicate is not a boolean over fields but a
  similarity threshold against an embedding, or a continuous SPARQL-like
  query [24]. Not yet mainstream but increasingly discussed in
  streaming-RAG contexts [21].

These are the building blocks. The rest of the note applies them to the
Context Engine.

---

## 3. Survey

### A. Pull / Retrieve Model

**RAG as the canonical pull pattern.** Lewis et al. [1] introduced
Retrieval-Augmented Generation for knowledge-intensive NLP: parametric LLM
memory plus a non-parametric dense index, queried at generation time. Gao et
al.'s survey [2] organises the literature into Naive, Advanced, and Modular
RAG, formalising retrieval-then-generate as the dominant design. Singh et
al.'s "Agentic RAG" survey [19] extends this: the agent *decides* when and
what to retrieve — retrieval becomes a tool call, not a preprocessing step.

In production, **LangChain** retrievers, **LlamaIndex** query engines [20],
and **Letta/MemGPT**'s archival-memory tools [9] all follow the same
agent-initiated pattern: the agent emits a natural-language or structured
query, the framework routes it to a retriever, and a chunk of text comes
back into the context window. LlamaIndex's FunctionAgent/ReActAgent explicitly
model retrieval as function calling [20]; Letta similarly exposes
`archival_memory_search` and `conversation_search` as tools that the agent
invokes [9].

**Limitation of pure pull.** The agent has to *know when to poll*. If an
urgent Slack message arrives, no retrieval will surface it until the agent
decides, on its own, to ask. This is fine for user-triggered sessions ("a
user just asked X, look things up") but pathological for long-running
agents reacting to world events. This is exactly the gap the thesis's
fast-path is meant to close.

### B. Push / Pub-Sub Model

**Classical brokers.** Kafka [12] is pull-based from the consumer's point of
view but semantically pub/sub (topics + consumer groups); it leaves
backpressure to the consumer, which is a clean property for mixed-latency
agent workloads. Apache Pulsar [23] adds four subscription types —
Exclusive, Failover, Shared, Key_Shared — where **Key_Shared is the most
interesting for agents**: messages with the same key are routed to the same
consumer, so per-entity (per-customer, per-ticket) ordering is preserved
while scaling across agent replicas. NATS [13] uses dot-separated subject
hierarchies with `*` and `>` wildcards — a lightweight form of topic
predicate that also fits the Context Engine.

**Content-based pub/sub.** Eugster et al. [3] note content-based is the most
expressive but the hardest to scale. SIENA [7] and Gryphon [7] are the
canonical systems: subscribers register predicates, brokers propagate them
through an overlay, and events are routed only where they match. Modern
serverless re-implements this: **AWS EventBridge** [17] lets you subscribe
with JSON patterns over event bodies (numeric ranges, prefix, anything-but,
wildcards), which is essentially SIENA-style content routing as a managed
service. **Apache Pulsar** and **Google Cloud Pub/Sub** [24] offer filter
expressions on subscriptions.

**Backpressure.** Pull-based systems handle backpressure naturally — the
consumer just polls slower [12]. Push-based systems need an explicit
mechanism: credit schemes, consumer-signalled rate limits, or buffering with
drop policy. For agents, which have highly variable processing time
(seconds for fast tool calls, tens of seconds for reflection loops),
backpressure must be first-class.

### C. Agent-Specific Protocols

**MCP (Model Context Protocol).** Anthropic's MCP [4,16] is the most
relevant protocol for this thesis. It defines a JSON-RPC interface between
an LLM client and a "server" exposing **tools** (callable actions),
**resources** (readable data), and **prompts** (templates). Crucially for
RQ4, MCP supports a **resource subscription** primitive [16]:

- Server declares `resources.subscribe = true` during capabilities
  negotiation.
- Client calls `resources/subscribe` with a resource URI.
- Server emits `notifications/resources/updated` when that resource
  changes.
- Client then pulls the new content with `resources/read`.

This is *exactly* the hybrid pattern: a lightweight push notification
triggers a targeted pull. Transport is Streamable HTTP with Server-Sent
Events [16], so long-lived subscriptions fit cleanly. As of the 2025-11-25
spec [4], MCP also supports asynchronous operations and a registry for
server discovery — useful if the Context Engine is itself an MCP server.

**A2A (Agent-to-Agent).** Google's A2A protocol [5] is complementary to MCP:
MCP standardises tool/resource access for a single agent, while A2A
standardises agent-to-agent communication via "Agent Cards" (capability
descriptors) plus JSON-RPC tasks. A2A supports synchronous request/response,
SSE streaming, and **asynchronous push notifications** — again a hybrid. The
Linux Foundation now governs both MCP and A2A, and they are converging on
interoperability.

**Agent framework context delivery.**

- **LangGraph** [8] uses a shared graph state with message-passing between
  nodes; context is both *pushed* (state updates propagate along edges via
  `Command` objects) and *pulled* (nodes call `get` on `InMemoryStore`).
  LangGraph also has *triggers* that let graphs subscribe to external
  events.
- **AutoGen 0.4** [14] moved to an event-driven, async-first runtime;
  agents communicate by publishing messages on a bus, subscribing to
  topics/addresses.
- **CrewAI** [14] uses role-based task passing (Crews) or explicit state
  machines (Flows); context is largely pull-style (agent reads `Task`
  inputs) but delivered eagerly.
- **OpenAI Swarm / Agents SDK** [15] is handoff-centric: conversation state
  is passed from one agent to the next; no pub/sub primitive.
- **Letta (MemGPT)** [9] is pure pull — agents explicitly tool-call memory
  operations; no external push.

The pattern: frameworks that operate in *closed-world, user-triggered*
settings lean pull; frameworks that operate in *open-world, long-running*
settings (LangGraph, AutoGen) add push.

### D. Semantic / Embedding-Based Subscriptions

Topic-based and content-based filters assume the publisher and subscriber
share a schema. The richer idea is **semantic subscription**: "notify me when
something similar to this embedding is published," or "when an event
semantically matches 'urgent customer complaint.'"

- Classical work on **continuous queries over streams** [24] (CQL, CQELS)
  provides the formal foundation: a query is a standing SPARQL/SQL
  expression evaluated as the stream advances; new matches are pushed.
- **Vector DBs** (Pinecone, Weaviate, Milvus) focus on *query-time* ANN
  search, not standing subscriptions. None of the major vector DBs in 2026
  expose a native "subscribe to an embedding" primitive — real-time
  updates exist on the *write* side (insert/upsert), not on the *notify*
  side. Streaming-RAG designs [21,22] sidestep this by re-indexing
  continuously rather than pushing.
- **StreamingRAG** [21] constructs evolving scene-aware knowledge graphs
  from data streams, achieving 5-6× faster throughput than classical RAG
  while keeping context accurate; it is effectively a *continuous
  indexing* system with pull-style retrieval.

**Implication for the thesis.** Semantic subscriptions are attractive but
immature. For now, content-based predicates over typed fields are the safe
default. The Context Engine can reserve an optional embedding-based filter
type, evaluated by a lightweight classifier when an event enters the
fast path (cheap to add later).

### E. Hybrid Patterns

Several widely-deployed patterns combine push and pull; each maps cleanly
onto the thesis architecture:

- **Event-driven + RAG.** Kafka/Flink provides a real-time event backbone;
  events update a projected vector store; agents RAG-query the store on
  demand [18,22]. The architecture described in [18] is almost
  isomorphic to this thesis's Processing Engine + Context Engine split.
- **CQRS** [10]. Separate write-side (event sourcing) from read-side
  (projected views). Agents read projections; events travel through a
  separate command path. This maps onto fast path = command/event path,
  batch path = projection/query path.
- **Transactional outbox** [11]. Writes to the Context Engine and emission
  of notification events happen atomically via an outbox table drained by
  a relay. Without this, notification and state drift: an agent is
  notified of an event that, when it pulls, has not yet been written.
  Outbox is a hard requirement, not a nice-to-have.
- **Notification-then-pull** (MCP-style [16]). Push a thin signal; pull
  the fat payload. Keeps per-agent fan-out cheap (signals are small),
  avoids per-agent serialization of large context, and lets the agent
  apply its own authorization/filtering at read time.

### F. Scalability Concerns

- **Fan-out.** With M agents and N events, naive broadcast is O(N·M).
  Content filtering pushes this toward O(N·k) where k is the average
  per-event match count. SIENA-style brokers [7] and EventBridge [17]
  rely on this. Topic hierarchies (NATS [13], Kafka [12]) partition
  upstream to bound fan-out.
- **Backpressure per agent.** A slow agent must not block the bus. Kafka's
  pull model [12] and Pulsar's Shared/Key_Shared subscriptions [23] solve
  this by persisting the backlog; each agent has its own offset. The
  Context Engine should follow suit: each agent subscription is a
  durable, independently-advancing cursor.
- **Rate limiting.** Tokens, LLM calls, and downstream tool-call budgets
  all cap an agent's intake. The Engine should support per-subscription
  max-in-flight and drop/coalesce policies (drop-duplicate, drop-oldest,
  sample-latest).
- **Dynamic membership.** Agents should join/leave without broker
  reconfiguration. MCP's registry model [4] plus A2A Agent Cards [5] give
  a concrete pattern: new agent boots, advertises a subscription
  descriptor, Engine registers it.

---

## 4. Evaluation Dimensions

| Dimension | Pull only (RAG) | Push/Pub-Sub only | Hybrid (notify + pull) |
|---|---|---|---|
| **Latency (urgent event → agent)** | Bad: bounded by poll period | Good: O(ms) plus processing | Good: notification is O(ms); pull adds ~one RTT |
| **Latency (background context)** | Good | Over-delivers, wastes tokens | Good: agent pulls on its own clock |
| **Extensibility (adding agents)** | Excellent: no coordination | OK with content filters; fan-out grows | Excellent: new agent = new subscription + read scope |
| **Subscription expressiveness** | N/A (query per call) | Topic < content < semantic | Can layer all three |
| **Implementation complexity** | Low | Medium (broker + filters) | Medium–high (both paths, outbox) |
| **Backpressure handling** | Trivial (consumer-driven) | Hard (requires credit/drop) | Clean: push is a thin signal; pull is consumer-paced |
| **Consistency (notify vs. state)** | Trivially consistent | Can drift | Needs transactional outbox [11] |
| **Durability / replay** | Depends on store | Depends on log (Kafka-good [12]) | Both paths durable if built on a log |
| **Observability / lineage** | Easy (every query is a call) | Requires topic-level tracing | Medium: notification ID ↔ pulled state ID pairing |

---

## 5. Applicability to the Thesis

The thesis's working hypothesis is: **push urgent events via the fast path;
pull aggregated context via retrieval on the batch path.** The survey
largely validates this, with refinements:

**Validation.** The hybrid pattern is not a compromise between two worse
options; it is an independent, well-studied design. Every production-grade
system encountered — MCP's resource subscriptions [16], A2A's push
notifications [5], Confluent's event-driven multi-agent patterns [18],
real-time RAG architectures [22], CQRS [10], outbox [11] — converges on
"notify cheaply, fetch richly." The thesis's Lambda split maps onto this
directly: fast path = event notification channel, batch path = projected
shared memory that agents pull/RAG over.

**Refinement 1: The fast path should push *signals*, not full events.**
Pushing raw, potentially large events to every subscribed agent duplicates
work and bloats per-agent bandwidth. Following MCP [16], the fast-path
notification should be a small envelope: `(event_id, source, type,
urgency, scope, …)`. Agents receive this, decide whether to act, and pull
the full event (and any needed aggregation) from the Context Engine. This
keeps the urgent path genuinely fast.

**Refinement 2: Subscriptions should be content-based first, semantic
later.** Topic hierarchies [13] plus content predicates [17] cover almost
all near-term needs (source = `jira`, priority ≥ P1; channel =
`#oncall`; entity scope contains agent's tenant). Embedding-based
subscriptions are an attractive research direction but should not be on
the critical path for RQ4 validation.

**Refinement 3: Agent pulls should speak a standard protocol.** Inventing a
bespoke retrieval API is tempting but narrows the agent ecosystem. MCP [4]
is the obvious target: expose the Context Engine as an MCP server, with
shared memory views as `resources`, retrieval functions as `tools`, and
fast-path subscriptions as subscribed resources emitting
`notifications/resources/updated` [16]. A2A [5] is then additive if the
thesis extends to multi-agent coordination.

**Refinement 4: Transactional outbox is a correctness requirement.**
Without [11], an agent will receive a "look at event 42" notification and
then pull and find no event 42 (because the write has not yet propagated
to the read side), or vice versa. This is not optional.

**Refutation (partial) of the naive split.** One thing the survey *does*
refute: the idea that batch-path context is *only* ever pulled. Some
batch-path outputs — e.g., "the rolling summary of this customer's week"
— may be more useful if pushed to agents subscribed to that customer
when the summary changes materially. The architecture should therefore
support push notifications on derived/aggregated resources, not just on
raw fast-path events.

---

## 6. Proposed Recommendation: Protocol Sketch

### 6.1 Surface

The Context Engine exposes two faces:

1. **Control + read plane.** An MCP server [4,16] with:
    - `resources/` — aggregated shared-memory views (per entity, per
      topic, rolling summaries). Each resource is URI-addressed and
      versioned.
    - `tools/` — retrieval functions (`search_context`,
      `get_entity_timeline`, `semantic_search`). These are the pull API.
    - `prompts/` — optional reusable templates.
2. **Notification plane.** Over MCP's streamable-HTTP/SSE transport [16]:
    - `resources/subscribe` — agent subscribes to a resource URI *or* a
      content predicate over the fast-path event envelope (predicate
      language: JSON pattern, à la EventBridge [17], optionally a
      semantic-similarity clause).
    - `notifications/resources/updated` — server pushes the *change
      signal*: `{resource_uri, version, trigger_event_id, reason}`.
    - `notifications/resources/list_changed` — the subscription surface
      itself changed.

### 6.2 Subscription expression

A subscription descriptor, conceptually:

```
{
  "id": "agent-X-urgent-slack",
  "scope": {
    "topic": "events.slack.*",             // NATS-style hierarchy
    "filter": {                             // EventBridge-style predicate
      "urgency": ["P0", "P1"],
      "mentions": [{"prefix": "@oncall"}]
    },
    "semantic": {                           // optional, v2
      "embedding": "...",
      "threshold": 0.82
    }
  },
  "delivery": {
    "mode": "notify",                       // notify | notify_with_payload
    "max_in_flight": 8,
    "on_overflow": "drop_oldest"
  }
}
```

This subsumes (i) topic-based pub/sub [12,13], (ii) content-based pub/sub
[3,7,17], and (iii) future semantic subscriptions [21,24], without forcing
all three on agents.

### 6.3 Delivery semantics

- **At-least-once** for fast-path notifications. Agents must idempotently
  handle duplicates (carry `event_id`, track seen set). This matches
  Kafka's [12] and Pulsar's [23] usual guarantees.
- **Durable cursor per subscription** (like Kafka consumer group or
  Pulsar subscription) so that an agent can go offline and catch up.
- **Notification-then-pull**: notifications carry IDs, not full payloads,
  by default. Agents pull with `resources/read` or a `tools/` call. A
  `notify_with_payload` mode exists for small payloads to save an RTT.
- **Transactional outbox** [11] between the write side (Processing
  Engine) and the notification side. Writes to the shared memory and
  enqueuing of the notification are atomic; a relay drains the outbox to
  the notification bus.
- **Backpressure**: each subscription has its own cursor and
  `max_in_flight`; a stuck agent slows only its own stream.

### 6.4 Onboarding a new agent

1. Agent connects to the Context Engine MCP endpoint.
2. Server advertises capabilities (`resources.subscribe = true`, tool
   list, resource templates) [4,16].
3. Agent posts one or more subscription descriptors.
4. Optionally, agent registers an A2A Agent Card [5] so other agents can
   route tasks to it.
5. The Processing Engine does not need to know the agent exists — the
   Context Engine alone owns subscription state. This satisfies the
   agent-extensibility non-functional requirement.

---

## 7. Open Questions

- **Semantic subscription evaluation.** How costly is it to evaluate an
  embedding-similarity predicate per event on the fast path? Is a
  two-stage filter (cheap content predicate → expensive semantic check)
  acceptable? What false-positive/negative rates are tolerable?
- **Push vs. pull on the batch path.** Should rolling summaries push when
  they materially change (and how is "materially" measured — token-level
  diff, embedding distance, downstream-relevance score)? Or always pull?
- **Duplicate-suppression across paths.** An event arrives on the fast
  path; it also ends up in the batch-path aggregation an hour later. Do
  agents see it twice? How is the fast-path notification correlated with
  the batch-path update (shared `event_id` lineage, per §4 observability)?
- **Subscription revocation semantics.** When an agent unsubscribes, are
  in-flight notifications delivered? Is the cursor retained for a grace
  period?
- **Cross-agent ordering.** Two agents subscribed to overlapping scopes
  — do they see the same ordering? Per-key ordering à la Key_Shared [23]
  seems sufficient; global ordering is likely over-spec.
- **MCP scope fit.** MCP was designed for tool-using chat assistants, not
  for long-running server-subscribed agents. Does its subscription
  surface actually scale to thousands of long-lived subscriptions, or is
  a broker (Kafka/Pulsar/NATS) needed behind an MCP façade?
- **Selection between MCP-as-transport vs. Kafka-as-transport.** MCP gives
  agent-native ergonomics; Kafka gives operational maturity. A likely
  answer is Kafka (or NATS/Pulsar) as the internal bus, with MCP as the
  external client protocol — but that wiring needs concrete prototyping.

---

## 8. References

[1] Lewis, P. et al. (2020). *Retrieval-Augmented Generation for
Knowledge-Intensive NLP Tasks.* NeurIPS 2020.
<https://arxiv.org/abs/2005.11401>

[2] Gao, Y. et al. (2023). *Retrieval-Augmented Generation for Large
Language Models: A Survey.* arXiv:2312.10997.
<https://arxiv.org/abs/2312.10997>

[3] Eugster, P. T., Felber, P. A., Guerraoui, R., Kermarrec, A.-M. (2003).
*The Many Faces of Publish/Subscribe.* ACM Computing Surveys, 35(2),
114–131. <https://dl.acm.org/doi/10.1145/857076.857078>
(PDF: <http://www.cs.ru.nl/~marko/onderwijs/oss/eugster-al_publish-subscribe.pdf>)

[4] Anthropic / Model Context Protocol project. *MCP Specification
(2025-11-25).* <https://modelcontextprotocol.io/specification/2025-11-25>

[5] Google et al. *Agent2Agent (A2A) Protocol Specification.*
<https://a2a-protocol.org/latest/specification/>;
announcement:
<https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/>

[6] Shen, H. *Content-based Publish/Subscribe Systems* (book chapter).
<https://www.cs.virginia.edu/~hs6ms/publishedPaper/bookChapter/2009/sub-pub-Shen.pdf>

[7] Carzaniga, A., Rosenblum, D. S., Wolf, A. L. (and successors). *SIENA
and content-based routing.* Columbia DS course reader and
<https://www.cl.cam.ac.uk/~ey204/teaching/ACS/R212_2015_2016/papers/carzaniga_infocom_2004.pdf>;
IBM Gryphon referenced in [3].

[8] LangChain. *LangGraph Graph API overview.*
<https://docs.langchain.com/oss/python/langgraph/graph-api>

[9] Letta. *Concepts: MemGPT and agent memory architecture.*
<https://docs.letta.com/concepts/memgpt/>;
<https://www.letta.com/blog/agent-memory>

[10] Fowler, M. *CQRS.* <https://martinfowler.com/bliki/CQRS.html>;
Microsoft Learn:
<https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs>

[11] Richardson, C. *Pattern: Transactional Outbox.*
<https://microservices.io/patterns/data/transactional-outbox.html>;
Confluent course:
<https://developer.confluent.io/courses/microservices/the-transactional-outbox-pattern/>

[12] Apache Kafka. *Consumer Groups and Consumer API.*
<https://developer.confluent.io/learn-more/kafka-on-the-go/consumer-groups/>;
backpressure discussion:
<https://streamkap.com/resources-and-guides/backpressure-stream-processing>

[13] NATS. *Subject-Based Messaging (hierarchies, `*`/`>` wildcards).*
<https://docs.nats.io/nats-concepts/subjects>

[14] Microsoft. *AutoGen Multi-agent Conversation Framework* (and
comparison with CrewAI).
<https://microsoft.github.io/autogen/0.2/docs/Use-Cases/agent_chat/>;
<https://www.zenml.io/blog/crewai-vs-autogen>;
<https://deepwiki.com/ombharatiya/ai-system-design-guide/7.3-autogen-and-crewai-architectures>

[15] OpenAI. *Swarm: lightweight multi-agent orchestration.*
<https://github.com/openai/swarm>;
handoff cookbook:
<https://developers.openai.com/cookbook/examples/orchestrating_agents>

[16] Model Context Protocol. *Transports (Streamable HTTP + SSE) and
Resource subscriptions/notifications.*
<https://modelcontextprotocol.io/specification/2025-06-18/basic/transports>;
<https://apxml.com/courses/getting-started-model-context-protocol/chapter-2-defining-resources-and-prompts/resource-subscriptions-notifications>

[17] Amazon Web Services. *EventBridge event pattern syntax and
content-based filtering.*
<https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-pattern.html>;
<https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns-content-based-filtering.html>

[18] Confluent. *A Guide to Event-Driven Design for Agents and Multi-Agent
Systems;* *Four Design Patterns for Event-Driven, Multi-Agent Systems.*
<https://www.confluent.io/resources/ebook/guide-to-event-driven-agents/>;
<https://www.confluent.io/blog/event-driven-multi-agent-systems/>

[19] Singh, A., Ehtesham, A., Kumar, S., Khoei, T. T., Vasilakos, A. V.
(2025). *Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG.*
arXiv:2501.09136. <https://arxiv.org/abs/2501.09136>

[20] LlamaIndex. *Agents, Query Engines, and Tools.*
<https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/>;
<https://www.llamaindex.ai/blog/agentic-rag-with-llamaindex-2721b8a49ff6>

[21] Sankaradas, M., Rajendran, R. K., Chakradhar, S. T. (2025).
*StreamingRAG: Real-time Contextual Retrieval and Generation Framework.*
arXiv:2501.14101. <https://arxiv.org/abs/2501.14101>

[22] Striim. *Real-Time RAG: Streaming Vector Embeddings and Low-Latency
AI Search.*
<https://www.striim.com/blog/real-time-rag-streaming-vector-embeddings-and-low-latency-ai-search/>;
AWS Architecture Diagrams. *Real-time Streaming for RAG.*
<https://docs.aws.amazon.com/architecture-diagrams/latest/exploring-real-time-streaming-for-retrieval-augmented-generation/exploring-real-time-streaming-for-retrieval-augmented-generation.html>

[23] Apache Pulsar. *Subscription Types (Exclusive, Failover, Shared,
Key_Shared).* <https://pulsar.apache.org/docs/next/concepts-messaging/>;
<https://streamnative.io/blog/pulsar-newbie-guide-for-kafka-engineers-part-4-subscriptions-consumers>

[24] Arasu, A., Babu, S., Widom, J. *The CQL Continuous Query Language:
Semantic Foundations and Query Execution.* Stanford InfoLab.
<http://ilpubs.stanford.edu:8090/758/1/2003-67.pdf>;
Google Cloud Pub/Sub documentation:
<https://cloud.google.com/pubsub/docs/overview>
