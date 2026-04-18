# Report: Context Management System for AI Agents — Research Findings

*Synthesis of RQ1–RQ5 research notes (see `research/` directory for source material). Problem-framing stage. Date: 2026-04-18.*

---

## Executive Summary

This report synthesises the answers to the five research questions (RQ1–RQ5) raised in `problem-statement.md`. Five parallel literature reviews (each ~3,000–4,500 words, combined ~170 numbered references spanning foundational systems papers, recent arXiv work, and industry documentation) converged on a surprisingly consistent architectural answer.

**Headline conclusion.** The problem statement's working design — "Lambda architecture with a fast queue and a batch queue" — is the right intuition but the wrong framing. Modern streaming engines (Flink, Spark Structured Streaming) and modern lakehouse formats (Iceberg, Delta) make the classical fast/batch codebase split unnecessary. The literature from Kreps, Akidau, Kleppmann, and the production experience at Uber, Netflix, LinkedIn, Databricks, and recently Confluent/Timeplus for AI-agent workloads, argues unanimously that the correct pattern is a **single streaming job graph with bifurcated output tiers** — not two architectures merged at query time. The thesis should adopt **Kappa+ / Streamhouse** (one pipeline, warehouse-backed replay) over Lambda.

**The five recommendations, integrated.**

1. **Architecture (RQ1).** A streaming-first Kappa+ topology: Kafka hot log + Iceberg/Delta warehouse as replay source, Flink (or Spark Structured Streaming) as the single compute engine, bifurcated sinks for low-latency push and windowed-aggregation outputs.
2. **Event classification (RQ2).** Four-stage cascade: CloudEvents envelope + per-source adapters → declarative rules (Drools/JsonLogic) → small distilled classifier (DistilBERT / fastText) → asynchronous LLM oracle for labeling and audit. Continuously re-tuned by a contextual bandit driven by downstream agent signals.
3. **Batch aggregation (RQ3).** Events aggregated into *sessionized, entity-centric Context Units* (not time-bucket dumps) using hierarchical map-reduce summarisation, indexed in a vector + graph + keyword store. An explicit utility function `U(C) = E[Performance|C] − λ·tokens(C) − μ·staleness(C)` makes "optimality" testable.
4. **Context distribution (RQ4).** Hybrid **notify-then-pull** protocol, modelled on MCP's resource-subscription primitive. The Context Engine is exposed as an MCP server; agents subscribe to typed predicates; the server pushes lightweight change signals and agents pull full payloads from resources/tools. Transactional outbox between write-side and notification-side is a correctness requirement, not optional.
5. **Shared memory (RQ5).** Bitemporal append-only event log as source of truth; derived read-side projections combine a Graphiti-style temporal knowledge graph (with GraphRAG community summaries) and a vector index over chunk/entity/community embeddings. Collaborative-Memory-style two-tier partitioning (shared vs. per-agent private) with provenance on every fragment.

**Cross-cutting themes.** The same primitives appear in all five answers: an event log as backbone, entity-centric grouping, bitemporal validity, LLM-as-oracle used offline rather than on the hot path, and MCP/pub-sub as the agent-facing protocol.

**Thesis contribution opportunities.** Three genuine gaps in the published literature emerged, each of which the thesis is positioned to address: (i) empirical benchmarking of streaming architectures on AI-agent workloads (everything current is vendor advocacy); (ii) a testable operational definition of "optimal context unit" with a full evaluation harness; (iii) a formal consistency model for multi-agent shared memory (under-specified in every surveyed system).

---

## 1. Background

The thesis proposes a **Context Management System** sitting between heterogeneous event sources (Slack, JIRA, e-commerce telemetry, observability feeds, …) and AI agents. The system ingests thousands of events per minute, routes a small latency-sensitive subset with sub-second delivery, aggregates the remainder into shared memory, and serves both through a *Context Engine* that agents interact with. See `problem-statement.md` for the full framing and the five research questions (RQ1–RQ5) that this report answers.

The research methodology was: (a) for each RQ, a dedicated literature-review agent was briefed to survey foundational, academic, and industry sources; (b) each agent produced a self-contained research note in `research/RQ{N}-*.md` with ~20–50 numbered references; (c) this document synthesises those notes. All inline citations in this report are traceable back to the per-RQ notes (see Appendix A for the unified bibliography).

---

## 2. The Proposed Integrated Architecture

The per-RQ recommendations compose into a single coherent system. At a high level:

```
┌───────────┐     ┌──────────────────────────────────────────────────┐     ┌─────────────────┐     ┌────────┐
│  Sources  │────▶│                 Processing Engine                │────▶│  Context Engine │◀──▶│ Agents │
│ (Slack,   │     │  (one Flink / Spark Structured Streaming job)    │     │   (MCP server)  │     │        │
│  JIRA,    │     │                                                  │     │                 │     │        │
│  e-com,   │     │  CloudEvents  →  4-stage classifier cascade      │     │  · temporal KG  │     │        │
│  ...)     │     │                      │                           │     │  · vector index │     │        │
│           │     │                      ├──▶ hot sink ──────────────┼────▶│  · structured   │     │        │
│           │     │                      └──▶ session-windowed,      │     │    facts store  │     │        │
│           │     │                          entity-keyed aggregator │     │  · event log    │     │        │
└───────────┘     │                          → hierarchical summary  │     │    (append-only │     │        │
                  │                          → Context Unit emit ────┼────▶│     bitemporal) │     │        │
                  │                                                  │     │                 │     │        │
                  │  Kafka hot log  ·  Iceberg/Delta replay store    │     │  notify-then-   │     │        │
                  │  (single codebase; replay = same job on warehouse)     │   pull interface│     │        │
                  └──────────────────────────────────────────────────┘     └─────────────────┘     └────────┘
```

**Data flow.** Every source emits CloudEvents-envelope messages into Kafka. A single Flink job graph performs: (i) classification via the four-stage cascade, routing ~1–5% of events to a **hot sink** (low-latency push channel) and the remainder through (ii) deduplication → entity-keyed session-windowed aggregation → LLM-based hierarchical summarisation → emission of typed *Context Units*. Both sinks write into the **Context Engine**, which exposes (a) an MCP server to agents with resources/tools for pull access and resource-subscription-based push notifications, (b) a bitemporal append-only log as source of truth, and (c) derived read-side projections (temporal knowledge graph + vector index + structured sidecar). Agents connect via MCP, declare subscriptions, receive lightweight notifications on relevant events, and pull the full payload on demand.

**What is not in this architecture (deliberately).** No separate batch codebase (Lambda's duplication is eliminated). No direct LLM inference on the hot path (too slow). No raw-event dumps into agent context windows (lost-in-the-middle, context rot). No agent-invisible coupling between sources and agents (the Processing Engine does not know which agents exist).

The rest of this report defends each component of this architecture against the alternatives, with reference to the literature.

---

## 3. Research Question Findings

### 3.1 RQ1 — Architecture (Lambda vs. Kappa vs. Hybrid)

**Paths considered** (detailed in `research/RQ1-lambda-vs-kappa.md`, 28 references):

| Path | Essence | Headline drawback |
|---|---|---|
| Classical Lambda [Marz & Warren 2015] | Separate batch + speed codebases merged at query | Code duplication; drift; operational complexity [Kreps 2014] |
| Pure Kappa [Kreps 2014] | Single stream engine; replay through Kafka from the beginning | Long-horizon replay through Kafka is cost-prohibitive [Flexera 2026] |
| Lambda+ [Gillet et al. 2021] | Category-theoretic reconstruction of Lambda | Still two engines; mostly academic |
| **Kappa+ (Uber) [Naik 2019; Fu & Soman 2021]** | Stream engine reads from warehouse for replay | Needs an Iceberg/Hive/Delta table as replay store |
| **Delta Lake / Lakehouse [Databricks 2020]** | Single streaming + batch job over ACID object-store tables | Lock-in risk to a specific stack |
| **Streamhouse (Ververica) [Ververica 2024]** | Kappa whose log *is* an Iceberg/Paimon table | Tooling maturity |
| Declarative over unified [Chronon; Beam] | Features authored once, compile to batch + stream | Real implementation is still one of the above |

**Winner.** Kappa+ / Streamhouse. The evaluation across ten dimensions (operational complexity, code duplication, reprocessing cost, latency floor, hot/cold consistency, replay horizon, cost, debuggability, evolvability, fit to mixed-latency SLOs, fit to AI-agent consumption) unanimously favours Kappa-family designs *provided* replay happens against columnar object storage rather than against unbounded Kafka retention. The single axis where Lambda historically wins — batch reprocessing speed over years of data — is neutralised by Kappa+'s warehouse-as-source.

**Why specifically for AI agents.** Kai Waehner's 2025 essay and Confluent's 2025 Real-Time Context Engine launch make an AI-specific argument: agents are non-deterministic consumers; *any* drift between hot and cold views manifests as incoherent agent behaviour. Lambda's reconciliation gap is not just an engineering nuisance in this setting — it is a correctness problem. This is the dimension on which Lambda is newly disqualified, beyond the long-standing duplication critique.

**Refinement of the problem-statement's framing.** The fast-path vs. batch-path distinction is real, but the correct implementation is *a branch in a single job graph*, not two systems. Users of the system perceive two channels; engineers maintain one codebase. This is the working pattern at Netflix Keystone, Uber, and throughout the recent event-driven-AI literature.

**Caveats.**
- If the thesis scope expanded to analytical reporting across years of strict-accounting data, Lambda's batch layer could still earn its keep.
- If fast-path SLOs tightened to single-digit milliseconds (e.g. ad bidding), a dedicated in-memory path may warrant genuine architectural separation.
- Neither caveat applies at current thesis scope.

### 3.2 RQ2 — Event Classification (Fast-path vs. Batch-path)

**Paths considered** (detailed in `research/RQ2-event-classification.md`, 52 references):

| Path | Latency | Strength | Weakness |
|---|---|---|---|
| Rule-based / per-source adapters [Drools; JsonLogic] | μs–ms | Explainable, zero training data | Brittle; scales poorly across new sources |
| Metadata-driven common envelope [CloudEvents] | ~0 | Decentralised; free per event | Requires producer cooperation; trust problem |
| Small ML classifier [DistilBERT; TinyBERT; fastText] | 1–10 ms | Generalises; handles free text | Needs labels; drift-sensitive |
| LLM-as-classifier [CARP; LLM classification survey] | 300 ms–2 s | Zero-shot; most nuanced | Too slow/expensive for hot path |
| **Hybrid cascade [FrugalGPT; RouteLLM; Unified Routing/Cascading]** | mixed | Pareto-optimal on cost/quality | More moving parts; threshold tuning |
| Learned routing from feedback [bandit; prequential accuracy] | μs (inference) | Self-improves without labels | Delayed/noisy reward; safety of exploration |

**Winner.** A four-stage hybrid cascade:

1. **CloudEvents envelope + per-source adapters** normalise events and capture obvious signals (JIRA priority, Slack @oncall) at sub-millisecond cost.
2. **Declarative rule layer** (Drools/JsonLogic) resolves unambiguous patterns with full explainability.
3. **Small distilled classifier** (DistilBERT or fastText over envelope + text features) handles ambiguous cases at p95 ≤ 10 ms.
4. **LLM oracle** (GPT-4-class) is *off the hot path* — used to label historical data for training, to audit a ~1% sample of live decisions, and to second-opinion low-confidence events on the batch path.

Plus a closed loop:

5. **Contextual bandit** driven by downstream agent outcomes (escalation, SLO breach, human override) retunes the cascade confidence thresholds and triggers re-training. Prequential accuracy with forgetting and ADWIN drift detectors monitor calibration.

**Why this design wins.** FrugalGPT demonstrates up to 98% LLM cost reduction at parity quality via cascades; RouteLLM shows ~85% cost reduction on MT-Bench; the Unified Routing/Cascading framework proves optimality conditions and delivers +4% on RouterBench. DistilBERT retains 97% of BERT quality at 60% faster inference; TinyBERT is 9.4× faster still. The asymmetry of costs — missing an urgent event is worse than a false alarm — favours recall-tuned cheap stages with conservative escalation. The LLM-oracle-as-labeler pattern is validated by PGKD (models 130× faster and 25× cheaper via LLM-driven labeling) and Zhao et al. on LLM-generated training labels.

**Why not any single approach.** Pure rules don't handle "hey someone awake?" (urgent Slack without @mention). Pure small-model ignores the long tail that only LLMs handle. Pure LLM is prohibitively slow on a per-event hot path at 10k events/min. Pure feedback routing has a cold-start problem.

**Ground truth.** True-urgency labels are *defined by downstream agent behaviour* — this is a delayed-reward / weak-label problem, aligned with Snorkel-style weak supervision plus online bandit adaptation. This is one reason the feedback loop is not optional.

### 3.3 RQ3 — Batch Aggregation

**Paths considered** (detailed in `research/RQ3-batch-aggregation.md`, 40 references):

| Path | Essence | Fit to LLM consumption |
|---|---|---|
| Raw event dump | Concatenate events in window | Terrible — "lost in the middle" [Liu et al. 2023]; context rot [Chroma 2025] |
| Tumbling windows | Fixed time buckets | Cuts conversational entities in half |
| Sliding windows | Overlapping buckets | Redundant copies; confuses agents |
| **Session windows [Akidau et al. 2015]** | Gap-closed, data-driven | Matches conversational / session entities natively |
| Stuff / Map-reduce / Refine summarisation [LangChain patterns] | Chain LLM summaries over chunks | Basic and imperfect; refine drifts; map-reduce loses structure |
| **Hierarchical / recursive summarisation [Wu et al. 2021]** | Tree-structured compression | Handles long contexts; amplifies errors across levels |
| Incremental / streaming summarisation [Chain-of-Key; SliSum] | Update structured rep as events arrive | Lower drift; more complex |
| Sketch-based aggregates [HLL; CMS; T-Digest] | Structured sidecars | Complement, don't replace, textual summary |
| Feature-store-style rollups [Tecton; Feast] | Per-entity online/offline consistent aggregates | Architecturally identical to what we want — just hadn't met LLMs |

**Winner.** A six-stage generic pipeline, specialised per source:

```
ingest
  → dedupe (MinHash + embedding similarity)
  → partition by entity_key (thread_ts / ticket_id / user_id)
  → event-time session or entity window with allowed lateness
  → enrich with sketches (HLL unique users, CMS top-terms, T-Digest values)
  → hierarchical summarise (map-reduce for long; refine for incremental)
  → LLM-judge faithfulness validate
  → emit typed Context Unit { summary, facts[], entities[], sketches,
                              valid_from, valid_to, source_refs[], version }
  → index (vector + keyword + temporal graph)
```

**Per-source specialisations.**
- **Slack**: session window on `channel_id + thread_ts`, 30-min gap, preserve speaker attribution (load-bearing for "who decided X?" queries).
- **JIRA**: keyed-state with compaction triggers (status change, N new comments, T elapsed) rather than time windows; three-level hierarchical summary (comment-cluster → phase → ticket); Graphiti-style temporal edges for every state change.
- **E-commerce**: 30-min inactivity session by `user_id`/`session_cookie`; extractive-first templated summaries (cheaper and more reliable than free generation for well-understood funnel events); LLM summarisation only for support-relevant sessions.

**Operational definition of "optimal".** The thesis should adopt the utility function:

> `U(C) = E_{t ∈ T} [Performance(agent | C, t)] − λ · tokens(C) − μ · staleness(C)`
>
> subject to `C` self-contained and temporally coherent.

Six measurable sub-criteria make this testable: token efficiency, atomic-fact recall (via LLM-as-judge faithfulness), task utility (via LongMemEval-style probes), freshness, ordering correctness (a dimension *under-measured* in the summarisation literature but critical for causal-reasoning agents), and density.

**Why the non-obvious design choices.**
- Lost-in-the-middle [Liu et al. 2023] proves U-shaped position sensitivity in LLM attention. Chroma's 2025 Context Rot study extends this across frontier models (GPT-4.1, Claude 4, Gemini 2.5, Qwen3) even on trivial retrieval tasks. More context is not monotonically better.
- LLMLingua / LongLLMLingua achieve 4–20× prompt compression with minimal (sometimes *positive*) quality effect, providing slack for active compression.
- The "Complexity Trap" paper (2025) shows simple observation masking matches LLM summarisation on some agent benchmarks — so the thesis should empirically justify summarisation costs per source rather than assuming summarisation wins.
- Ordering correctness is under-benchmarked in summarisation literature but essential for incident post-mortems and customer-journey reasoning. The thesis should introduce explicit ordering probes.

### 3.4 RQ4 — Context Distribution to Agents

**Paths considered** (detailed in `research/RQ4-context-distribution.md`, 24 references):

| Path | Essence | Limitation |
|---|---|---|
| Pull / RAG only [Lewis et al. 2020; Gao et al. survey] | Agent decides when to retrieve | Agent must *know* when to poll; pathological for urgent-event response |
| Pure push / pub-sub [Eugster et al. 2003] | Broker pushes all matching events | Backpressure and fan-out hard; bloats per-agent bandwidth |
| Content-based pub/sub [SIENA; EventBridge; NATS] | Subscribe by predicate over payload | More expressive; harder to scale |
| Semantic / embedding subscription [StreamingRAG; CQL] | Subscribe by embedding similarity | Immature; no major vector DB offers native subscribe-to-embedding primitive in 2026 |
| CQRS [Fowler] | Write-path events + read-path projections | Architectural pattern, needs concrete protocol |
| **Notify-then-pull [MCP `resources/subscribe`]** | Push lightweight change signal; pull rich payload | Best-documented hybrid; needs transactional outbox |

**Winner.** Hybrid **notify-then-pull**, implemented as the Context Engine being an MCP server:

- **Notification plane**: MCP's `resources/subscribe` + `notifications/resources/updated` (server pushes `{resource_uri, version, trigger_event_id, reason}` — not the payload).
- **Pull plane**: MCP `resources/read` and `tools/*` for retrieval functions (`search_context`, `get_entity_timeline`, `semantic_search`).
- **Subscription descriptor**: NATS-style topic hierarchy + EventBridge-style JSON content predicate + optional future semantic clause.
- **Delivery semantics**: at-least-once; durable cursor per subscription (Kafka consumer group / Pulsar Key_Shared model); per-subscription `max_in_flight` and overflow policy (drop-oldest / drop-duplicate / sample-latest) for backpressure.
- **Transactional outbox** [Richardson] between the write side (Processing Engine) and the notification side — *non-negotiable*, without it an agent gets notified of an event that it then fails to pull.

**Why this validates the thesis's working hypothesis — with two refinements.** The problem statement's intuition ("push urgent; pull aggregated") is correct and is independently converged on by: MCP's design, Google's A2A, Confluent's event-driven-multi-agent patterns, AWS's real-time-RAG architectures, CQRS, and outbox. But the survey produced two refinements:

1. **Fast-path pushes *signals*, not full events.** Pushing raw events to every subscribed agent duplicates work and bloats bandwidth. The notification carries `event_id` + metadata; the agent pulls the full event from the Context Engine.
2. **Batch-path outputs should also be pushable when they materially change.** "Push = fast; pull = batch" is too crisp a split. A rolling customer-summary may be worth pushing to subscribed agents on material change (measured by token-diff / embedding distance / downstream-relevance). The architecture must not lock out this case.

**Why not pure semantic subscriptions (yet).** None of Pinecone / Weaviate / Milvus / Qdrant exposes a native "subscribe to an embedding" primitive as of 2026. Real-time on the write side (insert/upsert) exists; real-time on the notify side does not. Content-based predicates are the safe default; reserve an extension point for semantic filters evaluated by a lightweight classifier.

**Agent-side implications.** MCP was designed for tool-using chat assistants, not long-running server-subscribed agents. Scaling MCP to thousands of long-lived subscriptions is an open question. A likely answer: Kafka (or NATS/Pulsar) as the internal bus, MCP as the external client façade.

### 3.5 RQ5 — Shared Memory

**Paths considered** (detailed in `research/RQ5-shared-memory.md`, 27 references):

| Path | Examples | Strength | Weakness |
|---|---|---|---|
| Hierarchical tiers | MemGPT / Letta | OS-style virtual memory | Mutable core limits multi-agent consistency |
| Memory stream | Generative Agents | Simple; reflection-driven | Flat vector retrieval limits relational queries |
| Vector-only | Mem0 | Fast, simple | Poor on multi-hop reasoning; LongMemEval regression |
| Agentic Zettelkasten | A-MEM | Self-organising links | Link rewriting hard under concurrent writers |
| Graph / Knowledge graph | HippoRAG, GraphRAG, LightRAG, PathRAG | Multi-hop reasoning, explicit provenance | Write throughput lower than vector |
| **Temporal / bitemporal graph** | Zep / Graphiti | Answers "as of" queries; supersession | Ingestion expense (LLM entity extraction) |
| Hybrid vector + graph | Cognee, LightRAG, Mem0g | Emerging production default | Complexity |
| Service-oriented | MaaS; Collaborative Memory | Governed access; multi-agent-ready | Few implementations; recent |
| Blackboard | 1970s pattern revisited | Conceptually simple | Weaker at relational querying |

**Winner.** A hybrid layered design with explicit source-of-truth / projections split:

**Source of truth.** An append-only event log (Kafka topic or Iceberg table — same backbone as RQ1) carrying every batch-path emission with `event_time` (business-event time) + `ingest_time` (system time). **Bitemporal by construction** [Fowler] because the Processing Engine already produces both timestamps.

**Read-side projections.**
- A **Graphiti-style temporal knowledge graph** with entities, relations, and `(t_valid_from, t_valid_to, t_ingested)` on every edge. Supersession (new fact invalidating old) sets `t_valid_to`; the old edge remains queryable — answering "what did we believe on day X?".
- GraphRAG-style **community summaries** pre-generated for coarse queries.
- A **vector index** over chunk / entity / community embeddings (pgvector for prototype; Qdrant/Milvus for scale).
- A **relational sidecar** (Postgres) for well-known structured facts (user profiles, ticket metadata), kept in sync *via the log*, not by direct writes.

**Multi-agent access.** Two-tier [Collaborative Memory 2025]: `shared` (written by the batch path; readable by all) and `private:agent_X` (writable only by agent X). Governed at the Context Engine service boundary [MaaS 2025]. Every fragment carries provenance — enabling retrospective audit, PII redaction, and consent revocation.

**Consistency guarantees delivered.**
- Read-your-writes for private memory (trivial: single writer).
- Monotonic reads for shared memory (projection state moves forward).
- Bounded staleness across agents (measured SLO; target p99 < a few seconds).
- **No multi-agent transactions.** Cross-agent coordination (two agents must not both claim a ticket) uses an explicit lock/queue primitive *outside* the memory layer.

**Forgetting.** The log is never forgotten (cheap storage; auditability non-negotiable). Projections *can* forget via periodic reflection-style compaction, gated by regression tests on a held-out benchmark. Low-salience facts decay in retrieval *score* (recency × importance × relevance weighting from Generative Agents) rather than being deleted.

**Why this specific combination.** Zep/Graphiti produced the strongest empirical memory results in 2025 (18.5% accuracy gain on LongMemEval, 90% latency reduction, ~115K tokens → ~1.6K). GraphRAG added community summaries for coarse queries. Collaborative Memory is the only published shared-vs-private design for multi-agent settings. Event sourcing + bitemporality are mature patterns [Fowler] with direct mapping to "batch path emits events, agents see projections" topology.

**The gap the survey surfaced.** No surveyed system formally specifies the consistency model offered to concurrent agent readers/writers. Graphiti uses "last-write-wins with explicit supersession"; Collaborative Memory uses permission graphs; most systems just don't say. This is a thesis contribution opportunity (see §6).

---

## 4. Cross-Cutting Themes

Five primitives appear consistently across the RQ answers; they are the non-negotiable core of the integrated architecture.

**4.1 Event log as the backbone.** RQ1 (Kappa+ hot log + warehouse replay), RQ3 (event-time session windows over the log), RQ4 (transactional outbox + durable subscription cursors), and RQ5 (append-only log as source of truth) all assume the same thing: a durable, ordered, replayable log is the spine of the system. This is Kleppmann's "log as source of truth" thesis and maps cleanly onto Kafka + Iceberg/Delta.

**4.2 Bitemporal validity.** RQ3 defines Context Units as carrying `(valid_from, valid_to)`. RQ5 makes bitemporality (valid-time + system-time) the consistency model for shared memory. RQ4's notify-then-pull uses version + trigger_event_id so a notification can be correlated to the specific log entry that produced it. Every RQ benefits when time is represented explicitly as two dimensions.

**4.3 Entity-centric grouping.** RQ3 groups aggregation by conceptual entity (thread, ticket, customer session). RQ5 structures shared memory around entities with relations. RQ4's subscription scopes include entity predicates. The recurring pattern: **group by entity first, window within entity, link across entities via graph**.

**4.4 LLM-as-oracle used offline, not online.** RQ2 confines LLMs to offline labeling and audit (too slow on hot path). RQ3 uses LLM-as-judge for faithfulness evaluation in a validation step, not a serving step. RQ5 uses LLM-driven entity extraction at ingestion — still offline-batch. Nowhere is an LLM placed on the per-event fast path. This is consistent with FrugalGPT-style cost-quality optimisation and with the SLM-first arguments in recent work.

**4.5 MCP as the external protocol surface.** RQ4 selects MCP as the client-facing protocol. RQ5 implements the shared memory behind an MCP server. The CloudEvents envelope from RQ2 pairs naturally with MCP resources. MCP is not yet mature for thousands of long-lived subscriptions, but the literature's direction of travel (Anthropic + Google A2A + Linux Foundation governance) makes it the right bet for a 2026 thesis.

---

## 5. Tensions and Refinements to the Problem Statement

Five specific refinements to `problem-statement.md` emerged from the research.

**5.1 "Lambda architecture" is the wrong framing.** The problem statement (§3.1 and §2) describes a Lambda-style fast/batch split. RQ1 finds that the correct implementation is a *single streaming job with two output tiers*, not two architectures. The problem statement should be updated to reflect that "fast path + batch path" is a *topology* decision *inside* a Kappa+ / Streamhouse architecture, not a Lambda-vs-Kappa choice.

**5.2 "Optimal context" needs a formal definition before it can be studied.** Problem-statement §3.3 asks "what does 'optimal' mean?" RQ3 answers: `U(C) = E[Performance|C] − λ·tokens(C) − μ·staleness(C)` subject to self-containedness and temporal coherence. Six measurable sub-criteria make it testable. The problem statement should adopt this definition.

**5.3 Push vs. pull is not a clean split along fast/batch.** Problem-statement §3.4's working hypothesis ("push for urgent; pull for background") is correct in spirit but oversimplified. RQ4 shows that (a) the "push" should be a lightweight signal, not a payload; (b) batch-path outputs should also support push on material change, not be pull-only. The architecture should support push/pull on both fast and batch outputs.

**5.4 Classification is a multi-stage problem, not a single-model decision.** Problem-statement §3.2 lists rule, metadata, ML, LLM as alternatives. RQ2 shows they are *complements*, layered as a cascade. The problem statement should be updated to frame the question as "what is the right cascade?" rather than "which single approach?".

**5.5 Multi-agent consistency is a first-class concern, not a noted non-goal.** Problem-statement §4 mentions consistency implicitly; RQ5 shows that multi-agent memory consistency is under-specified across the entire published literature and is a concrete contribution opportunity. The problem statement should promote it.

---

## 6. Thesis Contribution Opportunities

Three gaps where the thesis can produce genuinely novel work, not just implement consensus.

**6.1 Empirical streaming-architecture benchmark on AI-agent workloads.** All existing Lambda-vs-Kappa literature predates agentic AI. The 2025 Confluent/Timeplus/Waehner pieces are vendor advocacy without peer-reviewed evaluation. The thesis could construct a representative trace (mixed Slack / JIRA / e-commerce) and measure hot-cold drift, reprocessing cost, and agent-behaviour reproducibility across architectures. This would be the first empirical paper on the topic.

**6.2 Evaluation methodology for "optimal context units".** There is no published protocol for comparing aggregation strategies on AI-agent workloads without ground truth. RQ3 proposes combining (a) LLM-as-judge faithfulness for factuality, (b) LongMemEval-style probe tasks for task utility, (c) counterfactual A/B on agent accuracy with vs. without the unit. Operationalising this into a reproducible harness (with released probe sets per source type) would itself be a thesis contribution.

**6.3 Consistency model for multi-agent shared memory.** RQ5 flags this as unspecified in every surveyed system. The thesis could (a) formalise a consistency model (e.g., bitemporal monotonic reads with bounded staleness + provenance-aware supersession), (b) measure the actual consistency the proposed implementation delivers under multi-agent concurrent load, (c) characterise failure modes when the guarantees are violated. This is open both empirically and formally.

A secondary contribution area — less novel but valuable — is operationalising the four-stage classifier cascade with a feedback loop driven by downstream agent signals. Production cascades exist (FrugalGPT, RouteLLM) but the feedback-from-agent-outcomes closed-loop version is under-engineered.

---

## 7. Consolidated Open Questions

Questions deferred to implementation / experiment phases:

**On architecture (RQ1).**
- Cost model for LLM-in-the-loop aggregation in a Flink job. Flink's optimiser was not designed for side-effectful high-latency UDFs.
- Concrete replay SLOs for rebuilding shared memory when aggregation logic changes.

**On classification (RQ2).**
- Label sourcing from delayed/biased downstream signals.
- Cross-source context for classification (Slack needs JIRA state).
- Drift-sensitive threshold tuning cadence.
- Per-agent vs. global urgency definitions.
- Producer self-classification governance.
- Safety of bandit exploration.
- Cold start on new sources.

**On aggregation (RQ3).**
- Compression-ratio vs. task-utility curve per source type.
- Hallucination amplification across hierarchy levels.
- When summarisation is not worth it (Complexity Trap null hypothesis).
- Cross-entity Context Units.
- Update semantics for long-lived entities (JIRA): rebuild vs. append-and-refine.

**On distribution (RQ4).**
- Semantic-predicate evaluation cost on the fast path.
- Push-on-material-change threshold definition.
- Duplicate suppression when same event flows through both paths.
- MCP's scaling behaviour to thousands of long-lived subscriptions.
- MCP-as-transport vs. Kafka-behind-MCP.

**On shared memory (RQ5).**
- Concurrent-writer conflict resolution (CRDT vs. LWW + supersession).
- Semantic supersession adjudication under load.
- Compaction-quality metrics without an omniscient oracle.
- Cost ceiling on LLM-driven ingestion (entity/relation extraction).

---

## 8. Next Steps

1. **Update `problem-statement.md`** to reflect the five refinements in §5.
2. **Pick one RQ to deepen.** RQ1 is the natural first, since the architectural choice frames the rest. However, given that the literature consensus is strong on RQ1 (Kappa+), a higher-impact second pass might be on **RQ3 evaluation methodology** or **RQ5 multi-agent consistency** — the two genuine contribution gaps.
3. **Build a prototype.** Minimal end-to-end path: Kafka + Flink + one source (Slack is cheapest) + Context Engine skeleton (pgvector + a graph store) + one agent. Use this as the testbed for §6's evaluation harnesses.
4. **Continue the research loop.** The 2025/2026 literature on agent memory and event-driven AI is moving fast; schedule a rolling update on arXiv, Confluent, and Anthropic for new material. Key arXiv categories: cs.DC (distributed systems), cs.CL + cs.AI (agent memory), cs.DB (streaming).

---

## Appendix A — Unified Bibliography

Consolidated across all five research notes. Entries are grouped by topic for readability. Reference numbers in the format `[RQn.m]` denote the citation's original numbering inside `research/RQ{n}-*.md`, where each note has its own self-contained bibliography.

### A.1 Streaming architectures & foundational systems

- Akidau, T. (2015). *Streaming 101 / Streaming 102: The world beyond batch*. O'Reilly Radar. [RQ1.3, RQ3.7] <https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/>
- Akidau, T., Bradshaw, R., Chambers, C., et al. (2015). *The Dataflow Model*. VLDB 2015. [RQ3.1] <https://www.vldb.org/pvldb/vol8/p1792-Akidau.pdf>
- Akidau, T., Chernyak, S., & Lax, R. (2018). *Streaming Systems*. O'Reilly. [RQ1.23, RQ3.7]
- Apache Beam. *Basics of the Beam model*. [RQ1.27, RQ3.18] <https://beam.apache.org/documentation/basics/>
- Apache Flink. *Flink Windows (master docs)*. [RQ3.2] <https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/windows/>
- Apache Flink blog (2018). *End-to-End Exactly-Once Processing in Apache Flink*. [RQ3.23] <https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/>
- Begoli, E., Akidau, T., Hueske, F., et al. (2021). *Watermarks in Stream Processing Systems*. PVLDB 14(12). [RQ1.16, RQ3.17] <http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf>
- Confluent (2018). *Exactly-once Semantics is Possible*. [RQ3.22] <https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/>
- Databricks (2020). *Delta vs. Lambda: Why Simplicity Trumps Complexity for Data Pipelines*. [RQ1.20] <https://www.databricks.com/blog/2020/11/20/delta-vs-lambda-why-simplicity-trumps-complexity-for-data-pipelines.html>
- Decodable. *Understanding Apache Flink Event Time and Watermarks*. [RQ3.3] <https://www.decodable.co/blog/understanding-apache-flink-event-time-and-watermarks>
- Flexera (2026). *Kappa Architecture 101: Deep dive into stream-first design*. [RQ1.15] <https://www.flexera.com/blog/finops/kappa-architecture/>
- Fu, Y., & Soman, C. (2021). *Real-time Data Infrastructure at Uber*. SIGMOD '21. arXiv:2104.00087. [RQ1.6] <https://arxiv.org/abs/2104.00087>
- Gillet, A., Leclercq, É., & Cullot, N. (2021). *Lambda+: Category Theory to the Rescue*. CAiSE 2021. [RQ1.21]
- Kleppmann, M. (2015). *Turning the database inside-out with Apache Samza*. [RQ1.9] <https://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html>
- Kleppmann, M. (2016). *Making Sense of Stream Processing*. O'Reilly. [RQ1.25]
- Kleppmann, M. (2017/2025). *Designing Data-Intensive Applications*, 2nd ed., ch. 11. O'Reilly. [RQ1.26]
- Kreps, J. (2014). *Questioning the Lambda Architecture*. O'Reilly Radar. [RQ1.1] <https://www.oreilly.com/radar/questioning-the-lambda-architecture/>
- LinkedIn Engineering (2017, 2023). *Streaming Data Pipelines with Brooklin; Revolutionizing Real-Time Stream Processing*. [RQ1.8]
- Marz, N., & Warren, J. (2015). *Big Data*. Manning. [RQ1.4]
- Naik, R. (2019, 2021). *Designing a Production-Ready Kappa Architecture for Timely Data Stream Processing*. Uber Engineering. [RQ1.5] <https://www.uber.com/us/en/blog/kappa-architecture-data-stream-processing/>
- Netflix Technology Blog (2018). *Keystone Real-time Stream Processing Platform*. [RQ1.17]
- Netflix Technology Blog (2016). *Stream-processing with Mantis*. [RQ1.18]
- Noghabi, S. A., et al. (2017). *Samza: Stateful Scalable Stream Processing at LinkedIn*. PVLDB 10(12). [RQ1.7]
- Streamkap (2024). *The Kappa Architecture: Simplifying Data Pipelines with Streaming*. [RQ1.2]
- Uber Engineering. *Michelangelo Palette: A Feature Engineering Platform at Uber*. [RQ1.22]
- Ververica (2024). *From Kappa Architecture to Streamhouse*. [RQ1.24] <https://www.ververica.com/blog/from-kappa-architecture-to-streamhouse-making-lakehouses-real-time>

### A.2 AI-agent-specific context infrastructure

- Confluent (2025). *Introducing Real-Time Context Engine for AI*. [RQ1.11] <https://www.confluent.io/blog/introducing-real-time-context-engine-ai/>
- Confluent (2025). *Launches Confluent Intelligence to Solve the AI Context Gap*. [RQ1.12]
- Confluent. *A Guide to Event-Driven Design for Agents and Multi-Agent Systems*. [RQ4.18]
- CortexFlow (2024). *Revisiting the Lambda Architecture*. [RQ1.14]
- Timeplus (2025). *Real-Time Streaming as the Context Layer for AI Agents*. [RQ1.13] <https://www.timeplus.com/post/context-layer-for-ai-agents>
- Waehner, K. (2025). *The Rise of Kappa Architecture in the Era of Agentic AI*. [RQ1.19] <https://www.kai-waehner.de/blog/2025/07/08/the-rise-of-kappa-architecture-in-the-era-of-agentic-ai-and-data-streaming/>
- Waehner, K. (2026). *How Apache Kafka and Flink Power Event-Driven Agentic AI in Real Time*. [RQ2.21]

### A.3 Event envelopes and routing

- AWS. *EventBridge event pattern syntax and content-based filtering*. [RQ2.32, RQ4.17]
- AWS. *Sending and receiving CloudEvents with Amazon EventBridge*. [RQ2.32]
- Azure Event Grid — *CloudEvents Integration*. [RQ2.31]
- CNCF. *CloudEvents v1.0.2 Specification*. [RQ2.3]
- Google Cloud. *CloudEvents format — Eventarc docs*. [RQ2.30]

### A.4 Rule engines and complex event processing

- Chen, Z. (2019). *A Distributed Rule-Based Expert System for Large Event Stream Processing*. PhD thesis. [RQ2.29]
- Drools. *Rule Engine Documentation*. [RQ2.1]
- HPE Developer. *Better Complex Event Processing at Scale Using a Microservices-based Streaming Architecture*. [RQ2.27]
- JsonLogic. *Rules in JSON*. [RQ2.2]
- Microsoft. *RulesEngine — A JSON based Rules Engine*. [RQ2.28]
- Red Hat / KIE. *Event Driven Drools: CEP Explained*. [RQ2.26]

### A.5 Classifiers, cascades, and routing

- Aberdeen, D., Pacovsky, O., Slater, A. (2010). *The Learning Behind Gmail Priority Inbox*. NIPS 2010 workshop. [RQ2.35]
- Arora, M. et al. (2024). *Concept drift detection and adaptation*. WIREs DMKD. [RQ2.18]
- Belcak, P., Heinrich, G. et al. (2025). *Small Language Models are the Future of Agentic AI*. arXiv:2506.02153. [RQ2.15]
- Bucher, M., Martini, M. (2024). *Large Language Models for Text Classification: Case Study and Comprehensive Review*. arXiv:2501.08457. [RQ2.8]
- Chen, L., Zaharia, M., Zou, J. (2023). *FrugalGPT*. arXiv:2305.05176. [RQ2.11]
- *Cascadia: An Efficient Cascade Serving System for LLMs* (2025). arXiv:2506.04203. [RQ2.39]
- *Cost-Saving LLM Cascades with Early Abstention* (2025). arXiv:2502.09054. [RQ2.42]
- Datadog. *Detect anomalies with Datadog AIOps*. [RQ2.23]
- Dekoninck, J. et al. (2024). *A Unified Approach to Routing and Cascading for LLMs*. arXiv:2410.10347. [RQ2.14]
- DoorDash Engineering (2024). *Building scalable real time event processing with Kafka and Flink*. [RQ2.20]
- Dutta, A. et al. (2020). *Detecting Urgency Status of Crisis Tweets*. COLING 2020. [RQ2.25]
- *Dynamic Model Routing and Cascading: A Survey* (2026). arXiv:2603.04445. [RQ2.38]
- *Early-exit Convolutional Neural Networks* (2024). arXiv:2409.05336. [RQ2.40]
- *An Efficient Inference Framework for Early-exit LLMs* (2024). arXiv:2407.20272. [RQ2.41]
- Fang, B. et al. (2024). *Smart Expert System*. arXiv:2405.10523. [RQ2.37]
- Foster, D.J., Rakhlin, A. (2020). *Beyond UCB: Contextual Bandits with Regression Oracles*. ICML 2020. [RQ2.46]
- Guo, Y., Xu, Z. (2024). *Online Learning in Bandits with Predicted Context*. AISTATS 2024. [RQ2.17]
- Ho, S. et al. (2023). *A Survey of Time Series Anomaly Detection Methods in AIOps*. arXiv:2308.00393. [RQ2.52]
- Jiao, X. et al. (2019). *TinyBERT*. arXiv:1909.10351. [RQ2.6]
- Jin, T. et al. (2024). *C³: Confidence Calibration Model Cascade*. arXiv:2402.15991. [RQ2.13]
- Joulin, A. et al. (2016/2017). *Bag of Tricks for Efficient Text Classification (fastText)*. EACL 2017. [RQ2.4]
- Label Your Data (2026). *SLM vs LLM: Accuracy, Latency, Cost Trade-Offs 2026*. [RQ2.51]
- Liu, W. et al. (2020). *FastBERT*. arXiv:2004.02178. [RQ2.33]
- Ong, I. et al. (2024). *RouteLLM: Learning to Route LLMs with Preference Data*. arXiv:2406.18665. [RQ2.12]
- *Online Cascade Learning for Efficient Inference over Streams* (2024). arXiv:2402.04513. [RQ2.43]
- PagerDuty. *Event Intelligence — Intelligent Triage; AIOps for Incident Management*. [RQ2.22, RQ2.50]
- *Performance-Guided LLM Knowledge Distillation (PGKD)*. arXiv:2411.05045. [RQ2.10]
- Ratner, A. et al. (2019). *Snorkel: Rapid Training Data Creation with Weak Supervision*. arXiv:1711.10160. [RQ2.49]
- *Scalable and Interpretable Contextual Bandits* (2025). arXiv:2505.16918. [RQ2.47]
- Sachdeva, N. S., Chandrasekhar, V. (2020). *On detecting urgency in short crisis messages*. SNAM 10(70). [RQ2.24]
- Sanh, V. et al. (2019). *DistilBERT*. arXiv:1910.01108. [RQ2.5]
- Sun, X. et al. (2023). *Text Classification via LLMs (CARP)*. arXiv:2305.08377. [RQ2.7]
- *Text Classification in the LLM Era — Where do we stand?* (2025). arXiv:2502.11830. [RQ2.36]
- *Universal Model Routing for Efficient LLM Inference* (2025). arXiv:2502.08773. [RQ2.48]
- Wilson, J. et al. (2024). *Multi-armed bandit based online model selection for concept-drift adaptation*. Expert Systems. [RQ2.16, RQ2.45]
- Zhao, W. et al. (2024). *Knowledge Distillation in Automated Annotation*. arXiv:2406.17633. [RQ2.9]
- Žliobaitė, I. et al. (2017). *Concept drift in Streaming Data Classification*. Procedia CS 132. [RQ2.19]

### A.6 Aggregation, summarisation, and prompt compression

- Chroma Research (2025). *Context Rot: How Increasing Input Tokens Impacts LLM Performance*. [RQ3.5]
- *Context-Aware Hierarchical Merging for Long Document Summarization* (2025). arXiv:2502.00977. [RQ3.27]
- *Dynamic Tree Construction for Recursive Summarization* (ACL 2025). [RQ3.28]
- *Enhancing Incremental Summarization with Structured Representations* (2024). arXiv:2407.15021. [RQ3.29]
- Evidently AI. *LLM-as-a-judge: a complete guide*. [RQ3.15]
- Feast. *What is a Feature Store?* [RQ3.10]
- Google Cloud. *Cloud Dataflow support for HyperLogLog++*. [RQ3.19]
- Jiang, H. et al. (2023). *LLMLingua*. EMNLP. arXiv:2310.05736. [RQ3.13]
- Jiang, H. et al. (2023). *LongLLMLingua*. arXiv:2310.06839. [RQ3.37]
- LangChain. *load_summarize_chain (stuff / map_reduce / refine)*. [RQ3.26]
- Liu, N. F. et al. (2023/2024). *Lost in the Middle: How Language Models Use Long Contexts*. TACL 12. [RQ3.4]
- Milvus. *MinHash LSH in Milvus*. [RQ3.24]
- QuestDB. *Sketch Algorithm* (T-Digest, HLL, Bloom). [RQ3.21]
- Redis. *Count-Min Sketch*. [RQ3.20]
- Redis (2026). *LLM Token Optimization*. [RQ3.14]
- *Scaling LLM Multi-turn RL with Summarization-based Context Management* (2025). arXiv:2510.06727. [RQ3.31]
- *SliSum: Faithfulness via Sliding Generation and Self-Consistency* (2024). arXiv:2407.21443. [RQ3.30]
- Tecton. *Real-Time Aggregation Features for Machine Learning (Part 1)*. [RQ3.9]
- *The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization* (2025). arXiv:2508.21433. [RQ3.32]
- Vertica. *Sessionization with event-based windows*. [RQ3.35]
- Weaviate. *Chunking Strategies to Improve LLM RAG Pipeline Performance*. [RQ3.12]
- Wu, J. et al. (2021). *Recursively Summarizing Books with Human Feedback*. arXiv:2109.10862. [RQ3.6]
- Wu, X. et al. (2024/2025). *LongMemEval*. ICLR 2025. arXiv:2410.10813. [RQ3.16, RQ5.14]
- Xiao, G. et al. (2023/2024). *StreamingLLM (Attention Sinks)*. ICLR 2024. [RQ3.40]
- Zilliz. *Data Deduplication at Trillion Scale*. [RQ3.25]

### A.7 Agent memory systems

- *Axoniq — Event sourcing / CQRS for agent-ready systems*. [RQ5.15] <https://www.axoniq.io/>
- Chen, B. et al. (2025). *PathRAG*. arXiv:2502.14902. [RQ5.13]
- *Cognee — Knowledge engine for AI agent memory*. [RQ5.5] <https://github.com/topoteretes/cognee>
- Edge, D. et al. (2024). *GraphRAG: From Local to Global*. arXiv:2404.16130. [RQ5.11]
- *Exploring Advanced LLM Multi-Agent Systems Based on Blackboard Architecture* (2025). arXiv:2507.01701. [RQ5.27]
- Fowler, M. (2005, 2021). *Event Sourcing; Bitemporal History*. [RQ5.9, RQ5.10]
- *Graphiti — Build Real-Time Knowledge Graphs for AI Agents*. getzep. [RQ5.20]
- *Graphiti: Knowledge Graph Memory for an Agentic World* (Neo4j blog, 2025). [RQ5.25]
- Gutierrez, B. J. et al. (2024). *HippoRAG*. NeurIPS 2024. arXiv:2405.14831. [RQ5.6]
- *Hindsight is 20/20: Building Agent Memory*. arXiv:2512.12818. [RQ3.11]
- *LightRAG* (2024). arXiv:2410.05779. [RQ5.12]
- *Letta — Stateful agent platform (production MemGPT)*. [RQ5.24] <https://docs.letta.com/concepts/letta/>
- Maharana, A. et al. (2024). *LoCoMo: Evaluating Very Long-Term Conversational Memory*. ACL 2024. arXiv:2402.17753. [RQ5.8]
- *Memory as a Service (MaaS)* (2025). arXiv:2506.22815. [RQ5.22]
- *Memory in the Age of AI Agents* (2025). arXiv:2512.13564. [RQ5.23, RQ1.28]
- *MemoryBench* (2025). arXiv:2510.17281. [RQ5.21]
- *Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory*. arXiv:2504.19413. [RQ5.4, RQ3.33]
- Packer, C. et al. (2023). *MemGPT: Towards LLMs as Operating Systems*. arXiv:2310.08560. [RQ5.1, RQ3.38]
- Park, J. S. et al. (2023). *Generative Agents: Interactive Simulacra of Human Behavior*. UIST 2023. arXiv:2304.03442. [RQ5.7]
- Rasmussen, P. et al. (2025). *Zep: A Temporal Knowledge Graph Architecture for Agent Memory*. arXiv:2501.13956. [RQ5.3, RQ3.8]
- Rezazadeh, A. et al. (2025). *Collaborative Memory: Multi-User Memory Sharing in LLM Agents with Dynamic Access Control*. arXiv:2505.18279. [RQ5.17]
- Shapiro, M. et al. (2011). *Conflict-free Replicated Data Types*. INRIA. [RQ5.16]
- Xu, W. et al. (2025). *A-MEM: Agentic Memory for LLM Agents*. arXiv:2502.12110. [RQ5.2]
- Zhang, Z. et al. (2024). *A Survey on the Memory Mechanism of LLM-based Agents*. ACM TOIS. arXiv:2404.13501. [RQ5.18]

### A.8 Databases and storage

- *AI Knowledge Graph Platforms: Neo4j, TigerGraph, AWS Neptune*. SkillUp 2025. [RQ5.26]
- AWS. *Real-time clickstream sessions with Kinesis, Glue, Athena*. [RQ3.36]
- LiquidMetal AI (2025). *Vector Database Comparison: Pinecone vs Weaviate vs Qdrant vs Milvus vs FAISS*. [RQ5.19]

### A.9 Distribution, pub/sub, and protocols

- Anthropic / MCP project. *MCP Specification (2025-11-25)*. [RQ4.4] <https://modelcontextprotocol.io/specification/2025-11-25>
- Apache Kafka. *Consumer Groups and Consumer API*. [RQ4.12]
- Apache Pulsar. *Subscription Types*. [RQ4.23]
- Arasu, A., Babu, S., Widom, J. *The CQL Continuous Query Language*. Stanford InfoLab. [RQ4.24]
- AWS Architecture Diagrams. *Real-time Streaming for RAG*. [RQ4.22]
- Carzaniga, A. et al. *SIENA and content-based routing*. INFOCOM 2004. [RQ4.7]
- Eugster, P. T. et al. (2003). *The Many Faces of Publish/Subscribe*. ACM Computing Surveys 35(2). [RQ4.3]
- Fowler, M. *CQRS*. [RQ4.10]
- Gao, Y. et al. (2023). *Retrieval-Augmented Generation for LLMs: A Survey*. arXiv:2312.10997. [RQ4.2]
- Google et al. *Agent2Agent (A2A) Protocol Specification*. [RQ4.5]
- LangChain. *LangGraph Graph API overview*. [RQ4.8]
- Lewis, P. et al. (2020). *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*. NeurIPS 2020. [RQ4.1]
- LlamaIndex. *Agents, Query Engines, and Tools*. [RQ4.20]
- Microsoft. *AutoGen Multi-agent Conversation Framework*. [RQ4.14]
- Model Context Protocol. *Transports (Streamable HTTP + SSE) and Resource subscriptions/notifications*. [RQ4.16]
- NATS. *Subject-Based Messaging*. [RQ4.13]
- OpenAI. *Swarm: lightweight multi-agent orchestration*. [RQ4.15]
- Richardson, C. *Pattern: Transactional Outbox*. [RQ4.11]
- Sankaradas, M. et al. (2025). *StreamingRAG: Real-time Contextual Retrieval and Generation Framework*. arXiv:2501.14101. [RQ4.21]
- Shen, H. *Content-based Publish/Subscribe Systems* (book chapter). [RQ4.6]
- Singh, A. et al. (2025). *Agentic Retrieval-Augmented Generation: A Survey*. arXiv:2501.09136. [RQ4.19]
- Striim. *Real-Time RAG: Streaming Vector Embeddings*. [RQ4.22]

---

*End of report. For the per-RQ detailed research notes and their individual references, see `research/RQ{1..5}-*.md` in this directory.*
