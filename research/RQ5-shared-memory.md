# RQ5 — Shared Memory for Multi-Agent Context Management

**Research question.** How should the shared memory accessed by multiple AI agents
be structured (vector / graph / relational / hybrid), updated (append-only,
mutable, versioned), and made consistent across agents?

**Scope.** This note surveys the state of LLM-agent memory research as of
April 2026, with particular attention to multi-agent (rather than single-user)
settings. It feeds the RQ5 section of the thesis — the shared memory that the
batch path of the proposed Context Management System writes, and that agents
read from.

---

## 1. Executive Summary

Agent-memory research has converged, over roughly 2023–2026, on a small set of
architectural ideas:

1. **Hierarchical tiers** (MemGPT / Letta) — context-window-as-RAM, with
   explicit eviction into archival stores [1, 24].
2. **Vector retrieval over a memory stream** as the retrieval baseline
   (Generative Agents, baseline RAG) [7, 15, 16].
3. **Graph memory** for relational / multi-hop queries, typically a
   temporally-annotated knowledge graph (Zep/Graphiti, HippoRAG, GraphRAG,
   LightRAG, PathRAG, Cognee) [3, 6, 11, 12, 13, 25].
4. **Hybrid vector + graph** as the emerging default for production
   (Zep, Cognee, Mem0g, LightRAG) [3, 4, 12, 25].
5. **Bitemporal, append-only event log as source of truth**, with derived
   vector and graph indexes as query-side projections [9, 10, 20].
6. **Dual-tier shared / private memory with provenance and access control**
   for multi-agent settings (Collaborative Memory, Memory-as-a-Service) [17, 22].

For the thesis's batch-path-feeds-shared-memory topology, the recommended
architecture is:

> **Bitemporal append-only event log (source of truth) → derived hybrid index
> combining a temporal knowledge graph (entities, relations, community
> summaries) and a vector store over chunk / summary embeddings, with a
> two-tier (shared / private) partitioning, provenance on every fragment, and
> reflection-driven compaction.**

This is essentially the Zep/Graphiti pattern [3, 25], extended with the
Collaborative Memory [17] two-tier access model, sitting behind the batch
path of a Lambda architecture (see RQ1).

---

## 2. Background: Taxonomy of LLM Memory Architectures

The recent survey *Memory in the Age of AI Agents* [23] organises the field
along three axes — **Forms** (token-level, parametric, latent memory),
**Functions** (factual, experiential, working), and **Dynamics** (how memory
is formed, evolved, retrieved). A complementary taxonomy from the earlier
survey by Zhang et al. [18] groups memory modules by *source* (environmental
vs. agent-generated), *form* (textual vs. parametric), and *operation*
(reading / writing / management).

Mapping these axes onto concrete architectures gives four families:

| Family | Examples | Primary storage | Update model |
|---|---|---|---|
| Hierarchical context | MemGPT, Letta [1, 24] | KV + vector | Tool-driven mutable |
| Memory stream | Generative Agents [7], early MemGPT | Append-only log + vector | Append-only + reflection |
| Graph / knowledge graph | HippoRAG, GraphRAG, LightRAG, PathRAG, Cognee [6, 11, 12, 13, 25] | Graph DB + vector | Incremental graph update |
| Temporal / bitemporal graph | Zep/Graphiti [3, 20] | Temporal KG + vector | Append-only edges with validity intervals |
| Agentic Zettelkasten | A-MEM [2] | Vector + learned links | Self-organising, link-rewriting |
| Service-oriented memory | MaaS [22], Collaborative Memory [17] | Any of the above, behind a service API | Governed, permissioned |

The trajectory over the last three years has been from flat vector stores
toward structured, relationally-aware, temporally-aware memory — and, most
recently, toward *governed* memory that treats read/write access as a
first-class concern.

---

## 3. Survey

### 3A. LLM Long-Term Memory Systems

**MemGPT / Letta** [1, 24]. Packer et al. (2023) propose *virtual context
management*: the LLM's context window is RAM, with explicit "main memory"
(persona, scratchpad, recent events) and "archival memory" (vector-indexed
overflow). The agent issues tool calls to page data between tiers. Letta
[24] productionises this as a stateful-agent platform with REST-accessible
core, archival, and recall memory blocks, and introduces *shared memory
blocks* that multiple agents can read and write — the closest existing
open-source analogue to the shared memory this thesis proposes.

**A-MEM** [2]. Xu et al. (2025) borrow from Luhmann's Zettelkasten: each
memory is a self-describing "note" with generated keywords, tags, and
contextual description; new notes are linked to existing notes through
semantic similarity and LLM-guided relation discovery, and the system
rewrites older notes in light of new ones. Strength: memory structure
*emerges*. Weakness: link rewriting is expensive and hard to make
consistent across concurrent writers.

**Zep / Graphiti** [3, 25]. Rasmussen et al. (2025) build a
**bi-temporally annotated knowledge graph** of entities, relations, and
*episodic* (raw ingested), *semantic* (extracted fact), and *community*
(summary) subgraphs. Every edge carries both (t_valid, t_invalid) in
event-time and an ingestion timestamp in system-time. When a new fact
contradicts an older one, the older edge is *invalidated* (t_invalid set)
rather than deleted — enabling retrospective queries ("what did we believe
on day X?"). On LongMemEval [8] Zep reports up to 18.5% accuracy improvement
over full-context baselines while cutting context tokens from ~115K to
~1.6K and latency from ~30s to ~3s [25].

**Mem0 / Mem0g** [4]. Mem0 (2025) extracts salient facts from conversations
with a dedicated extract-consolidate loop, then stores them; Mem0g adds a
graph layer. Reports 26% LLM-as-judge improvement and ~91% latency reduction
over OpenAI's memory feature on LoCoMo. Weakness observed in follow-up
work: fact extraction discards dense content, hurting retrieval on
information-dense benchmarks like RULER / ∞-Bench.

**Cognee** [5]. Open-source "extract–cognify–load" engine producing a
semantic graph plus vector index; exposes a four-verb API (remember, recall,
forget, improve). Positioned as infrastructure rather than a research
contribution, but useful reference implementation for hybrid storage.

**HippoRAG** [6]. Gutierrez et al. (NeurIPS 2024) frame retrieval as
hippocampal indexing: LLM extracts triples → knowledge graph; at query
time, Personalised PageRank over the graph acts as pattern completion,
surfacing multi-hop neighbours of query entities. Reports up to 20%
improvement on multi-hop QA over iterative retrieval, at 10–30x lower cost.

**Generative Agents** [7]. Park et al. (2023) introduce the canonical
*memory stream*: an append-only log of natural-language observations,
retrieved by a weighted sum of recency, importance, and relevance;
periodic *reflection* produces higher-level summaries that are themselves
appended to the stream. This is the ancestral architecture for most agent
memory work and remains a strong baseline.

**Benchmarks.** LoCoMo [8] — multi-session conversations (~300 turns,
~9K tokens) with QA, summarisation, multi-modal subtasks. LongMemEval [14]
— 500 curated questions over scalable chat histories, targeting five
abilities (extraction, multi-session reasoning, temporal reasoning,
knowledge updates, abstention); reports ~30% accuracy drop on commercial
chat assistants across sustained interactions. MemoryBench [21] —
20,000 cases across 11 sub-benchmarks with two continual-learning
evaluation protocols (off-policy / on-policy). LV-Eval — long-context QA
with confusing-fact insertion and keyword replacement, showing recall
collapses at long context even in strong models.

### 3B. Storage Backends and Trade-offs

**Vector DBs** [19]. Pinecone (managed, serverless), Weaviate
(schema-rich, hybrid keyword+vector), Milvus (open-source, scales to
billions of vectors), Qdrant (real-time, open-source), pgvector
(Postgres extension). Benchmarks show 10–100 ms query times at 1–10M
vectors; Milvus leads at extreme scale, pgvector is simplest when
structured data already lives in Postgres. **Strength**: semantic
similarity retrieval. **Weakness**: poor at relational / multi-hop
queries, no built-in temporal model.

**Graph DBs** [26]. Neo4j (mature Cypher ecosystem, dominant in GraphRAG
tooling), Amazon Neptune (managed, supports property and RDF models),
TigerGraph (MPP architecture, sub-second deep traversal, increasingly
hybrid vector support via TigerVector). **Strength**: multi-hop
relational reasoning, explicit provenance. **Weakness**: embedding-based
similarity is a bolted-on feature; write throughput is typically lower
than vector DBs; schema evolution is non-trivial.

**Relational / KV.** Postgres (with pgvector), Redis (for hot / recency
tiers), SQLite (embeddable prototypes). Useful for structured facts
(user profiles, entity attributes) and for the **source-of-truth event
log** — an append-only table with (valid_time, system_time) columns is a
well-understood pattern [9, 10].

**Hybrid.** Microsoft's **GraphRAG** [11] constructs an entity graph,
clusters with Leiden community detection, and pre-generates
community-level summaries; global queries are answered by
map-reducing over community summaries. **LightRAG** [12] pairs a
graph with a dual-level retrieval (low-level entity + high-level topic)
and supports incremental update without rebuild. **PathRAG** [13]
extracts *relational paths* rather than nodes/subgraphs and prunes with
flow-based algorithms, reducing prompt redundancy. All three are
graph+vector hybrids; the distinction is *where* the graph structure
is used (indexing, retrieval, or prompting).

### 3C. Update Semantics and Consistency

**Append-only event log.** Event sourcing [10] captures state as an
ordered, immutable sequence of domain events; current state is a fold
over the log. This is the operational practice used in systems such as
Kafka and EventStoreDB, and maps naturally onto CQRS [15]: the log is
the write model, derived vector / graph / relational stores are the
read model. Axoniq [15] explicitly positions event-sourced stores as
"agent-ready" because every write has a timestamp, a payload, and a
provenance record.

**Bitemporal modelling** [9, 20]. Fowler's *Bitemporal History* separates
**valid time** ("when was this true in the world?") from **transaction
time** ("when did the system learn it?"). Corrections then become
*new* events ("as of transaction time T2, we now know that from valid
time V1 the fact was X") rather than destructive edits. Graphiti [20]
is the most cleanly bitemporal agent-memory system published to date:
every edge has (t_valid_from, t_valid_to, t_ingested), and supersession
leaves the old edge in place with t_valid_to set.

**Mutable vs versioned.** MemGPT [1] and Letta [24] use mutable core
memory blocks (because they live in the context window and must be
compact) and append-only archival (because history matters). A-MEM [2]
mutates links on existing notes — faster queries but harder to audit
and harder to make concurrency-safe. Versioned snapshots (e.g., Datomic-
style) are rare in published LLM-memory systems but a plausible
implementation substrate for the bitemporal model.

**Conflict resolution.** Classical CRDT theory [16] handles concurrent
updates in distributed data structures without coordination. Graphs
are harder: merge drivers must treat structural edits (add edge,
remove edge) order-independently, and semantic conflicts (two agents
assert contradictory facts) need application-level resolution —
typically last-write-wins on edges with explicit supersession, or
provenance-aware ranking (trust scores). Most production LLM-memory
systems today use last-write-wins plus provenance logging; none of the
surveyed systems implements formal CRDT semantics.

**Consistency guarantees needed by agents.** Agents typically need
**read-your-writes** within a single agent's actions (so they see the
effect of their own memory writes), and **eventual consistency** across
agents (so a write by one agent becomes visible to others within some
bound). Strong / linearizable consistency is usually unnecessary and
expensive; most systems accept bounded staleness. The exception is
shared-state coordination (e.g., two agents must not both claim the
same ticket) — these are better handled via a lock or coordination
primitive on top of the memory, not by the memory's own consistency
model.

### 3D. Multi-Agent Shared vs. Private Memory

Until mid-2025 the literature was almost entirely single-agent / single-
user. Three recent contributions have opened this space.

**Collaborative Memory** [17]. Rezazadeh et al. (May 2025) propose
two-tier memory (private per-user, shared selectively) governed by a
**bipartite access graph** linking users, agents, and resources. Every
fragment carries immutable provenance (contributing agents, resources,
timestamps), enabling retrospective permission checks. This is the
clearest articulation of shared-vs-private memory yet published and
directly addresses the thesis's setting.

**Memory as a Service (MaaS)** [22]. Zhang et al. (June 2025,
position paper) argue that "bound memory" — memory attached to a
specific agent — creates silos; instead, memory should be an
**independently-callable service** with access and composition
governed at the service boundary. This reframes RQ5: the shared memory
is a service that agents call, not a shared variable that agents read.

**Multi-Agent Blackboard** [27]. Blackboard architectures — a 1970s
pattern from speech understanding — have been revisited for LLM
multi-agent systems (e.g., Xu et al. 2025). A control unit watches the
blackboard; agents autonomously decide whether to contribute. The
blackboard *is* the shared memory; the memory model is append-only
message-passing with optional summarisation. Conceptually simpler than
a knowledge graph, but weaker at relational querying.

**Privacy / PII.** The surveyed systems mostly treat PII handling as
out of scope (same as the thesis's current explicit non-goals), but
Collaborative Memory's provenance-per-fragment design is a good starting
point for retrospective redaction and consent revocation.

### 3E. Forgetting and Compaction

**Hierarchical eviction (MemGPT) [1].** When context fills, the agent
uses a *save-to-archival* tool to move blocks out; recall tools fetch
them back when needed. Eviction policy is LLM-driven.

**Reflection (Generative Agents, MemGPT) [7, 1].** Periodically, the
agent summarises sets of memories into higher-level "reflections" or
"insights" that themselves become retrievable memories. Summaries-of-
summaries create a coarse-to-fine hierarchy.

**Reflective Memory Management (RMM)** constructs memory at adaptive
granularities (utterance / turn / session / topic) and uses online RL
feedback to rerank memory relevance.

**Explicit salience / forgetting curves.** Systems like SAGE and MARK
apply Ebbinghaus-style decay and persistence scores to age out
low-importance memories. MIRIX partitions memory into six typed sub-
stores (core, episodic, semantic, procedural, resource, knowledge-vault)
each managed by a dedicated sub-agent.

**Compaction trade-off** [14]. Summarisation *loses information* —
LongMemEval shows commercial assistants lose ~30% accuracy over long
histories. Aggressive compaction also masks failure signals, trapping
agents in bad trajectories. Compaction must be evaluated empirically,
not assumed.

### 3F. Benchmarks and Evaluation

| Benchmark | Year | Focus | Signal |
|---|---|---|---|
| DMR (MemGPT) | 2023 | Multi-session QA | MemGPT 93.4%, Zep 94.8% |
| LoCoMo [8] | 2024 | Very long conversations (~300 turns) | QA + summarisation + multimodal |
| LongMemEval [14] | 2024 | 5 memory abilities, scalable histories | 500 curated questions |
| MemoryBench [21] | 2025 | Memory + continual learning | 20K cases across 11 sub-benchmarks |
| LV-Eval | 2024 | Long-context with distractors | QA up to 256K tokens |

Metrics fall in three buckets: **task success** (QA accuracy, summarisation
quality), **retrieval quality** (recall@k, MRR over ground-truth memory
ids), and **system cost** (tokens per query, p95 latency). The field is
moving toward cost-adjusted metrics (e.g., accuracy / context-tokens)
because full-context baselines can "win" on accuracy while being
economically infeasible.

---

## 4. Evaluation Dimensions

For comparing shared-memory designs in the thesis prototype, the
following dimensions matter:

1. **Retrieval quality** — recall@k, MRR, and task-level accuracy on
   LoCoMo / LongMemEval-style benchmarks adapted to organisational
   events rather than chat.
2. **Update throughput** — sustained writes/sec the memory can accept
   from the batch path; batch-path backpressure tolerance.
3. **Query latency** — p50, p95, p99 for agent read queries (both
   semantic similarity and graph traversal).
4. **Consistency** — read-your-writes, bounded staleness bound, and
   behaviour under concurrent writes from multiple batch workers.
5. **Multi-agent scalability** — effect on latency and quality as the
   number of agents reading concurrently grows.
6. **Forgetting capability** — can the memory shrink? How is
   compaction quality measured (regression tests against a held-out
   set)?
7. **Auditability / provenance** — given any fact in memory, can the
   source event(s) be recovered? Can queries be answered "as of" a
   past time? (The bitemporal model makes this free.)
8. **Operational cost** — storage, LLM calls during ingestion (for
   entity / relation extraction), index maintenance.

---

## 5. Applicability to the Thesis

The thesis's topology is

```
Sources → Processing Engine (fast + batch) → Context Engine ↔ Agents
```

Shared memory is a component of the Context Engine, fed by the **batch
path**. Mapping onto the survey:

- The batch path already produces aggregated, de-duplicated, possibly
  LLM-summarised units (RQ3). Each unit arrives as an event with
  `event_time` (when the underlying business event occurred) and
  `ingest_time` (when the batch job emitted it). This is **natively
  bitemporal** — the system gets valid-time and transaction-time for
  free from the processing pipeline.
- Multiple agents read (§3.4, push+pull hybrid). The Collaborative
  Memory [17] two-tier model (shared + private) maps cleanly: the
  batch path writes to *shared* memory; each agent may maintain its
  own *private* memory (scratchpad, per-conversation state) that is
  never written back to the shared store unless the agent explicitly
  promotes a fact.
- The system requires durability and replay (§4 NFRs). Append-only
  event-sourced storage [10, 15] is the natural fit: the shared memory
  is a set of **read-side projections** (vector index, graph) over a
  durable event log that is itself the source of truth. This matches
  the CQRS split and composes with Lambda (the batch path produces
  write-side events; the projections are materialised views).
- Source extensibility (§4 NFR): because writes to shared memory go
  through an event schema rather than direct DB writes, adding a new
  source only requires an adapter that emits events in that schema.

The awkward case: entity resolution across sources. A "ticket" in JIRA
and a "complaint" in Slack may refer to the same real-world situation.
This is a **graph-construction problem** that vector stores alone cannot
solve — argues strongly for a graph component with LLM-driven entity
resolution (Graphiti-style [3, 20] or Cognee-style [5]).

---

## 6. Proposed Recommendation (Concrete)

**Storage stack.**

- **Source of truth**: an append-only event log (Kafka topic or equivalent,
  already present for Lambda/Kappa reasons per RQ1). Each event carries
  `event_time`, `ingest_time`, source id, payload, and a provenance chain
  pointing back to the raw source events.
- **Derived graph index**: a Graphiti-style temporal knowledge graph
  (Neo4j, Neptune, or a Postgres/AGE hybrid for prototype simplicity),
  with entities / relations extracted by an LLM ingestion step, and
  edges carrying `(t_valid_from, t_valid_to, t_ingested)`. Community
  summaries (GraphRAG-style [11]) are pre-generated for coarse queries.
- **Derived vector index**: embeddings of (a) raw event chunks, (b)
  entity summaries, (c) community summaries. pgvector is sufficient
  for prototype scale; Qdrant or Milvus are obvious production
  upgrades if throughput demands it.
- **Relational sidecar**: a Postgres table for structured facts where
  the schema is well-known (user profiles, ticket metadata). Kept
  in sync via the event log, not written directly.

**Update semantics.**

- **Append-only** at the log layer. No event is ever deleted; corrections
  are new events.
- **Monotonic with supersession** at the graph layer — new edges can
  invalidate older edges by setting `t_valid_to`, but the older edge
  remains retrievable.
- **Eventually consistent** projections — the graph and vector indexes
  lag the log by a bounded latency (SLO: p99 < a few seconds for the
  prototype). Agents can query "as of now" (latest projection state) or
  "as of T" (bitemporal query against the log).

**Multi-agent access.**

- Two-tier model per Collaborative Memory [17]: `shared` (written by
  the batch path, readable by all agents) and `private:agent_X`
  (writable only by agent X). Access is enforced at the Context Engine
  service boundary — MaaS-style [22].
- Every memory fragment carries provenance (source event id, batch job
  id, timestamps, contributing agent id if any). This makes
  retrospective audit and PII redaction tractable.
- Agents that want to promote a private memory to shared do so via an
  explicit "publish" operation that emits an event to the log — the
  same write path as the batch pipeline, so there is exactly one path
  into the shared graph.

**Consistency guarantees delivered.**

- **Read-your-writes** for private memory (single-writer → trivial).
- **Monotonic reads** at shared memory (projection state only moves
  forward).
- **Bounded staleness** across agents (SLO target; measured).
- **No multi-agent transactions**. If two agents need to coordinate on
  a single fact, they use an explicit coordination primitive (lock,
  queue), not the memory's own consistency model.

**Forgetting / compaction.**

- The log is never forgotten (storage is cheap; auditability is
  non-negotiable).
- Projections *can* forget — periodic compaction rewrites clusters of
  old edges into a single summary edge (reflection-style [7]), subject
  to regression tests against a held-out benchmark.
- Low-salience facts decay in retrieval score (recency/importance
  weighting [7]) rather than being deleted.

This design aligns with the strongest empirical results in the survey
(Zep/Graphiti on LongMemEval, GraphRAG on sensemaking), with the
strongest multi-agent framework (Collaborative Memory), and with the
existing architectural commitments of the thesis (durable log, Lambda
split, source extensibility).

---

## 7. Open Questions

1. **Multi-agent consistency remains under-specified.** No surveyed
   system formally characterises the consistency model offered to
   concurrent agent readers/writers. This is a concrete thesis
   contribution opportunity — define and measure the consistency model
   that the proposed memory actually delivers.

2. **Concurrent writes from multiple batch workers.** If the batch
   path is horizontally scaled, two workers may derive contradictory
   facts from overlapping windows. Last-write-wins is the operational
   default; whether CRDT-style merge on structured facts is worth the
   complexity is an open experimental question.

3. **Semantic supersession.** When is a new fact a *correction* of an
   old fact (supersedes) versus *additional information* (coexists)?
   Graphiti uses LLM adjudication; the failure modes of this
   adjudication under high event rates are not well studied.

4. **Compaction quality metrics.** Summary-of-summaries can lose
   information that becomes relevant only later. How is a compaction
   policy evaluated without an omniscient oracle?

5. **Multi-tenant / cross-org shared memory.** Explicitly out of scope
   for the thesis, but note that the Collaborative Memory [17] access
   model is the natural extension point when it is re-added.

6. **Push vs pull interaction with shared memory.** Agents that
   *subscribe* to memory deltas (CDC over the log) versus agents that
   *poll* (project-and-query). Couples with RQ4.

7. **Cost ceiling on LLM-driven ingestion.** Entity / relation
   extraction on every incoming batch unit is expensive. Adaptive
   extraction (extract-then-validate vs skip-if-low-salience) is
   unexplored.

---

## 8. References

[1] Charles Packer, Sarah Wooders, Kevin Lin, Vivian Fang, Shishir G.
Patil, Ion Stoica, Joseph E. Gonzalez. **MemGPT: Towards LLMs as
Operating Systems.** arXiv:2310.08560, 2023.
https://arxiv.org/abs/2310.08560

[2] Wujiang Xu, Zujie Liang, Kai Mei, Hang Gao, Juntao Tan, Yongfeng
Zhang. **A-MEM: Agentic Memory for LLM Agents.** arXiv:2502.12110,
2025. https://arxiv.org/abs/2502.12110

[3] Preston Rasmussen, Pavlo Paliychuk, Travis Beauvais, Jack Ryan,
Daniel Chalef. **Zep: A Temporal Knowledge Graph Architecture for
Agent Memory.** arXiv:2501.13956, 2025.
https://arxiv.org/abs/2501.13956

[4] **Mem0: Building Production-Ready AI Agents with Scalable Long-Term
Memory.** arXiv:2504.19413, 2025.
https://arxiv.org/abs/2504.19413

[5] **Cognee — Knowledge engine for AI agent memory.**
https://github.com/topoteretes/cognee

[6] Bernal Jiménez Gutierrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga,
Yu Su. **HippoRAG: Neurobiologically Inspired Long-Term Memory for
Large Language Models.** NeurIPS 2024, arXiv:2405.14831.
https://arxiv.org/abs/2405.14831

[7] Joon Sung Park, Joseph C. O'Brien, Carrie J. Cai, Meredith Ringel
Morris, Percy Liang, Michael S. Bernstein. **Generative Agents:
Interactive Simulacra of Human Behavior.** UIST 2023, arXiv:2304.03442.
https://arxiv.org/abs/2304.03442

[8] Adyasha Maharana, Dong-Ho Lee, Sergey Tulyakov, Mohit Bansal,
Francesco Barbieri, Yuwei Fang. **Evaluating Very Long-Term
Conversational Memory of LLM Agents (LoCoMo).** ACL 2024,
arXiv:2402.17753. https://arxiv.org/abs/2402.17753

[9] Martin Fowler. **Bitemporal History.** 2021.
https://martinfowler.com/articles/bitemporal-history.html

[10] Martin Fowler. **Event Sourcing.** 2005.
https://martinfowler.com/eaaDev/EventSourcing.html

[11] Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex
Chao, Apurva Mody, Steven Truitt, Jonathan Larson. **From Local to
Global: A Graph RAG Approach to Query-Focused Summarization.**
arXiv:2404.16130, 2024. https://arxiv.org/abs/2404.16130

[12] **LightRAG: Simple and Fast Retrieval-Augmented Generation.**
arXiv:2410.05779, 2024. https://arxiv.org/abs/2410.05779

[13] Boyu Chen et al. **PathRAG: Pruning Graph-based Retrieval
Augmented Generation with Relational Paths.** arXiv:2502.14902, 2025.
https://arxiv.org/abs/2502.14902

[14] Di Wu, Hongwei Wang, Wenhao Yu, Yuwei Zhang, Kai-Wei Chang, Dong
Yu. **LongMemEval: Benchmarking Chat Assistants on Long-Term
Interactive Memory.** ICLR 2025, arXiv:2410.10813.
https://arxiv.org/abs/2410.10813

[15] **Axoniq — Event sourcing / CQRS for agent-ready systems.**
https://www.axoniq.io/

[16] Marc Shapiro, Nuno Preguiça, Carlos Baquero, Marek Zawirski.
**Conflict-free Replicated Data Types.** INRIA Research Report, 2011.
https://inria.hal.science/inria-00609399v1/document

[17] Alireza Rezazadeh, Zichao Li, Ange Lou, Yuying Zhao, Wei Wei,
Yujia Bao. **Collaborative Memory: Multi-User Memory Sharing in LLM
Agents with Dynamic Access Control.** arXiv:2505.18279, 2025.
https://arxiv.org/abs/2505.18279

[18] Zeyu Zhang et al. **A Survey on the Memory Mechanism of Large
Language Model-based Agents.** ACM TOIS, arXiv:2404.13501, 2024.
https://arxiv.org/abs/2404.13501

[19] **Vector Database Comparison: Pinecone vs Weaviate vs Qdrant vs
Milvus vs FAISS.** LiquidMetal AI, 2025.
https://liquidmetal.ai/casesAndBlogs/vector-comparison/

[20] **Graphiti — Build Real-Time Knowledge Graphs for AI Agents.**
getzep. https://github.com/getzep/graphiti

[21] **MemoryBench: A Benchmark for Memory and Continual Learning in
LLM Systems.** arXiv:2510.17281, 2025.
https://arxiv.org/abs/2510.17281

[22] **Memory as a Service (MaaS): Rethinking Contextual Memory as
Service-Oriented Modules for Collaborative Agents.** arXiv:2506.22815,
2025. https://arxiv.org/abs/2506.22815

[23] **Memory in the Age of AI Agents.** arXiv:2512.13564, 2025.
https://arxiv.org/abs/2512.13564

[24] **Letta — Stateful agent platform (production MemGPT).**
https://docs.letta.com/concepts/letta/

[25] **Graphiti: Knowledge Graph Memory for an Agentic World.** Neo4j
Developer Blog, 2025.
https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/

[26] **AI Knowledge Graph Platforms: Neo4j, TigerGraph, AWS Neptune.**
SkillUp, 2025.
https://skillup.ccccloud.com/2025/07/01/ai-knowledge-graph-platforms-comparing-neo4j-tigergraph-and-aws-neptune-for-scalable-ai-applications/

[27] **Exploring Advanced LLM Multi-Agent Systems Based on Blackboard
Architecture.** arXiv:2507.01701, 2025.
https://arxiv.org/abs/2507.01701
