# RQ3 — Batch Aggregation for the Context Engine

**Research question:** *How should events be aggregated on the batch path so that what reaches the Context Engine / agent memory is the most useful unit of context? What does "optimal" even mean here?*

---

## 1. Executive Summary

The batch path should not deliver *raw events* nor *flat windowed aggregates* (both of which map poorly onto LLM attention behaviour). It should deliver **sessionized, entity-centric rollups that are hierarchically summarised and vector-indexed**, emitted on event-time session windows with bounded lateness. Concretely:

1. **Group** events first by *conceptual entity* (Slack thread, JIRA ticket, customer session), not by wall-clock window. The dataflow primitive for this is the **session window** with an inactivity gap [1][2][3].
2. **Compact** each entity-rollup using **hierarchical / map-reduce summarisation** so the unit fits within a dense token budget and avoids "lost-in-the-middle" and "context rot" degradation [4][5][6][7].
3. **Attach** structured metadata (entity IDs, temporal validity, source, participants, sketch-based aggregates) alongside the summary — inspired by Zep's Graphiti and feature-store aggregation patterns [8][9][10].
4. **Index** the unit in a vector + keyword + graph store so retrieval — not dumping — feeds the agent's context window at inference time [11][12][8].
5. **Optimise** for an explicit, measurable objective: *maximise task-weighted information retention per token*, subject to freshness and compute budgets (Section 2).

An "optimal context unit" is therefore not a time-bucketed blob but a **self-contained narrative atom about a bounded entity**, engineered to be the smallest useful slice the agent could retrieve.

---

## 2. Defining the "Optimal Context Unit"

I propose the following operational definition, intended to be testable in the thesis evaluation.

> A *Context Unit* `C` is optimal for an agent task distribution `T` if it maximises
>
>   `U(C) = E_{t ∈ T} [ Performance(agent | C, t) ] − λ · tokens(C) − μ · staleness(C)`
>
> subject to `C` being *self-contained* (interpretable without adjacent units) and *temporally coherent* (all facts inside `C` hold over a single, declared validity interval).

Measurable sub-criteria:

| Dimension | Measure | Notes |
|---|---|---|
| Token efficiency | `tokens(C)` | Direct API cost driver [13][14]. |
| Information retention | Recall of "atomic facts" vs source events, LLM-as-judge faithfulness score | Faithfulness evaluators described in [15]. |
| Task utility | Downstream agent accuracy on LongMemEval-style probes [16] | Avoids measuring summaries intrinsically. |
| Freshness | `now − max(event_time in C)` | Bounded by watermark + allowed lateness [1][2]. |
| Ordering correctness | % of causally-ordered event pairs preserved | Critical for "what happened before X?" queries. |
| Density | Useful-fact / token ratio (LLM-judge or entailment) | Mirrors LLMLingua's density framing [13]. |

The two weights `λ, μ` are not magic numbers but **product choices**: a cost-sensitive agent sets `λ` high, a real-time incident-response agent sets `μ` high.

**Why not just "more context"?** Liu et al. show U-shaped performance with position (lost-in-the-middle) [4]; Chroma's "Context Rot" study shows non-uniform degradation with input length across GPT-4.1, Claude 4, Gemini 2.5, Qwen3 even on semantically trivial tasks [5]. More tokens is not monotonically better, so the batch path must *actively compress*.

---

## 3. Survey

### A. Classical stream-aggregation techniques

Akidau et al.'s Dataflow Model [1] is the canonical decomposition: *what* is being computed (the aggregation), *where* in event time (windows), *when* in processing time (triggers + watermarks), and *how* refinements relate (accumulation mode). Three windowing primitives are relevant here:

- **Tumbling windows** — fixed, non-overlapping. Good for "events per minute" telemetry rollups but a poor fit for conversational entities because threads cross boundaries.
- **Sliding windows** — fixed size, overlapping stride. Useful for moving aggregates (e.g., "error rate over last 5 min, every 30 s") but produce redundant copies.
- **Session windows** — gap-based, data-driven [2][3]. Windows open on activity, close after an inactivity gap. This is the natural fit for Slack threads, customer browsing sessions, and JIRA comment bursts.

**Watermarks** [17] give the system a bounded belief that "all events ≤ t have arrived", allowing emission without waiting forever. Flink and Beam both support *allowed lateness* that keeps a window's state alive past the watermark so late events can retrigger the summary [3][18]. For our use case, late Slack edits or out-of-order webhook deliveries are the norm — the pipeline must tolerate it.

**Approximate sketches** provide cheap, mergeable aggregates alongside the textual rollup:

- **HyperLogLog** for distinct counts (unique users in a session) with sub-linear memory [19].
- **Count-Min Sketch** for top-K heavy-hitter frequencies (most-mentioned components in a Slack channel) [20].
- **T-Digest** for quantile estimates of latencies or order values [21].

These don't replace the summary — they're structured *side-channels* the agent (or a tool call) can query without reading raw events.

### B. Deduplication & noise reduction

Exactly-once semantics via Kafka idempotent producers and Flink's two-phase commit sinks [22][23] handle *system-level* duplicates. What matters more for context quality is **semantic deduplication**:

- **MinHash + LSH** on shingled text for near-duplicate messages and forwarded notifications — the classical approach, scalable to trillions of docs [24][25].
- **SimHash** for compact fingerprints of similar documents [24].
- **Embedding-similarity thresholds** (e.g., cosine > 0.92) for semantic dedup — expensive but catches rephrased duplicates (tools like SemHash) [24].

A practical pipeline uses MinHash-LSH as a cheap first pass, then embedding similarity inside candidate buckets — this is what training-data dedup pipelines do at scale [24][25].

### C. Summarization of event streams for LLMs

The literature separates into three families:

1. **Map-reduce / refine / stuff chains** [26]. *Stuff* concatenates; *map_reduce* summarises each chunk then summarises the summaries; *refine* rolls a running summary forward chunk by chunk. For streaming, *refine* maps naturally onto incremental updates but suffers drift; *map_reduce* is parallelisable but loses cross-chunk structure.

2. **Hierarchical / recursive summarisation**. Wu et al.'s recursive book summarisation [6] is the seminal LLM application: split → summarise → summarise summaries, up a tree. Recent work (Context-Aware Hierarchical Merging [27], dynamic tree construction [28]) enriches merges with source context to reduce hallucination amplification — a real risk when each merge is lossy.

3. **Incremental / streaming summarisation**. Chain-of-Key [29] updates a structured representation rather than regenerating; SliSum [30] uses overlapping windows + self-consistency voting across local summaries; summarisation-based context management in long RL episodes periodically compacts tool-call histories [31]. The "Complexity Trap" paper [32] is a useful counterpoint: on some agent benchmarks, *observation masking* is as good as LLM summarisation — suggesting summarisation is only worth its cost when it actually produces denser meaning.

### D. Entity / thread / topic rollups

Graph-based memory has become the dominant frame for agent memory at scale:

- **Zep / Graphiti** [8] stores agent memory as a temporal knowledge graph where each edge carries `valid_from`, `valid_to`, `invalid_at`. This supports queries like "what did we believe about this customer before the escalation?" — impossible with a flat vector store.
- **Mem0** [33][34] extracts, consolidates, and retrieves salient facts; vector-first with optional graph.
- **Hindsight** and similar systems [11] split memory into logical networks (world facts, agent experiences, entity summaries, evolving beliefs) with temporal / semantic / entity / causal edges.
- **Sessionization** primitives from the analytics world — Vertica's event-based windows, Kinesis Data Analytics clickstream pipelines — define the canonical "group by user, gap-split by inactivity" pattern [35][36] that maps cleanly onto Flink session windows.

The pattern is consistent: **group by entity first, aggregate within entity, then link across entities in a graph**.

### E. "Optimality" criteria for LLM context

Key results bounding what "optimal" can mean:

- **Lost in the Middle** [4]: U-shaped recall across context position; putting relevant info at the ends dominates putting it in the middle. Implication: a well-ordered, short summary beats a long dump.
- **Context Rot** (Chroma) [5]: even trivially easy retrieval tasks degrade non-uniformly as context grows, across frontier models. Implication: cost-free long context is a myth.
- **LLMLingua / LongLLMLingua** [13][37]: 4–20× prompt compression with minimal quality loss (sometimes *improving* quality on long-context QA by +21.4%). Implication: there is real slack in how densely we package context.
- **LongMemEval** [16]: long-context LLMs drop 30–60% on realistic long-term memory tasks; commercial systems hit only 30–70% in even the easier setting. Implication: simply extending context window ≠ solving memory.
- **LLM-as-judge faithfulness** [15]: ~80% agreement with humans on evaluating summary faithfulness, making automated evaluation tractable (though with known bias pitfalls).

Taken together: "optimal" = **task-conditioned, compressed, well-ordered, temporally-consistent** — not "maximal recall of raw events".

### F. Systems work

- **Feature stores (Tecton, Feast)** [9][10] normalise the pattern of "rolling time-window aggregation, consistent online/offline". Tecton's built-in stream feature views compute `count_over_last_30m` per entity key and serve the same value to training and inference. Direct analogue: our batch path should produce *per-entity aggregates* that can be served at inference via point-in-time lookup. The feature store is a design ancestor of what we're building; it just hadn't met LLMs yet.
- **MemGPT / Letta** [38][39] use OS-style virtual paging: main context (hot) + external context (archival/recall), with functions to move data between. Periodic self-directed summarisation compacts working memory. The compaction policy is what our batch path is, architecturally.
- **Zep / Graphiti** [8] — temporal knowledge graph; sub-second retrieval; 18.5% accuracy gain and 90% latency reduction over baselines on LongMemEval.
- **Mem0** [33][34] — extract-consolidate-retrieve loop; vector + optional graph; explicit fact-level granularity.
- **StreamingLLM** (attention sinks) [40] — architectural answer at inference time; orthogonal to but compatible with our approach.

---

## 4. Evaluation Dimensions

| Dimension | Metric | Measurement approach |
|---|---|---|
| **Token efficiency** | Tokens per Context Unit, compression ratio vs raw events | Tokeniser count; compare summary to concatenated source events |
| **Information retention** | Atomic-fact recall | Extract facts from source via LLM, check entailment against summary [15] |
| **Task performance** | Accuracy on agent probe set | Construct LongMemEval-style probes per source (Slack / JIRA / e-com) [16] |
| **Latency to freshness** | `p95(emit_time − event_time)` | Instrument Flink pipeline; bounded by watermark + allowed lateness [2] |
| **Ordering correctness** | % causally-ordered event pairs preserved | Inject ordering probes ("A before B") into probe set |
| **Compute cost of aggregation** | $/1M events; LLM calls per unit | Track tokens-in/out per summary and per merge step |
| **Noise level** | Duplicate fact rate; contradiction rate | LLM-judge over Context Units [15] |
| **Graph coverage** | % of probe entities resolvable in the rollup graph | Evaluate against held-out entity set |

Two design flags emerge as particularly important:

- **Ordering correctness is under-measured in the summarisation literature.** Most summarisation benchmarks reward semantic faithfulness, not temporal order. For agents doing *reasoning over sequences* (incident post-mortems, customer-journey analysis) this is critical.
- **Freshness and compression trade off directly.** A session window with 30-min inactivity gap delays emission by ≥30 min; a tumbling 1-min window emits sooner but cuts entities in half.

---

## 5. Applicability to this Thesis

The three heterogeneous sources stipulated by the thesis have *very different natural aggregation grains*:

- **Slack** — the natural unit is a **thread** (or a channel-topic-within-time-window where thread structure is absent). Events: messages, reactions, edits, joins. The correct window is **session**, keyed by `thread_ts` (or channel + user-inactivity-gap where no thread).
- **JIRA** — the natural unit is a **ticket**, with a *timeline* of status changes, comments, links, subtask transitions. Events arrive over days to months. The correct window is essentially **global per key with periodic compaction** — closer to an evolving entity rollup than a classical window.
- **E-commerce telemetry** — the natural unit is a **customer session**, gap-split by 30-min inactivity (web-analytics convention [36]) or by a session cookie. Events: page views, add-to-cart, search, checkout.

Common pattern: **key-partitioned session or entity window → entity-scoped summary → vector-and-graph index**. This is exactly the combination the executive summary proposed: *sessionized entity rollups + hierarchical summarisation + vector/graph indexing*.

The design cleanly separates two concerns the thesis cares about:

- The **batch path** is responsible for *building* high-quality, self-contained Context Units.
- The **Context Engine** is responsible for *retrieving and composing* units at agent query time — applying Lost-in-the-Middle-aware ordering and token budgeting.

Neither layer carries raw events into the agent prompt.

---

## 6. Proposed Recommendation

### 6.1 Generic pattern

For every source, the batch path runs:

```
ingest  →  dedupe (MinHash + embed)
        →  partition by entity_key
        →  event-time session/entity window (with allowed lateness)
        →  enrich (add sketches: HLL users, CMS top-terms, T-Digest latencies)
        →  summarise (map-reduce for large windows, refine for incremental)
        →  validate (LLM-judge faithfulness + fact-extraction check)
        →  emit Context Unit { summary, facts[], entities[], sketches, valid_from, valid_to, source_refs[] }
        →  index (vector + keyword + temporal graph edges)
```

A Context Unit is a **typed record**, not a free-text blob. The summary is one field; structured facts, entity references, temporal validity, and pointers to raw events are equal citizens. This supports three retrieval modes the Context Engine can mix: vector-nearest, graph-traversal, and metadata-filter.

### 6.2 Slack — thread rollup

- **Entity key:** `channel_id + thread_ts` (fallback: `channel_id + sliding(1h)` for loose channels).
- **Window:** session, 30-min inactivity gap, allowed lateness 2h (edits/reactions).
- **Summary strategy:** map-reduce for long threads (>50 messages); refine for streaming updates within a live thread. Preserve speaker attribution — it is load-bearing for agent reasoning ("who decided X?").
- **Structured facts:** decisions made, action items assigned, open questions, linked JIRA/PR IDs (regex extraction + LLM confirmation).
- **Sketches:** HLL distinct participants; CMS top-terms.
- **Graph edges:** `mentions(ticket)`, `references(pr)`, `assigns(user, task)`.

### 6.3 JIRA — ticket timeline

- **Entity key:** `ticket_id`.
- **Window:** not time-based. Use **keyed-state with compaction triggers** — emit a new Context Unit version when any of: status change, N new comments, T elapsed since last emission, or on demand.
- **Summary strategy:** hierarchical. Level 1 = comment-cluster summaries; Level 2 = phase summaries (triage / in-progress / review / done); Level 3 = ticket-level narrative. Preserve a *strictly-ordered* timeline of status transitions — for "when did this regress?" queries.
- **Structured facts:** current status, assignee, priority, linked tickets, fix versions, reporter, SLA clock.
- **Temporal edges** (Graphiti-style [8]): every fact carries `valid_from` / `valid_to`, so we can answer "who owned this last Tuesday?".

### 6.4 E-commerce — customer session

- **Entity key:** `user_id` (logged-in) or `session_cookie`.
- **Window:** session, 30-min inactivity gap [36], allowed lateness 15 min.
- **Summary strategy:** extractive-first. For a session, the LLM-relevant signal is usually "user browsed X, added Y, abandoned at Z" — a template-filled summary is cheaper and more reliable than free generation. Reserve abstractive summarisation for support-relevant sessions (error events, failed checkouts).
- **Structured facts / sketches:** pages viewed (HLL), top-category (CMS), cart value (T-Digest), final funnel state.
- **Graph edges:** `viewed(product)`, `purchased(product)`, `session_of(user)`.

### 6.5 Emission discipline

Every Context Unit carries: `unit_id`, `entity_key`, `source`, `valid_from`, `valid_to`, `produced_at`, `watermark_at`, `version`, `parent_unit_ids[]`, `token_count`, `compression_ratio`. This makes the pipeline auditable and the evaluations in Section 4 directly computable.

---

## 7. Open Questions

1. **Evaluation without ground truth.** There is no gold standard for "optimal summary of this Slack thread". Options: (a) LLM-as-judge on faithfulness [15] — reliable for factuality, weak for task utility; (b) downstream probe tasks (LongMemEval-style [16]) — gold standard but expensive to construct; (c) counterfactual A/B on agent task accuracy with vs without the unit — gold standard but needs a running agent harness. Combining (a) + (b) is probably necessary.

2. **Compression ratio vs task utility curve.** Where does the knee live per source? LongLLMLingua [37] reports improvements at 4× compression; does that transfer to our sources or are threads already information-dense?

3. **Hierarchical merge hallucination.** Each merge can amplify errors [27]. Worth quantifying: fact-drift rate per hierarchy level. Mitigation: context-aware merging that keeps pointers back to source.

4. **When is summarisation *not* worth it?** "Complexity Trap" [32] suggests simple observation masking sometimes matches LLM summarisation. We should test the null hypothesis: on each source, does a structured-fact-extraction-only pipeline (no free-text summary) perform as well on the probe set?

5. **Cross-entity units.** Some agent queries span entities ("all escalations related to service X last week"). Do we compose at query time from per-entity units, or do we materialise cross-cutting units? The former is more flexible; the latter may be faster.

6. **Update semantics for long-lived entities (JIRA).** When a new comment arrives 3 weeks in, do we re-summarise the whole ticket (expensive, consistent) or append-and-refine (cheap, drift-prone)? A hybrid: always append, periodically rebuild.

7. **Privacy / redaction.** Summaries leak PII that raw-event filters might have caught. Dedup and summarisation stages both need to be PII-aware — relevant for GDPR/production deployment but out of scope for the thesis prototype.

---

## 8. References

[1] Akidau, T., Bradshaw, R., Chambers, C., Chernyak, S., Fernández-Moctezuma, R., Lax, R., McVeety, S., Mills, D., Perry, F., Schmidt, E., & Whittle, S. (2015). *The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing*. VLDB 2015. https://www.vldb.org/pvldb/vol8/p1792-Akidau.pdf

[2] Apache Flink — Windows (master docs). https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/windows/

[3] Decodable — Understanding Apache Flink Event Time and Watermarks. https://www.decodable.co/blog/understanding-apache-flink-event-time-and-watermarks

[4] Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2023/2024). *Lost in the Middle: How Language Models Use Long Contexts*. TACL 12. https://arxiv.org/abs/2307.03172

[5] Chroma Research — *Context Rot: How Increasing Input Tokens Impacts LLM Performance* (2025). https://research.trychroma.com/context-rot

[6] Wu, J., Ouyang, L., Ziegler, D. M., Stiennon, N., Lowe, R., Leike, J., & Christiano, P. (2021). *Recursively Summarizing Books with Human Feedback*. arXiv:2109.10862. https://arxiv.org/abs/2109.10862

[7] Akidau, T., Chernyak, S., & Lax, R. (2018). *Streaming Systems: The What, Where, When, and How of Large-Scale Data Processing*. O'Reilly. https://www.oreilly.com/library/view/streaming-systems/9781491983867/

[8] Rasmussen, P. et al. (2025). *Zep: A Temporal Knowledge Graph Architecture for Agent Memory*. arXiv:2501.13956. https://arxiv.org/abs/2501.13956

[9] Tecton — *Real-Time Aggregation Features for Machine Learning (Part 1)*. https://www.tecton.ai/blog/real-time-aggregation-features-for-machine-learning-part-1/

[10] Feast — *What is a Feature Store?* https://feast.dev/blog/what-is-a-feature-store/

[11] *Hindsight is 20/20: Building Agent Memory that Retains, Recalls, and Reflects*. arXiv. https://arxiv.org/html/2512.12818

[12] Weaviate — *Chunking Strategies to Improve LLM RAG Pipeline Performance*. https://weaviate.io/blog/chunking-strategies-for-rag

[13] Jiang, H. et al. (2023). *LLMLingua: Compressing Prompts for Accelerated Inference of Large Language Models*. EMNLP. https://arxiv.org/abs/2310.05736

[14] Redis — *LLM Token Optimization: Cut Costs & Latency in 2026*. https://redis.io/blog/llm-token-optimization-speed-up-apps/

[15] Evidently AI — *LLM-as-a-judge: a complete guide to using LLMs for evaluations*. https://www.evidentlyai.com/llm-guide/llm-as-a-judge

[16] Wu, X. et al. (2024/2025). *LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory*. ICLR 2025. https://arxiv.org/abs/2410.10813

[17] Begoli, E. et al. (2021). *Watermarks in Stream Processing Systems: Semantics and Comparative Analysis of Apache Flink and Google Cloud Dataflow*. PVLDB 14(12). http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf

[18] Apache Beam — *Basics of the Beam model*. https://beam.apache.org/documentation/basics/

[19] Google Cloud — *Cloud Dataflow support for HyperLogLog++ for count-distinct*. https://cloud.google.com/blog/products/data-analytics/using-hll-speed-count-distinct-massive-datasets

[20] Redis — *Count-Min Sketch: The Art and Science of Estimating Stuff*. https://redis.io/blog/count-min-sketch-the-art-and-science-of-estimating-stuff/

[21] QuestDB — *Sketch Algorithm* (T-Digest, HLL, Bloom). https://questdb.com/glossary/sketch-algorithm/

[22] Confluent — *Exactly-once Semantics is Possible: Here's How Apache Kafka Does it*. https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/

[23] Apache Flink blog — *An Overview of End-to-End Exactly-Once Processing in Apache Flink*. https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/

[24] Milvus — *MinHash LSH in Milvus: The Secret Weapon for Fighting Duplicates in LLM Training Data*. https://milvus.io/blog/minhash-lsh-in-milvus-the-secret-weapon-for-fighting-duplicates-in-llm-training-data.md

[25] Zilliz — *Data Deduplication at Trillion Scale*. https://zilliz.com/blog/data-deduplication-at-trillion-scale-solve-the-biggest-bottleneck-of-llm-training

[26] LangChain — `load_summarize_chain` (stuff / map_reduce / refine). https://api.python.langchain.com/en/latest/chains/langchain.chains.summarize.chain.load_summarize_chain.html

[27] *Context-Aware Hierarchical Merging for Long Document Summarization* (2025). arXiv:2502.00977. https://arxiv.org/abs/2502.00977

[28] *Dynamic Tree Construction for Recursive Summarization* (ACL 2025). https://aclanthology.org/2025.acl-long.536.pdf

[29] *Enhancing Incremental Summarization with Structured Representations* (2024). arXiv:2407.15021. https://arxiv.org/html/2407.15021v1

[30] *Improving Faithfulness of LLMs in Summarization via Sliding Generation and Self-Consistency* (SliSum, 2024). arXiv:2407.21443. https://arxiv.org/abs/2407.21443

[31] *Scaling LLM Multi-turn RL with End-to-end Summarization-based Context Management* (2025). arXiv:2510.06727. https://arxiv.org/abs/2510.06727

[32] *The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management* (2025). arXiv:2508.21433. https://arxiv.org/html/2508.21433v1

[33] Mem0 — *Building Production-Ready AI Agents with Scalable Long-Term Memory*. arXiv:2504.19413. https://arxiv.org/pdf/2504.19413

[34] Mem0 GitHub. https://github.com/mem0ai/mem0

[35] Vertica — *Sessionization with event-based windows*. https://docs.vertica.com/25.3.x/en/data-analysis/sql-analytics/sessionization-with-event-based-windows/

[36] AWS — *Create real-time clickstream sessions and run analytics with Kinesis Data Analytics, Glue, and Athena*. https://aws.amazon.com/blogs/big-data/create-real-time-clickstream-sessions-and-run-analytics-with-amazon-kinesis-data-analytics-aws-glue-and-amazon-athena/

[37] Jiang, H. et al. (2023). *LongLLMLingua: Accelerating and Enhancing LLMs in Long Context Scenarios via Prompt Compression*. arXiv:2310.06839. https://arxiv.org/abs/2310.06839

[38] Packer, C. et al. (2023). *MemGPT: Towards LLMs as Operating Systems*. arXiv:2310.08560. https://arxiv.org/abs/2310.08560

[39] Letta Docs — *MemGPT concepts*. https://docs.letta.com/concepts/memgpt/

[40] Xiao, G. et al. (2023/2024). *Efficient Streaming Language Models with Attention Sinks* (StreamingLLM). ICLR 2024. https://github.com/mit-han-lab/streaming-llm
