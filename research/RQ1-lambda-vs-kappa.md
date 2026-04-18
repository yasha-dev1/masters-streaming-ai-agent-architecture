# RQ1 — Lambda vs. Kappa (vs. Hybrid) for a Context Management System Feeding AI Agents

*Research note, problem-framing stage. Date: 2026-04-18.*

## 1. Executive summary

For a context-management substrate that (a) ingests thousands of heterogeneous events per minute, (b) must route a small fraction of latency-sensitive events within hundreds of milliseconds, (c) aggregates the remainder into shared, durable memory for LLM agents, and (d) requires replay in order to rebuild agent memory as schemas and prompts evolve, the literature converges on **a streaming-first architecture with a pragmatic hybrid twist** rather than classical Lambda.

The original Lambda architecture [1][4] has been repeatedly criticised as unnecessarily complex once modern stream engines achieved exactly-once semantics and event-time correctness [2][3][14]. Kappa [2][9] eliminates the dual-codebase problem but pure Kappa has well-documented failure modes for long-horizon reprocessing and for mixing bounded with unbounded data [5][15]. Production-scale operators (Uber [5][6], Netflix [17][18], LinkedIn [7][8], Airbnb [10]) have converged on variants: **Kappa+ (warehouse-as-stream-source)**, **Delta / lakehouse unification**, or **Kappa with a materialised hot tier**.

For this thesis specifically, the fast-path / batch-path distinction in the problem statement is best understood not as two *codebases* (Lambda) but as **two output tiers of a single streaming job graph**: a low-latency push channel for urgent events and a windowed-aggregation channel that writes into the Context Engine's shared memory. That matches Confluent's "Real-Time Context Engine" framing [11][12], Timeplus's context-layer pattern [13], and Kai Waehner's argument that agentic AI specifically demands Kappa-style unification of fresh and historical context [19]. Recommendation: **streaming-first Kappa-style pipeline with a materialised hot tier and a warehouse-backed replay source (Kappa+)**; write logic once, expose two delivery semantics (push and pull), and budget explicitly for a replay path that reads from object-store snapshots rather than unbounded Kafka retention.

## 2. Background: precise definitions

**Lambda architecture.** Coined by Nathan Marz around 2011 and presented in full in *Big Data* (Marz & Warren, Manning, 2015) [4]. Three layers sit behind an immutable master dataset: a **batch layer** recomputing views over all history, a **speed layer** approximating the most recent window in real time, and a **serving layer** merging the two at query time. The canonical justification is fault tolerance via recomputation and the ability to absorb bugs in the speed layer by periodically overwriting its outputs from the batch layer.

**Kappa architecture.** Proposed by Jay Kreps in the 2014 O'Reilly Radar essay "Questioning the Lambda Architecture" [1]. A single stream-processing framework handles both real-time and historical workloads. Reprocessing is done by spinning up a second instance of the streaming job that reads from the beginning of a durable, ordered log (typically Kafka), producing an updated output topic; consumers are then switched over and the old job retired [1][2]. The essential property is that *only one codebase* exists, and history is replayed through it rather than processed by a separate batch system.

**Streaming-supersedes-batch thesis.** Tyler Akidau's "Streaming 101" / "102" articles [3] and the subsequent *Streaming Systems* book [23] argue that a well-designed streaming system is "a strict superset of batch" [3]: with event-time windowing, watermarks [16], triggers, and exactly-once state, a stream engine can emit the same results as a batch engine plus bounded-latency updates. This is the theoretical grounding for Kappa.

**Kappa+ (Uber).** Introduced by Roshan Naik and described in Uber's 2021 engineering blog [5] and the SIGMOD'21 paper "Real-time Data Infrastructure at Uber" [6]. Pure Kappa requires replaying months of Kafka data to reprocess; Kappa+ instead treats the warehouse (Hive/HDFS/Iceberg) as an unbounded streaming source via a custom connector, so the *same* streaming code runs against either Kafka (live) or warehouse partitions (historical). Production numbers reported: a 75-core / 1.2 TB-memory Flink job, 9-day / 10 TB backfills, windows relaxed from 10 s to 2 h during backfill to rate-limit output [5].

**Delta / Lakehouse architecture (Databricks).** Delta Lake [20] turns object storage into an ACID, streamable table; a single Spark Structured Streaming job can both serve real-time queries and reprocess history from the same table. Databricks' "Delta vs Lambda" post [20] is explicitly a Kappa-aligned critique, citing customer migrations that cut Python LOC from 565 → 317 and YAML from 252 → 23 after collapsing Lambda into a Delta pipeline.

**Lambda+ (Gillet et al., CAiSE 2021).** Academic revival of Lambda using category theory to prove composition properties across layers [21]. Of limited direct relevance here but worth noting as evidence that Lambda is still academically defended for systems where batch and speed have genuinely different operators.

**Hybrid variants relevant to this thesis.**
- *Kappa with a materialised hot tier.* Single streaming job writes both to a hot (Redis/Cassandra/in-memory) store for fast retrieval and to a durable log/table for historical access [11][22]. This is the dominant pattern in modern feature stores (Uber Michelangelo Palette [22], Airbnb Chronon [10], Tecton).
- *Streamhouse / Flink + Iceberg / Paimon.* Kappa whose "log" is an open table format, not Kafka — what Ververica calls "Streamhouse" [24]. Addresses Kappa's retention-cost problem by tiering raw events into cheap object storage.
- *Event-sourced context layer.* Timeplus [13] and Confluent [11][12] frame the AI-agent use case as a live materialised view continuously updated by streaming SQL; Confluent calls this the "Real-Time Context Engine" and explicitly positions it as streaming-as-generalisation-of-batch rather than Lambda [11].

## 3. Evaluation dimensions

The axes on which the literature compares these architectures:

1. **Operational complexity** — number of distinct systems, on-call surface, deployment modes.
2. **Code duplication / logic drift** — whether fast and batch code must be maintained separately and kept consistent [1][14][20].
3. **Reprocessing cost and speed** — re-deriving outputs after logic/schema change; scanning columnar files vs replaying an event log [15][20].
4. **Latency floor** — minimum end-to-end delay on the fast path.
5. **Consistency between hot and cold outputs** — can agents trust that the "5-minute ago" and "yesterday" views came from the same logic? [1][14][19]
6. **Durability and replay horizon** — how far back can you rebuild from? What is the cost of that retention? [5][15]
7. **Cost** — storage for long-retention logs; compute for replay; steady-state CPU [15][20].
8. **Debuggability and lineage** — tracing an output back to source events across layers [5][6].
9. **Evolvability of business logic** — cost to change a windowing rule, aggregation, or schema [1][20].
10. **Fit to mixed-latency SLOs** — ability to serve both "push this instantly" and "use this in the next aggregation" semantics [19][11][13].
11. **Fit to AI-agent consumption patterns** — suitability for RAG / context retrieval / MCP-style pulling [11][12][13][26].

## 4. Comparative analysis per dimension

**4.1 Operational complexity.** Lambda runs at minimum a batch engine (historically Hadoop/Spark) plus a stream engine (Storm/Samza/Flink) plus a serving merge layer. Kreps [1] reports that even at LinkedIn, with strong engineering, "keeping code written in two different systems perfectly in sync was really, really hard." Kappa eliminates one engine; Kappa+ adds a warehouse-as-source connector but reuses the stream engine [5]. Databricks' Delta case studies report 4–10× reductions in config and code [20]. For a thesis prototype, Lambda triples the number of moving parts. **Winner: Kappa / Kappa+ / Delta.**

**4.2 Code duplication.** This is the headline Lambda critique. Kreps [1]: "The problem with the Lambda Architecture is that maintaining code that needs to produce the same result in two complex distributed systems is exactly as painful as it seems." Akidau [3] reaches the same conclusion from first principles — if streaming is a superset of batch, the batch layer is architecturally redundant. Chronon [10] and Tecton [10] exist precisely because the feature-store community felt this pain and built declarative abstractions that compile to both batch and stream. Kappa and Delta eliminate duplication by construction. **Winner: Kappa, Delta, or a write-once abstraction (Chronon / Beam).**

**4.3 Reprocessing cost.** This is Kappa's weakest axis. Replaying months of Kafka through a streaming job is slower than scanning Parquet in a batch engine [15]. Columnar formats and vectorised execution advantage batch for bounded, historical scans. Lambda's batch layer handles this naturally; pure Kappa needs long log retention and pays for it. Kappa+ [5] and Delta [20] both resolve this by letting the stream engine read directly from a columnar warehouse format (Hive, Iceberg, Delta), recovering most of the batch advantage without a separate codebase. **Winner: Lambda or Kappa+/Delta; pure Kappa loses.**

**4.4 Latency floor.** All modern architectures can hit sub-second if they use a proper stream engine (Flink, Samza, Kafka Streams). Lambda's serving-layer merge adds latency unless the speed layer is queried directly — which defeats the point. Kappa variants and Delta-style Spark Structured Streaming can both push events through in tens to hundreds of milliseconds [6][17]. **Winner: tie for the streaming engines; Lambda disadvantaged by the merge step.**

**4.5 Consistency between hot and cold.** Lambda's central reconciliation problem: the speed layer is an approximation, the batch layer is ground truth, and their disagreement must be tolerated or papered over by the serving layer [4][14]. For AI agents this is actively harmful: Kai Waehner [19] argues that agentic systems *cannot* tolerate batch/stream drift because an agent's decisions become non-reproducible. Kappa and Delta guarantee by construction that every view descends from one job graph. **Winner: Kappa / Delta.**

**4.6 Durability and replay horizon.** Lambda's master dataset is immutable by design and so is inherently replayable — but only through the batch codebase, which as noted diverges from the speed codebase. Kappa needs Kafka retention as long as its deepest required replay window. Kappa+ and Delta sidestep this by persisting raw events in cheap object storage as the replay source [5][20][24]. Kleppmann's "log as source of truth" framing [9][25] applies uniformly; DDIA chapter 11 [26] is the canonical reference for the underlying semantics. **Winner: Kappa+/Delta/Streamhouse (best cost/horizon trade-off).**

**4.7 Cost.** Long Kafka retention is expensive; tiered storage (Kafka's own tiered storage, or Iceberg-backed logs) materially reduces this [24]. Two-engine Lambda operations cost more in licence/infra and in engineers' time. Delta reports cost reductions in the 50–90 % range for comparable workloads [20], though vendor-authored. **Winner: Kappa+/Delta; pure Kappa middling; Lambda worst.**

**4.8 Debuggability and lineage.** Lambda's two paths make per-event lineage genuinely hard: an output may have come from the batch or the speed layer, and reconciling them backward is manual. Streaming-first systems with exactly-once state and a single job graph have cleaner lineage [6][26]. LinkedIn's Samza / Brooklin stack [7][8] builds lineage on top of Kafka offsets — a single coordinate per event. **Winner: Kappa / Kappa+ / Delta.**

**4.9 Evolvability.** Changing an aggregation in Lambda means changing it in two languages/paradigms and validating that the outputs agree [1][14]. In Kappa you deploy a new job instance, let it catch up from the log, and switch [1][2]. In Delta you update the pipeline and rerun over the Delta table [20]. The declarative abstractions (Chronon [10], Beam [27]) make this even simpler by letting authors express features once. **Winner: Kappa / Delta / declarative-over-unified.**

**4.10 Fit to mixed-latency SLOs.** This is the axis on which the *problem-statement* instinct for Lambda is most defensible — the claim being that fast-path and batch-path are genuinely different. But Flink, Kafka Streams, and Spark Structured Streaming all support this natively *within* a single job graph: one branch emits low-latency per-event outputs (a "hot" sink), another branch applies event-time windows with watermarks [16] and emits aggregated outputs. This is the pattern Uber uses for sessionisation [5] and the one Confluent promotes for Streaming Agents [11][12]. The fast/slow split is a *topology* concern, not an *architecture* concern. **Winner: single streaming pipeline with bifurcated sinks.**

**4.11 Fit to AI-agent consumption.** This is the newest axis and also the most asymmetric. Recent industry work [11][12][13][19] and survey-style pieces on agent memory [28] converge on: agents want a live materialised view of organisational state, exposed via MCP-style pulling, plus push notifications for urgency — *the same bifurcated-sink pattern as §4.10*. Crucially, because agents are non-deterministic consumers, any drift between hot and cold views surfaces as hallucination or incoherent behaviour — a direct argument against Lambda-style reconciliation [19]. Agent-memory research [28] also assumes a unified stream of observations rather than two reconciled layers. **Winner: Kappa-style unified stream with materialised hot/cold tiers.**

## 5. Applicability to this thesis

Mapping the thesis workload onto the dimensions:

| Thesis property                                                            | Dimension                 | Winning architecture                                                      |
| -------------------------------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------- |
| Thousands events/minute, burst-tolerant                                    | §4.4 latency, §4.7 cost   | Kappa / Kappa+                                                            |
| Heterogeneous sources (Slack, JIRA, analytics…)                            | §4.2, §4.9                | Kappa + declarative adapter (Brooklin-like [8], Beam [27])                |
| Fast-path routing for urgent events (~hundreds of ms)                      | §4.4, §4.10               | Single stream with low-latency sink                                       |
| Batch-path aggregation into shared memory (windowed, dedup, summarisation) | §4.3, §4.9                | Single stream with windowed sink                                          |
| Agents need instant + aggregated context consistently                      | §4.5, §4.11               | Kappa / Delta (explicitly rules out Lambda per [19])                      |
| Durability + replay (rebuild shared memory as prompts/schema evolve)       | §4.3, §4.6                | Kappa+ or Streamhouse (warehouse-as-source)                               |
| Schema evolution as sources and agents change independently                | §4.9                      | Kappa (single job graph redeploy); declarative features help              |
| Observability / lineage                                                    | §4.8                      | Kappa / single job graph                                                  |
| Small team / thesis-scope ops                                              | §4.1                      | Kappa — Lambda's two-stack operational cost is disproportionate at thesis scale |

The fast-vs-batch distinction in §3.1 of the problem statement is real and useful, but the literature unanimously argues it should be implemented as a **topology decision inside one streaming job**, not as an **architecture decision across two engines**. Kreps [1], Akidau [3], Kleppmann [9][25], and the Uber [5][6] / Netflix [17][18] / LinkedIn [7][8] production experiences all point this way. The AI-specific literature [11][12][13][19][28] reinforces it because agents are especially intolerant of batch/stream drift.

The one place the thesis could still legitimately *look* Lambda is in physical deployment — e.g. a dedicated low-latency Kafka topic ("hot") and a dedicated windowed-output topic ("warm") feeding the Context Engine, with both produced by the same Flink (or equivalent) job graph. Users of the system would perceive two paths; engineers would maintain one.

## 6. Recommendation

**Primary recommendation: a streaming-first architecture in the Kappa+ / Streamhouse family, with a bifurcated-sink topology.**

Concretely for the thesis prototype:

- **Log substrate:** Kafka (or Redpanda) for the recent window; raw events also tiered into object storage via Kafka tiered storage or an Iceberg/Delta table, to make multi-week replay affordable.
- **Processing engine:** Apache Flink (or, if team familiarity demands, Spark Structured Streaming). Flink is the best-documented choice for the Kappa+ pattern [5][6][11] and Netflix Keystone [17] / Uber [6] precedents are directly applicable.
- **Event envelope / schema:** a single canonical schema (Avro or Protobuf with a schema registry) written once per source adapter, satisfying the "add a source without changing downstream" NFR of §4 in the problem statement.
- **Classification / routing (RQ2):** expressed as branches of the job graph. Rule-based / metadata-driven predicates produce the hot-path emit; the unmatched remainder flows into windowed aggregation. A learned classifier can be added as an asynchronous enrichment stage.
- **Hot path → Context Engine:** low-latency push channel (Kafka topic + MCP / WebSocket / subscription API per §3.4), intended for sub-second delivery.
- **Warm path → Context Engine:** event-time windowed aggregation (tumbling or session, with watermarks per [16]) producing rolled-up context objects (entity summaries, thread reconstruction) that land in the shared-memory store.
- **Shared memory (RQ5):** a combination of (a) a vector index for semantic retrieval and (b) a structured KV/table store for entity state — consistent with Michelangelo Palette [22], Chronon [10], and Confluent's Real-Time Context Engine [11].
- **Replay:** a "snapshot query" mode [11] or Kappa+ Hive-as-source connector [5] that runs the same job graph over historical object-store partitions. This rebuilds the shared memory whenever aggregation logic changes, which will happen often as the thesis explores RQ3.

**Caveats and conditions under which I'd change this recommendation.**
- If the thesis scope expanded to include *analytical reporting against years of history with strict numerical correctness* (e.g. finance), Lambda's batch layer might still earn its keep; columnar scans in dedicated OLAP engines remain unbeaten there [15]. Not the case here.
- If the fast-path SLO tightened into single-digit milliseconds (e.g. ad bidding), an in-memory specialised stream engine (Hazelcast, Redis Streams) might warrant a truly separate path — in effect a deliberate micro-Lambda. Not the case at "hundreds of ms for a human-visible alert."
- If replay cost dominates because raw events are huge and retention is long, it is worth investing in Streamhouse / Iceberg-backed logs [24] earlier rather than later; defer the decision past the prototype only if event volume stays at the thousands/minute range.
- If the team has strong Spark/Delta skills and no Flink experience, Delta-Lake-on-Spark-Structured-Streaming is an acceptable substitute [20] — the architectural argument is identical.

## 7. Open questions / gaps in the literature

- **Benchmarks on AI-agent context workloads are almost nonexistent.** All the Lambda-vs-Kappa literature predates agentic AI; the 2025 Confluent [11][12], Timeplus [13], and Waehner [19] pieces are vendor-aligned advocacy without peer-reviewed evaluation. A real contribution of this thesis could be an empirical comparison on a representative trace.
- **Defining "optimality of context"** (problem statement §3.3) is unresolved. The streaming literature measures throughput, latency, correctness, watermark accuracy [16]; the LLM literature measures answer quality; there is no published bridge. This is RQ3's problem and is genuinely open.
- **Consistency semantics between a vector store and a structured store** updated by the same streaming job — snapshot isolation vs eventual consistency — is underspecified in feature-store literature [10][22] and barely touched in agent-memory literature [28].
- **Contradictions in the literature.** Kreps [1] and Akidau [3] argue streaming strictly dominates batch; practitioners at Uber [5] and in the Delta camp [20] concede that for truly long-horizon reprocessing, streaming-against-Kafka is worse than columnar batch — hence Kappa+ and Delta explicitly read columnar files from a stream engine. The resolution in the field is that "Kappa" in 2025 *already means* Kappa+-style warehouse-backed replay, not Kreps's 2014 Kafka-only vision. Any thesis citing "Kappa" should be explicit about which it means.
- **Onboarding new sources vs. onboarding new agents** — the problem statement NFRs — get a lot of attention in the Brooklin [8] and CDC literature (source side) but very little on the agent side; most agent frameworks assume a fixed data surface. MCP [11] is an emerging standard but barely empirically studied.
- **Cost of LLM-in-the-loop aggregation** (problem statement RQ3) is absent from the streaming-architecture literature. If the warm path invokes an LLM per window, the stream engine's throughput model needs revisiting — a side-effectful, high-latency UDF is not what Flink's optimiser was designed for.

## 8. References

1. Kreps, J. (2014-07-02). *Questioning the Lambda Architecture*. O'Reilly Radar. <https://www.oreilly.com/radar/questioning-the-lambda-architecture/>
2. Streamkap (2024). *The Kappa Architecture: Simplifying Data Pipelines with Streaming*. <https://streamkap.com/resources-and-guides/kappa-architecture-guide>
3. Akidau, T. (2015-08-05). *Streaming 101: The world beyond batch*. O'Reilly Radar. <https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/> ; and *Streaming 102*, <https://www.oreilly.com/radar/the-world-beyond-batch-streaming-102/>
4. Marz, N., & Warren, J. (2015). *Big Data: Principles and best practices of scalable realtime data systems*. Manning. <https://www.manning.com/books/big-data>
5. Naik, R., et al. (2021). *Designing a Production-Ready Kappa Architecture for Timely Data Stream Processing*. Uber Engineering Blog. <https://www.uber.com/us/en/blog/kappa-architecture-data-stream-processing/> (See also *Moving from Lambda and Kappa to Kappa+ at Uber*, Flink Forward SF 2019, <https://www.slideshare.net/FlinkForward/flink-forward-san-francisco-2019-moving-from-lambda-and-kappa-architectures-to-kappa-at-uber-roshan-naik>.)
6. Fu, Y., & Soman, C. (2021). *Real-time Data Infrastructure at Uber*. In *Proc. SIGMOD '21*. arXiv:2104.00087. <https://arxiv.org/abs/2104.00087>
7. Noghabi, S. A., et al. (2017). *Samza: Stateful Scalable Stream Processing at LinkedIn*. *PVLDB* 10(12). <https://dl.acm.org/doi/10.14778/3137765.3137770>
8. LinkedIn Engineering (2017). *Streaming Data Pipelines with Brooklin*. <https://engineering.linkedin.com/blog/2017/10/streaming-data-pipelines-with-brooklin>; and *Revolutionizing Real-Time Stream Processing: 4 Trillion Events Daily at LinkedIn* (2023). <https://engineering.linkedin.com/blog/2023/revolutionizing-real-time-streaming-processing--4-trillion-event>
9. Kleppmann, M. (2015-03-04). *Turning the database inside-out with Apache Samza*. <https://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html>
10. Simha, N. (2023). *Chronon — A declarative feature engineering framework*. Airbnb Tech Blog / chronon.ai. <https://chronon.ai/> and <https://medium.com/airbnb-engineering/chronon-a-declarative-feature-engineering-framework-b7b8ce796e04>
11. Confluent (2025-10-29). *Introducing Real-Time Context Engine for AI*. <https://www.confluent.io/blog/introducing-real-time-context-engine-ai/>
12. Confluent (2025-10-29). *Confluent Launches Confluent Intelligence to Solve the AI Context Gap* (press release). <https://investors.confluent.io/news-releases/news-release-details/confluent-launches-confluent-intelligence-solve-ai-context-gap>
13. Timeplus (2025). *Real-Time Streaming as the Context Layer for AI Agents*. <https://www.timeplus.com/post/context-layer-for-ai-agents>
14. CortexFlow (2024). *Revisiting the Lambda Architecture: Challenges and Alternatives*. The Software Frontier. <https://medium.com/the-software-frontier/revisiting-the-lambda-architecture-challenges-and-alternatives-1da4fd5ce08f>
15. Flexera (2026). *Kappa Architecture 101: Deep dive into stream-first design*. <https://www.flexera.com/blog/finops/kappa-architecture/>
16. Begoli, E., Akidau, T., Hueske, F., et al. (2021). *Watermarks in Stream Processing Systems: Semantics and Comparative Analysis of Apache Flink and Google Cloud Dataflow*. *PVLDB* 14(12). <http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf>
17. Netflix Technology Blog (2018-09). *Keystone Real-time Stream Processing Platform*. <https://netflixtechblog.com/keystone-real-time-stream-processing-platform-a3ee651812a>
18. Netflix Technology Blog (2016-03). *Stream-processing with Mantis*. <https://netflixtechblog.com/stream-processing-with-mantis-78af913f51a6>
19. Waehner, K. (2025-07-08). *The Rise of Kappa Architecture in the Era of Agentic AI and Data Streaming*. <https://www.kai-waehner.de/blog/2025/07/08/the-rise-of-kappa-architecture-in-the-era-of-agentic-ai-and-data-streaming/>
20. Databricks (2020-11-20). *Delta vs. Lambda: Why Simplicity Trumps Complexity for Data Pipelines*. <https://www.databricks.com/blog/2020/11/20/delta-vs-lambda-why-simplicity-trumps-complexity-for-data-pipelines.html>
21. Gillet, A., Leclercq, É., & Cullot, N. (2021). *Lambda+, the Renewal of the Lambda Architecture: Category Theory to the Rescue*. In *Advanced Information Systems Engineering (CAiSE 2021)*, LNCS 12751, Springer. <https://doi.org/10.1007/978-3-030-79382-1_23>
22. Uber Engineering (2019/2023). *Michelangelo Palette: A Feature Engineering Platform at Uber* (InfoQ talk + slides) and *Palette Meta Store Journey*. <https://www.infoq.com/presentations/michelangelo-palette-uber/> ; <https://www.uber.com/us/en/blog/palette-meta-store-journey/>
23. Akidau, T., Chernyak, S., & Lax, R. (2018). *Streaming Systems: The What, Where, When, and How of Large-Scale Data Processing*. O'Reilly. <https://www.oreilly.com/library/view/streaming-systems/9781491983867/>
24. Ververica (2024). *From Kappa Architecture to Streamhouse: Making the Lakehouse Real-Time*. <https://www.ververica.com/blog/from-kappa-architecture-to-streamhouse-making-lakehouses-real-time>
25. Kleppmann, M. (2016). *Making Sense of Stream Processing*. O'Reilly. <https://www.oreilly.com/library/view/making-sense-of/9781492042563/>
26. Kleppmann, M. (2017, 2nd ed. 2025 with Riccomini). *Designing Data-Intensive Applications*, ch. 11. O'Reilly. <https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/>
27. Apache Beam project. *Apache Beam: A unified model for batch and streaming*. <https://beam.apache.org/> ; see also Apache Flink Runner docs, <https://beam.apache.org/documentation/runners/flink/>
28. Liu, S., et al. (2025/2026). *Memory in the Age of AI Agents: A Survey* and related works (A-MEM, MIRIX, AgeMem). GitHub paper list: <https://github.com/Shichun-Liu/Agent-Memory-Paper-List> ; representative paper: Xu et al., *A-Mem: Agentic Memory for LLM Agents*, arXiv:2502.12110, <https://arxiv.org/abs/2502.12110>
