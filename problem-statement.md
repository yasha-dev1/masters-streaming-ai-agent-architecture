# Context Management System for AI Agents — Problem Statement

## 1. Motivation

Modern organizations generate events continuously across heterogeneous systems: Slack
messages, JIRA tickets, CRM updates, granular customer-interaction events on an
e-commerce site, telemetry, and so on. At organizational scale this can reach
thousands of events per minute, across dozens of source systems, each with its own
schema, delivery semantics, and latency characteristics.

AI agents (LLM-based or otherwise) that operate on behalf of the organization need
access to this stream of events as **context** in order to act usefully. However,
agents are not event consumers in the traditional sense:

- Different agents care about different subsets of events.
- Some events require **immediate** agent attention (e.g., a critical customer
  complaint arriving in Slack); most events only contribute to a **slowly-evolving
  shared understanding** of the business state.
- Raw events are rarely the right unit of context — agents typically need
  aggregated, de-duplicated, or summarized views.
- The set of sources and the set of agents both change over time; the system must
  tolerate both sides evolving independently.

This thesis proposes and studies a **Context Management System** that sits between
event sources and AI agents, providing a durable, source-agnostic substrate for
turning raw events into agent-consumable context.

## 2. System Overview

The high-level topology (see `img.png`) has three layers:

```
 Sources (1..N)  →  Processing Engine  →  Context Engine  ←  Agents (1..M)
```

- **Sources** — heterogeneous event producers (Slack, JIRA, web analytics, …).
- **Processing Engine** — ingests, classifies, aggregates, and routes events.
  Internally it is split into a **fast path** (low-latency, per-event) and a
  **batch path** (aggregation over a window), pushing results into the Context
  Engine.
- **Context Engine** — the durable store and query surface that agents *retrieve*
  context from (and/or are notified by).
- **Agents** — consumers of context; each has its own scope of interest and its
  own memory/state.

The diagram currently shows agents *pulling* ("Retrieve") from the Context Engine;
whether a push/subscribe path should also exist is an open question (see §3.4).

## 3. Core Challenges & Research Questions

Each subsection below names an open question. These are the starting points for
the literature review and prototyping work, not settled design decisions.

### 3.1 Architectural choice: Lambda vs. Kappa

The proposed design — separate *fast queue* and *batch queue* paths — is a
**Lambda architecture**. **Kappa** collapses both paths into a single streaming
layer that handles batch processing via stream reprocessing.

**RQ1.** For a context-management workload that mixes latency-sensitive routing
with aggregation-heavy context construction, is Lambda the right choice, or does
Kappa achieve the same outcomes with less operational complexity?

Sub-questions:
- What workload properties of this system (e.g., need to re-derive aggregated
  memory from scratch, heterogeneity of sources, mixed latency SLOs) favor one
  over the other?
- Are there hybrid approaches (e.g., Kappa with a materialized "hot" topic) that
  dominate both?
- How does the choice interact with durability/replayability requirements (§4)?

### 3.2 Event classification: fast-path vs. batch-path

Once an event arrives, the Processing Engine must decide whether it needs
immediate attention (fast queue) or can be aggregated (batch queue). Sources are
heterogeneous and the "is this urgent?" signal is not uniformly encoded.

**RQ2.** How should events from arbitrary sources be classified into fast-path
vs. batch-path?

Candidate approaches to evaluate:
- **Rule-based / per-source adapters** — each source defines a simple predicate
  (e.g., JIRA priority ≥ P1, Slack message mentions `@oncall`). Cheap,
  interpretable, but brittle and requires per-source work.
- **Metadata-driven** — a common event envelope where producers self-classify;
  assumes cooperation from source systems.
- **Learned classifier** — a lightweight model (possibly distilled from an LLM)
  that scores urgency from event content + metadata. More general but introduces
  training data, drift, and latency concerns.
- **LLM-as-classifier** — cost/latency likely too high for the fast path itself,
  but usable offline to generate training labels.

The sub-question is whether a single approach generalizes, or whether the system
should expose a pluggable classification interface and let each source choose.

### 3.3 Batch aggregation strategy

The batch path's job is not just "buffer and flush." Agents need *optimal* units
of context, not a raw dump of a 5-minute window.

**RQ3.** How should events be aggregated on the batch path so that what reaches
the Context Engine is the most useful unit of context per agent class?

Dimensions to explore:
- **Windowing** — tumbling vs. sliding vs. session windows; event-time vs.
  processing-time; watermarks and late-arrival handling.
- **Aggregation semantics** — deduplication, summarization (possibly LLM-based),
  entity-level rollups (e.g., "all activity on ticket X in the last hour"),
  topic/thread reconstruction.
- **Optimality criterion** — what does "optimal" mean here? Minimizing token
  count delivered to agents? Maximizing information per token? Preserving
  causal order? This must be defined before it can be evaluated.
- **Cost** — aggregation that invokes an LLM has its own latency/cost budget.

### 3.4 Context distribution to agents

Different agents care about different events. The Context Engine must route
correctly without the Processing Engine knowing every agent.

**RQ4.** What is the right interaction model between the Context Engine and the
agents?

Options:
- **Pull / retrieve** — agents query the Context Engine when they need context.
  Simple, but requires the agent to know when to poll and what to ask for.
- **Push / pub-sub** — agents subscribe to topics/predicates; Context Engine
  pushes matching updates. Better for the latency-sensitive fast path; adds
  back-pressure and subscription-management complexity.
- **Hybrid** — push for urgent events (fast path), pull for background context
  (batch path / shared memory). This maps naturally onto the Lambda split and is
  the working hypothesis, but needs validation.

Sub-questions: how are subscriptions expressed (topic names, predicates,
semantic/embedding queries)? How are new agent types onboarded?

### 3.5 Shared agent memory as a batch-path output

One concrete instantiation of "Processing Engine output" is a **shared memory**
that all agents can read from — a durable, queryable representation of the
organization's current state, continuously updated by the batch path.

**RQ5.** How should the shared memory be structured and updated?

- Vector store, graph, relational store, or a combination?
- Update semantics: append-only log vs. in-place mutation vs. versioned
  snapshots.
- Consistency: what guarantees does an agent get about recency and
  cross-agent consistency?
- How does this interact with each agent's *private* memory (if any)?

## 4. Non-Functional Requirements

- **Durability** — no event should be silently lost; the system must support
  replay from a durable log (motivates Kafka-like substrates regardless of
  Lambda/Kappa choice).
- **Source extensibility** — adding a new source should not require changes to
  downstream components. Implies a common event envelope and an adapter layer.
- **Agent extensibility** — adding a new agent should not require changes to
  the Processing Engine.
- **Scalability** — target: thousands of events/minute sustained, with
  headroom for bursts. Revisit concrete SLOs after literature review.
- **Observability** — per-event lineage from source → classification →
  aggregation → delivery, for debugging and for evaluating §3.3's "optimality."

## 5. Explicit Non-Goals (for now)

- Building the agents themselves — this thesis studies the substrate, not the
  agent logic on top.
- Source-specific connectors beyond what is needed to demonstrate the
  architecture on 2–3 representative sources.
- Multi-tenant / cross-organization deployment.
- Security, authz, and PII handling beyond acknowledging they are required
  (they are, but they are out of scope for the core research questions).

## 6. Research Plan (initial)

1. **Literature review** per research question, starting with RQ1
   (Lambda vs. Kappa in the context of AI-agent feeding workloads).
2. **Define evaluation criteria** for each RQ — without these, comparisons
   devolve into preference.
3. **Prototype** the minimal end-to-end path: one source → Processing Engine
   with both paths → Context Engine → one agent. Use it as a testbed rather
   than a product.
4. **Experiment** per RQ on the prototype, using synthetic and (where possible)
   real event traces.

## 7. Open Terminology Issues

- "Streaming architecture" is used colloquially to mean both Lambda and Kappa;
  this document uses **Lambda** and **Kappa** precisely and reserves
  "stream processing" for the underlying compute model common to both.
- "Context" is overloaded (LLM context window vs. organizational context); this
  document means the latter — the information an agent needs to act — and will
  distinguish explicitly where confusion is possible.
