# Thesis Proposal — Open-Source Context Engine for Streaming AI Agents

## Prior art: Confluent Real-Time Context Engine (closed source)

A commercial system with this exact architectural shape already exists: Confluent's **Real-Time Context Engine (RTCE)**, currently Early Access on Confluent Cloud.

![Confluent RTCE architecture](img_1.png)

*Website/mobile/CDC events stream into Kafka, Flink jobs materialise per-entity context views (CUSTOMER360, ORDERS), and an LLM agent retrieves them at inference time.* RTCE is **proprietary**, Early Access, and its agent interface is **pull-only** over MCP — the agent must ask; it is never notified. Tableflow (the Iceberg/Delta materialisation layer) is likewise proprietary.

## Problem statement

> **AI agents operating over high-volume heterogeneous event streams (Slack, JIRA, e-commerce telemetry, …) need a context management system that delivers the right context at the right time, with business-critical events pushed immediately — and no open-source reference architecture or protocol exists for this.**

## Architectural commitments

Two opinionated stances frame the rest of the proposal; reviewers should read the diagram with both in mind.

**Stateless agents — the CMS owns every token that reaches an LLM.** Agents are pure functions: their *entire* input is a single `(event, context)` envelope produced by the CMS, and their output is an action. There is no agent-side memory, no in-process cache, and no runtime callback into the engine to fetch additional context. All long-lived state (entity timelines, relations, recently-believed facts) lives inside the Context Engine, in [Graphiti](https://github.com/getzep/graphiti). This collapses two otherwise-tangled responsibilities — *deciding what context an agent needs* and *running an agent* — onto opposite sides of a clean contract.

**Generalizable delivery via Kafka.** Enriched `(event, context)` envelopes are produced to Kafka topics partitioned by entity key. Developers consume them with whatever runtime fits their workload — a long-lived async consumer, a Lambda-style ephemeral spawn, k8s, Modal, anything that speaks Kafka. The CMS does not dictate the agent runtime, language, or framework. MCP is used *internally to the engine* for the Graphiti memory layer, but the agent-facing edge is Kafka, not MCP. The trade-offs against MCP-streaming push and MCP-pull surfaces are surveyed in RQ4 below.

## Proposed architecture

![Proposed Context Management System architecture](architecture.svg)

*Heterogeneous events enter the Context Management System and are first routed by an Event Classification stage onto a fast lane (urgent, signal-grade) or a batch lane (aggregated Context Units) — a latency split, not a Lambda-style dual codebase. The Context Engine indexes events into Graphiti, materialises per-entity views, computes the `context` object per outgoing event, and produces `(event, context)` envelopes onto partitioned Kafka topics. Agents are stateless consumers of those topics — events are routed to the right agent by partition key.*

**Processing note.** Both lanes flow through the same indexing path inside the Context Engine; only the *moment of publish* differs (fast = on arrival, batch = at window close). The per-event pipeline — Graphiti episode write (in-memory, FalkorDB-backed), context computation, envelope emission — is described in *Context Engine — internal architecture* below.

## Context Engine — internal architecture

The Context Engine is the box inside the dashed CMS boundary that turns a raw classified event into the `(event, context)` envelope that ends up on Kafka. Four concerns:

**Memory layer — Graphiti, in-memory.** The engine uses [Graphiti](https://github.com/getzep/graphiti) — the open-source implementation of the Zep architecture (Rasmussen et al., *Zep: A Temporal Knowledge Graph Architecture for Agent Memory*, arXiv:2501.13956) — as its bitemporal temporal-knowledge-graph backing store. Each ingested event is written as a Graphiti *episode*; Graphiti automatically extracts entities and relations, attaches `valid_from` / `invalidated_at` edges, and supersedes contradicted facts. This gives the engine native bitemporal recall ("what did we believe about customer X at time t?") without bespoke versioning code. Graphiti is run on **FalkorDB** (a Redis-backed in-memory graph database that Graphiti supports as a backend) so all hot-path context queries hit RAM; durability comes from Redis's standard persistence (RDB snapshots + AOF). Cold-tier event storage (audit log, archived raw events past the working set) lives separately in Paimon / Iceberg, but the Graphiti graph itself stays exclusively in FalkorDB — there is no second graph backend in v1. *No hand-rolled "materialised view" cache layer is maintained on top* — Graphiti is the single source of truth for memory state, and FalkorDB is the in-memory execution engine. The detailed rationale and trade-off survey for Graphiti as the memory primitive is in `research/RQ5-shared-memory.md`.

*Scope note on memory scaling.* FalkorDB is single-node (Redis-style) and therefore scales vertically only. The thesis accepts this ceiling for v1: the reference implementation runs a single FalkorDB instance, and benchmarks operate within its working-set capacity. Horizontal scaling — sharding the Graphiti graph across a Redis Cluster keyed by entity-id, or a comparable scheme — is explicitly **future work**. It is non-trivial because graph traversals don't shard cleanly across Redis hash slots, but the rest of the architecture is shard-friendly (events are already partitioned by entity key in Kafka), so the eventual sharding boundary aligns naturally with the existing partition key. Surveying that sharding design — and whether FalkorDB's roadmap, an alternative Graphiti backend, or a custom routing layer is the right primitive — is left for a follow-on study.

**Context computation at publish time.** For every outgoing event, the engine queries Graphiti for the partition key's *current entity state*, the *last N events* on that key, and *relevant facts* pinned to the event time. The result is the `context` half of the envelope. Because Graphiti runs in-memory, these queries are sub-millisecond on the hot working set. This is the work the engine performs that lets agents stay stateless — the LLM never has to call back to a memory store mid-turn.

**Authorisation + filtering.** The engine is the single authorisation boundary. Tenant and session scoping are applied to the `context` object *before* publish, not on the agent side. An agent process that subscribes to a Kafka topic for `tenant:acme` cannot see events whose authorisation predicates exclude that tenant — the engine simply will not have produced them onto a topic the agent has access to.

### The `(event, context)` envelope

The contract between the CMS and any agent is a single JSON envelope. Bumping it is a versioned migration; agents pin a version.

```jsonc
{
  "schema":  "cms/event-context/v1",
  "event":   {
    // CloudEvents v1.0 envelope
    "source":  "...",
    "id":      "...",
    "type":    "...",
    "subject": "customer:42",
    "time":    "...",
    "data":    { /* event payload */ }
  },
  "context": {
    "entity_state":   { /* current per-key snapshot from Graphiti */ },
    "recent_events":  [ /* last N events for the same partition key */ ],
    "relevant_facts": [ /* graph-queries pinned to event.time */ ]
  }
}
```

The `event.subject` field doubles as the Kafka partition key, guaranteeing per-entity ordering across consumers in the same group.

## Research questions

- **RQ1 — Streaming architecture.** Lambda vs Kappa vs Kappa+/Streamhouse for a context engine feeding agents? *[see `research/RQ1-lambda-vs-kappa.md`]*
- **RQ2 — Event classification.** How to decide, per event, whether it belongs on the fast path, the batch path, or both? *[see `research/RQ2-event-classification.md`]* — detailed pipeline below.
- **RQ3 — Aggregation.** Events for the same entity (e.g. customer id) are grouped together and released to the next stage on a windowed timeframe. *[see `research/RQ3-batch-aggregation.md`]*
- **RQ4 — Delivery surface.** What protocol carries the `(event, context)` envelope from the engine to a stateless agent? Decision: Kafka direct, partitioned by entity key — surveyed against MCP Streaming push, MCP pull, and the broader landscape. *[see `research/RQ4-context-distribution.md` + `research/push-protocol/report.md`]*

## RQ2 — Proposed classification pipeline

![Event classification pipeline](rq2-classification.svg)

**How it works:**

- **Stage 1 — Envelope.** Every ingested event is wrapped in a CloudEvents v1.0 envelope by a per-source adapter. Extracts structured features (source, type, time, priority hints, subject, tenant). Essentially free (<1 ms).
- **Stage 2 — Declarative rules.** A small, auditable rule set (JsonLogic / Drools) short-circuits the obvious patterns — JIRA P0, Slack @oncall, Stripe dispute ≥ $N, PagerDuty incident — straight to the routing action. Keeps the bandit's exploration budget focused on the genuinely ambiguous events.
- **Stage 3 — Calibrated classifier.** LightGBM / XGBoost on envelope features (plus an optional small text embedding). Emits `p(urgent)` and a calibrated uncertainty estimate. CPU-only, p95 < 5 ms. Trained offline.
- **Stage 5 — Contextual bandit.** LinUCB or Thompson Sampling (Vowpal Wabbit `--cb_explore_adf`). Context = envelope features + classifier score + classifier uncertainty + operational state (queue depth, time-of-day, tenant). Actions = `{fast, batch}` — binary. The agents themselves close the feedback loop. Picks the action with the highest uncertainty-adjusted expected reward.
- **Agent feedback — fully automated reward loop.** Every agent writes a `routing_feedback` record to the `cms.feedback.v1` Kafka topic once per event it consumes. The record contains `{event_id, verdict}`, where `verdict ∈ {appropriate, too_urgent, too_slow, irrelevant}` — a tiny, well-defined vocabulary. No humans involved.
- **Two learning loops on different clocks, both fed by that tool:**
  - *Online* — verdict maps to scalar **reward** (`+1` appropriate, `−0.5` too_urgent, `−0.5` too_slow, `−1` irrelevant). Bandit updates per event.
  - *Offline* — accumulated `(event, verdict)` pairs become **labels** for weak-supervision retraining of the classifier, nightly / weekly.

**Runtime behaviour:**

| Event | Classifier | Ops state | Action | Agent verdict (tool call) | Reward | What the bandit learns |
|---|---|---|---|---|---|---|
| Stripe dispute $50K | `p=0.95` | fast queue healthy | **fast** | `appropriate` | `+1` | reinforce `fast` for high-value disputes |
| `#random` chit-chat | `p=0.08` | — | **batch** | `appropriate` | `+0.1` | reinforce `batch` for low-score events |
| JIRA P2 status update | `p=0.42` | healthy | **batch** (conservative default) | `appropriate` | `+0.2` | reinforce `batch` for mildly ambiguous, low-stakes events |
| Verbose log alert | `p=0.61` | healthy | **fast** (explore) | `too_urgent` — agent burned tokens, event wasn't actionable | `−0.5` | learn: verbose log alerts don't justify fast-path cost |
| Slack @oncall | — | — | **fast** *(rule short-circuit)* | — | — | bandit never sees it — exploration budget preserved for hard cases |

## RQ4 — Delivery surface for stateless agents

Given the architectural commitments above (CMS owns context; agents are stateless), RQ4 reduces to: *which protocol carries the `(event, context)` envelope from engine to agent?* Three plausible families exist — a streaming push protocol over the LLM-tooling stack (MCP Streaming Resources), a pull/polling protocol over the same stack (Confluent RTCE's actual interface today), or a brokered event bus that stays out of the LLM-tooling stack entirely (Kafka). The proposal commits to the third.

### Decision: Kafka direct

![(event, context) delivery over Kafka](rq4-kafka-delivery.svg)

The Context Engine produces `(event, context)` envelopes onto Kafka topics partitioned by entity key. Concrete conventions:

- **Topics — split by lane.** `cms.events.fast.v1` (24 h retention, urgent events) and `cms.events.batch.v1` (7 d retention, windowed Context Units). Agent → engine feedback flows on `cms.feedback.v1`, where agents write `routing_feedback` verdicts that train the RQ2 bandit.
- **Partition key.** The CloudEvent `subject` field (`customer:42`, `order:9931`, `tenant:acme`). Kafka hashes the key to a single partition, so per-entity ordering is preserved across the consumer group. Cross-entity ordering is *not* preserved — agents only ever care about per-entity timelines.
- **Partition count.** 50 for fast, 30 for batch. Sized for peak parallelism; over-provisioned because partition count is hard to change live.
- **Consumer groups — one per agent role.** `cms.agents.triage`, `cms.agents.compliance`, `cms.agents.analytics`. Multiple roles on the same topic give independent fan-out with separate offset cursors. Within a role, replicas share partitions via the cooperative-sticky assignor (Kafka ≥ 2.4) to avoid stop-the-world rebalances.
- **Idempotency.** The CloudEvent `(source, id)` pair is the dedupe key. The CMS guarantees at-least-once; agents dedupe.
- **DLQ.** `<group>.dlq` per role for envelopes that fail N retries.

Developers consume from these topics with whatever runtime fits the workload — a long-lived async consumer process, a Lambda-style ephemeral spawn (one invocation per event), Knative, Modal, anything that speaks Kafka. The CMS does not dictate.

### Alternative considered — MCP Streaming Resources push

An earlier draft of this proposal centred a content-bearing extension of MCP's `resources/subscribe` + `notifications/resources/updated` over Streamable HTTP, with `flow/*` back-pressure and `Last-Event-ID` resumability. The protocol is a clean fit for LLM-tooling-native agents and could be submitted as an MCP SEP. It is **rejected as the primary surface** for three reasons:

- **Scalability.** Long-lived SSE / Streamable-HTTP connections impose per-connection memory and event-loop cost that scales worse than Kafka consumer-group fan-out for the subscriber counts a CMS will see. `research/push-protocol/report.md:318` flagged this as an explicit open problem; `research/push-protocol/04-push-delivery-semantics.md` enumerates the production patterns — Kafka groups, MQTT 5 `$share/`, AMQP shared subscriptions — that exist precisely to side-step single-connection-per-subscriber.
- **Generalizability.** Forcing every adopter onto an MCP client narrows the runtime universe; Kafka is everywhere.
- **Failure-mode complexity.** Replay via `Last-Event-ID` is a bespoke server-state contract; Kafka offsets are a well-understood broker-state contract.

Future work *can* add an MCP-push façade *on top of* the Kafka spine — a server that consumes from `cms.events.*` and re-emits as `notifications/resources/updated` — as a convenience surface for agents already in the MCP ecosystem. It would not be on the critical path.

### Alternative considered — MCP pull / polling (RTCE baseline)

The agent calls `resources/read` against a CMS-side MCP server on a schedule. This is the public state-of-the-art today — Confluent RTCE's only agent interface (`research/confluent-rtce-deep-dive.md`). Rejected because polling either wastes calls during quiescence or lags the fast-path SLA on bursty traffic. The whole point of RQ4 is to close the push gap RTCE leaves open; a polling baseline would re-open it.

### Other alternatives surveyed and discarded

Briefly, with citations into the research notes:

- **NATS JetStream durable consumers** — viable, but doesn't beat Kafka on the dimensions that matter here (`research/push-protocol/04-push-delivery-semantics.md:144–154`). One broker is enough.
- **MQTT 5 shared subscriptions** — competing-consumer pattern at scale, but the IoT-protocol baggage (QoS state machines, session expiry) buys nothing for this workload (`04-push-delivery-semantics.md:124–134`).
- **AMQP 1.0** — most complete reliability + native priority, but operational complexity and tooling mass are inferior to Kafka in the data-engineering ecosystem (`04-push-delivery-semantics.md:135–142`).
- **Raw WebSocket / gRPC server-streaming** — isomorphic to MCP Streaming at the transport layer; same per-connection cost story (`04-push-delivery-semantics.md:80–87, 102–109`).
- **A2A push, OpenAI Realtime, AGNTCY SLIM, Temporal Signals** — wrong direction or wrong shape for upstream→agent delivery (`research/push-protocol/02-a2a-and-agent-push-landscape.md`).
- **Webhook → queue → worker** — degenerate ephemeral-Kafka with strictly worse semantics (no replay, retry-only) (`02-a2a-and-agent-push-landscape.md:232–264`).

**References:**

- Apache Kafka — partitioning, consumer groups, cooperative-sticky assignor: <https://kafka.apache.org/documentation/>
- CloudEvents v1.0 specification: <https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md>
- Push-protocol literature survey: `research/push-protocol/report.md`
- Confluent RTCE deep-dive: `research/confluent-rtce-deep-dive.md`

## Agent Development Kit

The thesis ships a thin Python SDK alongside the protocol contract. Its purpose is to lower the friction of writing a stateless agent against the `(event, context)` Kafka topic — but it does *not* run business logic, hold state, or manage an LLM. Adopters who prefer a different language drop down to a raw Kafka client; the contract is the contract.

### Subscribing and handling an event

```python
import asyncio
from cms_sdk import CMSClient

async def main():
    cms = CMSClient(
        brokers="kafka:9092",
        group="cms.agents.triage",          # consumer-group name = agent role
    )

    async for ec in cms.subscribe("cms.events.fast.v1"):
        # ec.event   -> typed CloudEvent
        # ec.context -> { entity_state, recent_events, relevant_facts }
        result = await my_llm_call(ec.event, ec.context)
        await ec.feedback("appropriate")    # closes the RQ2 bandit loop

asyncio.run(main())
```

What the SDK provides: typed CloudEvent decoding, `context`-object decoding, automatic offset commits on successful feedback, dead-letter handling for envelopes that throw, and a one-liner for `routing_feedback`. Verdict vocabulary: `appropriate`, `too_urgent`, `too_slow`, `irrelevant`.

### Plugging in Claude Code as the agent

The thesis evaluation on ProAgentBench uses **the same SDK loop** with Claude Code as the agent runtime. The shape is identical — only the body of the loop changes:

```python
import asyncio
from cms_sdk import CMSClient
from claude_agent_sdk import query, ClaudeAgentOptions

options = ClaudeAgentOptions(
    system_prompt=(
        "You are a customer-triage agent. Decide what action to take given "
        "the event and its precomputed context. Use the tools available."
    ),
)

async def main():
    cms = CMSClient(brokers="kafka:9092", group="cms.agents.triage")

    async for ec in cms.subscribe("cms.events.fast.v1"):
        prompt = (
            f"Event:\n{ec.event}\n\n"
            f"Context:\n{ec.context}\n\n"
            "Decide and act."
        )
        async for msg in query(prompt=prompt, options=options):
            # stream Claude's responses — log, post to Slack, write back to Graphiti, etc.
            pass

        await ec.feedback("appropriate")

asyncio.run(main())
```

The agent stays stateless: every loop iteration is a fresh `query()` with the complete `(event, context)` payload as the only input. There is no agent-side memory, no cross-event continuation. Memory lives in Graphiti, and Graphiti is queried *by the Context Engine on the producer side*, not by the agent.

### What the SDK is not

- **Not an agent framework.** No router, no tool registry, no memory abstraction. Tools come from whatever runtime the agent uses (Claude Agent SDK, LangGraph, raw Anthropic SDK, …); memory comes from the CMS.
- **Not a lifecycle manager.** The user runs the process — k8s Deployment, Lambda over a Kafka source mapping, Modal, Knative, anything. The SDK is a library, not a platform.
- **Not Python-only at the contract level.** Any language with a Kafka client (Java, Go, Rust, TS) can implement the same `(event, context)` contract. The Python SDK is a convenience for the most common stack, not a requirement.

## Evaluation

![ProAgentBench dataset overview](evaluation-dataset.svg)

The Context Engine is evaluated on **ProAgentBench** — Tang et al., *ProAgentBench: Evaluating LLM Agents for Proactive Assistance with Real-World Data*, arXiv:2602.04482 (Feb 2026) [<https://arxiv.org/abs/2602.04482>]. It is the closest public benchmark to the thesis's target workload.

**What the dataset is:**

- **Scale.** 28,000+ events collected from 500+ hours of *real* user sessions (not LLM-synthesised). Privacy-compliant. Released by the authors under an open licence.
- **Burstiness `B = 0.787`.** The event arrival process is strongly bursty — clumps of activity separated by quiet periods — as opposed to synthetic Poisson streams where `B ≈ 0`. The paper's core finding is that synthetic streams fail to capture authentic human decision patterns, so this property must be preserved.
- **Hierarchical task framework:**
  - *Task 1 — Timing prediction.* Given the event stream up to time *t*, decide whether the agent should intervene now. Isomorphic to our RQ2 `{fast, batch}` routing decision.
  - *Task 2 — Assist content generation.* Given an intervention window, produce the assistance text. Exercises our Context Unit shape (RQ3) and Kafka-direct delivery of `(event, context)` envelopes (RQ4).
- **Metrics.** Appropriateness and timeliness of proactive suggestions. Maps cleanly onto our `routing_feedback` verdict vocabulary (`appropriate` / `too_urgent` / `too_slow` / `irrelevant`).
- **Baselines.** LLM- and VLM-based agents evaluated in the paper, with the finding that long-term memory + historical context lift prediction accuracy — which is the exact argument for maintaining the Context Engine's materialised views.

**How we use it:**

1. **Offline classifier evaluation.** Timing-prediction labels are ground truth for measuring the LightGBM/XGBoost urgency classifier (RQ2, Stage 3) before the bandit is engaged.
2. **Online bandit evaluation.** Replay the event stream through the full pipeline; measure per-event reward, regret vs. oracle routing, and drift stability (prequential accuracy, ADWIN alarms).
3. **Head-to-head delivery benchmark.** Same workload replayed into three delivery backends — RTCE (pull-only over MCP), Confluent Streaming Agents (in-pipeline push, proprietary), our Context Engine (Kafka-direct `(event, context)` envelopes) — measuring latency-to-agent, missed events, and token waste attributable to `too_urgent` misrouting.
4. **Synthetic ablation.** Flatten the burst distribution to `B ≈ 0` and re-run; quantify how much of the fast-queue + back-pressure design is justified only under real burstiness.

## Deliverables

1. **Thesis report** (this document's parent) synthesising RQ1–RQ5.
2. **`(event, context)` Kafka-delivery contract** — JSON Schema for the envelope plus per-topic conventions (naming, partitioning, consumer-group rules, idempotency, DLQ). The architectural specification an open-source CMS implementation must conform to. Replaces the earlier MCP Streaming Resources profile draft.
3. **Reference Context Engine implementation** — open-source, backed by a free-tier streaming stack (Redpanda + Flink + Paimon) and Graphiti for the memory layer, producing `(event, context)` envelopes onto Kafka topics.
4. **Python SDK for agent authors** — thin Kafka consumer + envelope decoder + `routing_feedback` helper. Lets developers write a stateless agent against the contract in a few lines; raw Kafka clients in any language remain a first-class option.
5. **Empirical evaluation on ProAgentBench** — Tang et al., *ProAgentBench: Evaluating LLM Agents for Proactive Assistance with Real-World Data*, arXiv:2602.04482 (Feb 2026). <https://arxiv.org/abs/2602.04482>
   - 28,000+ events from 500+ hours of real user sessions with preserved bursty interaction patterns (burstiness `B = 0.787`), not LLM-synthesised.
   - Hierarchical tasks: (i) **timing prediction** — when to intervene — maps directly onto our RQ2 fast-vs-batch routing decision; (ii) **assist content generation** — what to deliver — exercises our Context Unit shape (RQ3) and Kafka-direct delivery of `(event, context)` envelopes (RQ4).
   - Metrics already match the loop we need: timing appropriateness + assist-content quality ≈ our `routing_feedback` verdict vocabulary (`appropriate` / `too_urgent` / `too_slow` / `irrelevant`).
   - Real-event-stream data (vs. synthetic) also directly validates the paper's finding that long-term memory + historical context lift prediction accuracy — which is exactly what the Context Engine's materialised views are for.
