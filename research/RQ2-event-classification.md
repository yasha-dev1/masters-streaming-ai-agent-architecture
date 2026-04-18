# RQ2 — Real-time Classification of Heterogeneous Events into Fast-path vs. Batch-path

**Thesis context:** A Context Management System sits between heterogeneous event sources (Slack, JIRA, e-commerce, observability feeds, etc.) and downstream AI agents. Its Processing Engine has two lanes: a **fast path** (per-event, low-latency, immediate agent attention) and a **batch path** (aggregation/summarization). Traffic is thousands of events per minute from arbitrary producers. RQ2 asks: *How should events from arbitrary sources be classified in real time into fast-path vs. batch-path?*

---

## 1. Executive Summary

Classifying arriving events into "act-now" versus "aggregate-later" is the *dispatcher decision* of the whole architecture: it determines latency, cost, and false-alarm rates end-to-end. Reviewing the literature on rule engines, event schemas, lightweight classifiers, LLM cascades, and online learning, no single approach dominates on all the relevant axes (latency, accuracy, extensibility, drift resistance, explainability). Rule/metadata methods [1][2][3] are millisecond-latency and explainable but brittle to new sources. Small ML classifiers (fastText, DistilBERT) are fast and accurate once trained [4][5][6] but need labels. LLM-as-classifier is accurate and zero-shot [7][8] but too slow and expensive for the fast path; it is best used offline as a *labeling oracle* [9][10]. The evidence from FrugalGPT [11], RouteLLM [12], C3 cascades [13], and the Unified Routing/Cascading framework [14] strongly supports a **hybrid cascade**: (a) a metadata/CloudEvents envelope [3] normalizes per-source self-classification, (b) a declarative rule layer [1][2] resolves unambiguous cases at sub-millisecond cost, (c) a small distilled classifier [4][5][6][15] handles ambiguous events at single-digit-ms cost, and (d) an LLM oracle audits a sample and generates labels [9][10], while (e) a feedback loop from downstream agents drives online adaptation [16][17] and concept-drift detection [18][19]. This recommendation is elaborated in §6.

---

## 2. Problem Framing

### 2.1 Latency budget
If the fast path targets end-to-end p95 latencies in the hundreds of milliseconds, the classifier itself must complete in roughly 1–20 ms per event [20][21]. At thousands of events per minute (say, 10k/min ≈ 167 events/s at peak), this is not strictly "high throughput" by streaming standards but the *tail* latency matters: a batch call to an LLM API typically takes 300 ms–2 s [11][12], which is an order of magnitude above what the fast path can tolerate per event. This immediately rules out synchronous LLM classification on the hot path unless amortized or cascaded.

### 2.2 Asymmetric error cost
Missing an urgent event (a false negative — routing a real P1 to the batch path) is strictly worse than a false positive (routing a non-urgent event to the fast path, which just increases noise and agent cost). In incident-management tooling, noise reduction is a first-order goal [22][23], but alert *suppression* is tolerated only when recall of true incidents remains high. This resembles crisis-informatics urgency detection, where recall of actionable tweets is the headline metric [24][25]. The classifier must therefore optimize **recall@low-FPR** or equivalently a **cost-sensitive** objective — not plain accuracy.

### 2.3 Source heterogeneity
A JIRA issue has structured priority, reporter, components; a Slack message is free text with @-mentions and channel context; an e-commerce event is a JSON payload with fields like `order_total` and `status`. A uniform classifier cannot ignore this: different sources expose different *features*, different *urgency semantics*, and different *label distributions*. Common envelope standards such as CloudEvents [3] exist exactly to provide a consistent minimum contract across producers (id, source, type, time, plus extension attributes like a hypothetical `urgency` hint). This does not eliminate heterogeneity — it just pushes source-specific logic into adapters [1][2] or metadata extensions.

### 2.4 Cross-source context
Some events are urgent *only* conditional on external state. A Slack message "prod is down" is P1 only if no parallel JIRA P1 already exists for the incident; an e-commerce refund event is urgent only for VIP customers. This is the hardest case, and it implies that the classifier may need to *look up* cross-source context mid-decision — either via a low-latency feature store [20] or by deferring to a slower second-stage classifier [11][14].

---

## 3. Survey of Approaches

### 3.1 Rule-based / per-source adapters
**How:** A declarative DSL (Drools rules [1][26], JSON-Logic [2], SpEL) expresses conditions such as `jira.priority in {P1, P2}` or `slack.text contains @oncall`. Each source has an adapter that maps raw event → features → rule evaluation.
**Strengths:** Sub-millisecond latency (in-memory pattern matching [1]); complete explainability (which rule fired); zero training data required; easy to audit. Drools CEP even supports sliding windows and temporal operators directly on the event stream [26].
**Weaknesses:** Brittleness — new sources require hand-written adapters; rule maintenance scales poorly; no generalization to unseen phrasing (e.g., "hey someone awake?" in Slack is urgent but does not @mention).
**Cost/latency:** O(μs–ms) per event; zero inference cost; O(N·M) expert-time cost for N sources and M rules.
**Literature:** Drools documentation and CEP chapter [1][26]; HPE microservices CEP case [27]; JsonLogic / Microsoft RulesEngine [2][28]; distributed rule-based expert systems for event streams [29].

### 3.2 Metadata-driven common envelope
**How:** Every producer emits events wrapped in a standardized envelope (e.g., CloudEvents v1.0 [3]) with a small, agreed-upon extension attribute like `urgencyhint ∈ {high, normal, low}`. The router simply reads the header. JIRA's own priority field is an in-domain example.
**Strengths:** Zero ML inference; source-provider knows its own semantics best; decentralized scaling.
**Weaknesses:** Requires producers to cooperate (often impossible for third-party systems); trust problem (producers over-claim "urgent"); legacy and SaaS events rarely carry this metadata; heterogeneity in what "urgent" *means* across producers persists.
**Cost/latency:** Essentially free per event; high upfront coordination cost.
**Literature:** CloudEvents v1.0 spec and extension-attribute model [3]; Google Cloud Eventarc, Azure Event Grid, AWS EventBridge implementations [30][31][32] show industry consensus but also the fact that `urgency` is *not* a standardized attribute — it must be a user extension, so governance and drift are real issues.

### 3.3 Small ML classifier
**How:** A lightweight model scores urgency per event. Candidates: fastText (bag-of-tricks linear model over subword embeddings) [4]; DistilBERT (97% of BERT quality at 60% speed) [5]; TinyBERT (9.4× faster than BERT, ~96.8% performance) [6]; FastBERT with self-distillation and adaptive exit [33]; logistic regression / gradient-boosted trees over sentence embeddings.
**Strengths:** Generalizes beyond hand-written rules; can combine text and metadata features; proven on ticket-priority [34] and crisis-urgency [24][25] tasks with F1 >0.8; inference latency 1–10 ms on CPU for distilled or linear models [4][5][6].
**Weaknesses:** Needs labeled data (hundreds–thousands of examples); retraining overhead on new sources; opacity compared to rules; subject to concept drift [18][19].
**Cost/latency:** Training: hours–days; inference: ~1 ms (fastText) to ~10 ms (DistilBERT CPU); operational cost negligible vs. LLM.
**Literature:** fastText [4]; DistilBERT [5]; TinyBERT [6]; FastBERT [33]; urgency-in-crisis-tweets [24][25]; JIRA priority prediction case studies [34]; Gmail Priority Inbox as a production analog of per-user importance scoring [35].

### 3.4 LLM-as-classifier
**How:** A hosted or self-served LLM (GPT-4-class, Llama-3, Gemini) reads the raw event and emits a class label, often with a short rationale. Prompting strategies include CARP (clue-and-reasoning prompting) [7][8].
**Strengths:** Zero/few-shot across arbitrary sources; handles free-form text and nuance (sarcasm, implicit urgency); robust to new event types without retraining.
**Weaknesses:** Latency typically 300 ms–several seconds per call [11][12]; USD-cents per 1000 tokens cost adds up at thousands of events per minute; hallucinated labels; rate-limit and availability risk.
**Cost/latency:** 10²–10³× more expensive than a small classifier per event; impractical as the sole fast-path decision point.
**Literature:** *Text Classification via Large Language Models* (CARP) [7]; reviews of LLMs for text classification [8][36]; *Smart Expert System* uses LLMs as classifiers with prompt engineering [37]. Industry practice (Gmail AI Inbox) has moved to LLM-assisted prioritization but for email-scale, not millisecond-latency streams [35].

### 3.5 Hybrid / cascade
**How:** Chain multiple classifiers of increasing cost. A cheap filter (rules or small model) decides *most* events with high confidence; only ambiguous ones escalate to the more expensive stage. The confidence threshold is the key knob [13][14][38]. FrugalGPT [11] showed up to 98% cost reduction at parity accuracy; RouteLLM [12] showed 2×+ cost savings with no quality loss; the unified routing/cascading framework [14] proves optimality conditions and shows +4% on RouterBench; Cascadia [39] is a serving-level realization. Early-exit models (FastBERT [33], early-exit CNNs [40], early-exit LLM frameworks [41]) apply the same idea *inside* a single network.
**Strengths:** Best cost/accuracy Pareto point empirically; scales gracefully; cheap stage gives fast path for the ~80–95% easy cases, expensive stage protects tail accuracy.
**Weaknesses:** More moving parts; tuning the confidence threshold is non-trivial and drift-sensitive [42]; harder to explain than a single model.
**Cost/latency:** Average latency dominated by the cheap stage; p99 dominated by escalations, bounded by SLA. Online cascade learning can adapt thresholds to streams [43].
**Literature:** FrugalGPT [11]; RouteLLM [12]; *A Unified Approach to Routing and Cascading for LLMs* [14]; *Dynamic Model Routing and Cascading: A Survey* [38]; C3 confidence-calibrated cascades [13]; BEST-Route [44]; Cost-Saving LLM Cascades with Early Abstention [42]; early-exit NNs [33][40][41]; Online Cascade Learning [43].

### 3.6 Learned routing from feedback
**How:** Treat routing as a contextual-bandit or online-learning problem. The "arm" is {fast, batch}; the reward is a downstream signal — did the agent escalate the event? did a human mark it important? did SLO breach occur? Online algorithms (Thompson sampling with sliding windows, f-DSW-TS [45], online-regression-based bandits [46]) continuously re-fit routing policy.
**Strengths:** Self-improves without manual labeling; naturally handles concept drift; aligns the objective with real downstream utility (the thing we actually care about).
**Weaknesses:** Reward is delayed and noisy (agent behavior is itself stochastic); cold-start problem on new sources; hard to offer safety guarantees; exploration means some urgent events will be deliberately mis-routed to gather data.
**Cost/latency:** Inference very cheap (linear scorer); the complexity lives in the training/update pipeline.
**Literature:** Multi-armed-bandit for concept-drift adaptation [45]; online learning in bandits with predicted context [46]; contextual-bandit survey and retail prototype [47]; universal model routing for LLM inference uses a learned-routing flavor [48].

---

## 4. Evaluation Dimensions

| Dimension | What to measure | Target for fast-path dispatcher |
|---|---|---|
| **Latency** | p50 / p95 / p99 classification time per event | p95 ≤ 20 ms, p99 ≤ 50 ms [20][21] |
| **Accuracy / recall** | Recall of true-urgent events; precision at fixed recall; ROC-AUC | Recall ≥ 0.95 at FPR ≤ 0.10 (asymmetric cost [24][25]) |
| **Cost per event** | USD / 10⁶ events, GPU-second / event | Dominated by cheap stage; cascade budget [11][14] |
| **Extensibility** | Time to onboard a new source | Hours (envelope/rules) to days (retrain) |
| **Explainability** | Can we show *why* an event was fast-pathed? | Required for audit; rules natively [1][2]; models need post-hoc tools |
| **Drift resistance** | Performance decay over time; detection lag | Monitored via prequential accuracy and ADWIN/DDM [18][19] |
| **Label requirements** | Number of labeled events needed | Few-shot via LLM oracle + weak supervision [9][10][49] |

The ground-truth label for "was this event truly urgent" is almost always *defined by downstream agent behavior*: did the agent act within its SLA, did a user re-escalate it, did a human override the routing? This makes evaluation a **delayed-reward / weak-label** problem, aligning it with both weak supervision (Snorkel [49]) and online bandits [45][46]. Prequential accuracy with forgetting [19] is the recommended continuous-evaluation metric; ADWIN and DDM detect significant drift [18].

---

## 5. Applicability to this Thesis

Mapping the above dimensions to the thesis setting:

- **Latency budget (1–20 ms)** precludes synchronous LLM calls on every event [11][12]. A cheap first-stage is non-negotiable.
- **Source heterogeneity (Slack, JIRA, e-commerce)** makes a single monolithic model brittle; but writing exhaustive hand-rules for every source also scales poorly [1]. A *common envelope* (CloudEvents [3]) plus *source adapters* plus a *general text/metadata classifier* together cover the cases.
- **Asymmetric cost of missed urgency** favors a two-stage design where the cheap first stage is tuned for high recall, and anything it is not confident about is *conservatively* escalated — either to the fast path directly (precautionary) or to a slower LLM audit [11][14][42].
- **Thousands of events per minute** is within the range where a small ML model on CPU is comfortable (fastText inference is in μs per event [4]; DistilBERT ~10 ms [5]), and an LLM audit of even 5–10% of events is financially plausible.
- **Cross-source context** (Slack message urgent because of parallel JIRA P1) can be handled by enriching events with a low-latency lookup into a per-entity state store before classification, as DoorDash does [20].
- **Drift** — a new Slack channel, a new product launch, a new error-message template — is near-certain; the design must plan for it [18][19][45].
- **Labels come from downstream agent behavior**, so the feedback loop from §3.6 is not optional; it is the main mechanism that keeps the system calibrated over time [16][17][45][46].

The thesis is at problem-framing stage, so the question is less "which model has highest F1 on benchmark X" and more "what is the *architecture* within which we can evolve from rules → small model → cascade → feedback-driven routing without rewriting the system." This favors a layered design.

---

## 6. Proposed Recommendation

**Adopt a layered hybrid cascade** with the following four stages, evaluated in order, short-circuiting as soon as a confident decision is made:

1. **Envelope normalization (CloudEvents v1.0 [3]) + per-source adapter [1][2]).** Each producer or ingest adapter emits a CloudEvent whose extension attributes include at least a source-native priority (`jira.priority`, `slack.mention_targets`, `ecom.order_total`) plus any producer-supplied `urgency_hint`. This is essentially free per event and handles the "structured obvious" cases (JIRA P1, Slack @oncall) at ≪1 ms.

2. **Declarative rule layer (json-logic / Drools [1][2][26]).** On top of the envelope, a small set of hand-curated rules produces a fast-path decision for events that match an unambiguous pattern. Keeps explainability and audit trail for the majority of production traffic.

3. **Small ML classifier (distilled BERT or fastText over features from envelope + text [4][5][6][33]).** Trained on labels produced by (a) human curation on a seed set, (b) LLM-oracle labeling on a sampled corpus [9][10], and (c) weak supervision via labeling functions derived from step 2 rules [49]. Invoked only when steps 1–2 are non-confident; target p95 ≤ 10 ms.

4. **LLM oracle (GPT-4-class, async / offline) [7][8][11].** Not on the hot path. Used (i) to label historical data for training the small model; (ii) to audit a sampled slice of live decisions (e.g., 1%) for drift detection; (iii) optionally to "second-opinion" the small model's low-confidence events on the batch path.

5. **Feedback-driven online adaptation.** Downstream agent outcomes (did-escalate / SLO-breach / user-override) are fed back as rewards to a contextual bandit [45][46][47] which re-tunes the confidence threshold between stages 3 and 4 and (eventually) retrains the small model. Prequential accuracy and ADWIN [18][19] monitor drift.

**Why this design wins on the evaluation dimensions:**
- *Latency:* steps 1–2 dispatch the majority of events in sub-ms; step 3 adds ~10 ms for the remainder; step 4 is never on the hot path.
- *Accuracy:* distilled model + rules + LLM oracle training is the empirically strongest combination [9][10][11][14].
- *Cost:* cascade design from FrugalGPT/RouteLLM literature [11][12] gives near-LLM quality at a fraction of LLM cost.
- *Extensibility:* a new source drops in by writing an envelope adapter + optional rules; the ML model generalizes without code changes; LLM oracle provides labels cheaply.
- *Explainability:* most decisions (steps 1–2) carry a rule id; model decisions carry a confidence; LLM audits generate rationales.
- *Drift resistance:* built-in via step 5 feedback and prequential monitoring.

This is consistent with the pattern prevailing in incident-management AIOps (PagerDuty Event Intelligence [22][50], Datadog AIOps [23]), which combines rule-based correlation with ML noise reduction, and with the SLM-first trend documented in 2025–2026 industry analysis [15][51].

---

## 7. Open Questions

1. **Where do labels come from?** Downstream agent behavior is the "true" oracle but signals are delayed, noisy, and biased (agents only see what the router sent them). Proposals: (a) log *both* paths' events, use shadow-mode evaluation on a sampled mirror fast-path; (b) LLM oracle on a random sample for calibration; (c) weak-supervision labeling functions [49] combining multiple heuristics.
2. **How to classify an event that needs cross-source context?** A Slack "prod down" matters only if no parallel JIRA P1 already exists. Options: enrich events with a low-latency feature lookup (cost: extra ms, state-store engineering); or route ambiguous cross-source events to a slower cascade stage that can afford a lookup [14][38].
3. **Threshold tuning under drift.** The cheap-stage confidence threshold determines the cascade budget; it is drift-sensitive [42]. Should it be re-tuned nightly (batch), continuously (online bandit), or event-rate-adaptive?
4. **How much of the "urgent" label is per-user / per-team?** Gmail Priority Inbox is per-user [35]; our agents may each have different tolerance for noise. Should the classifier be per-downstream-agent, or global with a per-agent threshold?
5. **Governance of the envelope.** If producers self-classify (`urgency_hint`), how do we prevent over-claiming? Reputation weighting? LLM audit of producer calibration? [3]
6. **Safety of exploration in the bandit.** Deliberately routing a small fraction of possibly-urgent events to the batch path to gather data is risky. Can we use counterfactual / off-policy evaluation instead [47]?
7. **Cold start on a new source.** Until the small model has seen traffic, rules and producer metadata are the only signal. What is the minimum rule/envelope coverage required to accept a new source into production?
8. **Multi-label vs. single-label urgency.** Is "fast vs. batch" really binary, or do we need tiers (P0 / P1 / normal / bulk)? Multidimensional classification literature [8] suggests the latter is often more tractable and more useful.

---

## 8. References

[1] Drools. *Drools Documentation — Rule Engine*. Red Hat / KIE community. https://docs.drools.org/8.38.0.Final/drools-docs/docs-website/drools/rule-engine/index.html  
Reference implementation of a production rule engine with complex-event-processing, sliding windows, and stream mode — establishes the baseline for rule-based event classification.

[2] JsonLogic project. *JsonLogic — rules in JSON*. https://jsonlogic.com/  
Small, side-effect-free declarative rule format for portable business rules; represents the lightweight end of the rule-engine space.

[3] Cloud Native Computing Foundation. *CloudEvents v1.0.2 Specification*. GitHub: cloudevents/spec. https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md  
CNCF graduated standard for event envelopes across producers; required context attributes and optional extension attributes enable a consistent cross-source classification surface.

[4] Joulin, A., Grave, E., Bojanowski, P., Mikolov, T. (2016/2017). *Bag of Tricks for Efficient Text Classification* (fastText). EACL 2017. arXiv:1607.01759. https://arxiv.org/abs/1607.01759  
fastText classifies half a million sentences among 312K classes in under a minute on CPU — the canonical cheap, high-throughput text classifier for a first-stage filter.

[5] Sanh, V., Debut, L., Chaumond, J., Wolf, T. (2019). *DistilBERT, a distilled version of BERT: smaller, faster, cheaper and lighter*. arXiv:1910.01108. https://arxiv.org/abs/1910.01108  
Retains 97% of BERT's GLUE performance at 40% fewer parameters and 60% faster inference; the workhorse distilled classifier for low-latency NLP.

[6] Jiao, X. et al. (2019). *TinyBERT: Distilling BERT for Natural Language Understanding*. arXiv:1909.10351. https://arxiv.org/abs/1909.10351  
4-layer TinyBERT achieves 96.8% of BERT-base performance while being 7.5× smaller and 9.4× faster at inference.

[7] Sun, X. et al. (2023). *Text Classification via Large Language Models* (CARP prompting). arXiv:2305.08377. https://arxiv.org/abs/2305.08377  
Clue-and-Reasoning prompting shows LLMs can match or surpass fine-tuned classifiers on many benchmarks, but at substantial latency/cost — justifies LLM-as-oracle, not LLM-on-hot-path.

[8] Bucher, M., Martini, M. (2024). *Large Language Models for Text Classification: Case Study and Comprehensive Review*. arXiv:2501.08457. https://arxiv.org/html/2501.08457v1  
Comprehensive comparison of LLMs and traditional classifiers; LLMs win on complex multi-class tasks but at the cost of long inference times.

[9] Zhao, W. et al. (2024). *Knowledge Distillation in Automated Annotation: Supervised Text Classification with LLM-Generated Training Labels*. arXiv:2406.17633. https://arxiv.org/html/2406.17633v1  
Shows that classifiers fine-tuned on LLM-generated labels match those trained on human labels — validates using LLMs as a labeling oracle to train the small fast-path model.

[10] *Performance-Guided LLM Knowledge Distillation for Efficient Text Classification at Scale* (PGKD). (2024). arXiv:2411.05045. https://arxiv.org/abs/2411.05045  
Distilled task-specific models up to 130× faster and 25× cheaper than LLMs at inference, via active LLM-driven labeling and hard-negative mining.

[11] Chen, L., Zaharia, M., Zou, J. (2023). *FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance*. arXiv:2305.05176. https://arxiv.org/abs/2305.05176  
Classic LLM-cascade paper: sequential cheap→expensive LLM querying with confidence-gated escalation delivers up to 98% cost reduction at GPT-4 parity.

[12] Ong, I. et al. (2024). *RouteLLM: Learning to Route LLMs with Preference Data*. arXiv:2406.18665. https://arxiv.org/abs/2406.18665  
Trains routers from ~120K preference samples; achieves 85% cost reduction on MT-Bench and ≥2× savings on MMLU and GSM8K without quality loss.

[13] Jin, T. et al. (2024). *C³: Confidence Calibration Model Cascade for Inference-Efficient Cross-Lingual NLU*. arXiv:2402.15991. https://arxiv.org/pdf/2402.15991v1  
Confidence-calibrated cascades are central to the thesis's proposed stage-2→stage-3 decision rule.

[14] Dekoninck, J. et al. (2024). *A Unified Approach to Routing and Cascading for LLMs*. arXiv:2410.10347. https://arxiv.org/abs/2410.10347  
Proves optimality conditions for routing vs. cascading and combines them; +4% on RouterBench over naïve baselines — directly informs the fast-path dispatcher design.

[15] Belcak, P., Heinrich, G. et al. (2025). *Small Language Models are the Future of Agentic AI*. arXiv:2506.02153. https://arxiv.org/pdf/2506.02153  
Argues that 7B-parameter SLMs are 10–30× cheaper in latency and FLOPs than 70–175B LLMs while being adequate for most agent subtasks — strong support for SLM-based cheap-stage.

[16] Wilson, J. et al. (2024). *Multi-armed bandit based online model selection for concept-drift adaptation*. Expert Systems. https://onlinelibrary.wiley.com/doi/10.1111/exsy.13626  
Shows how f-DSW Thompson Sampling can pick among classifier ensembles on a drifting stream — applicable to routing among cheap/expensive classifiers in production.

[17] Guo, Y., Xu, Z. (2024). *Online Learning in Bandits with Predicted Context*. AISTATS 2024. arXiv:2307.13916. https://arxiv.org/abs/2307.13916  
Handles the realistic case where the context fed to the bandit is itself a noisy ML prediction — directly analogous to feeding a small classifier's output to a router.

[18] Arora, M. et al. (2024). *A systematic review on detection and adaptation of concept drift in streaming data using machine learning techniques*. WIREs Data Mining Knowl. Discov. https://wires.onlinelibrary.wiley.com/doi/10.1002/widm.1536  
Survey of drift detection (DDM, EDDM, ADWIN) and adaptation (ensembles) — defines the monitoring layer.

[19] Žliobaitė, I. et al. (2017). *Concept drift in Streaming Data Classification: Algorithms, Platforms and Issues*. Procedia Computer Science 132. https://www.sciencedirect.com/science/article/pii/S1877050917326881  
Canonical reference for prequential accuracy with forgetting and for the general framework of streaming classifier evaluation under drift.

[20] DoorDash Engineering. (2024). *Building scalable real time event processing with Kafka and Flink*. https://careersatdoordash.com/blog/building-scalable-real-time-event-processing-with-kafka-and-flink/  
Industry reference architecture for low-latency, high-throughput event routing; concrete latency targets and feature-store patterns.

[21] Waehner, K. (2026). *How Apache Kafka and Flink Power Event-Driven Agentic AI in Real Time*. https://www.kai-waehner.de/blog/2025/04/14/how-apache-kafka-and-flink-power-event-driven-agentic-ai-in-real-time/  
Direct treatment of the Kafka+Flink+AI agent pattern that frames this thesis; latency and routing implications.

[22] PagerDuty. *Event Intelligence — Intelligent Triage* (press release and product page). https://www.pagerduty.com/newsroom/intelligent-triage-dashboards/ ; https://www.pagerduty.com/platform/incident-management/  
Industrial precedent for ML-driven alert triage (grouping, noise reduction, triage recommendations) — a direct analog of the proposed Context Management System's dispatcher.

[23] Datadog. *Detect anomalies before they become incidents with Datadog AIOps*. https://www.datadoghq.com/blog/early-anomaly-detection-datadog-aiops/  
Production AIOps pipeline combining rules, ML anomaly detection, and correlation — practical baseline for event triage in real systems.

[24] Singh Sachdeva, N., Chandrasekhar, V. (2020). *On detecting urgency in short crisis messages using minimal supervision and transfer learning*. Social Network Analysis and Mining 10(70). https://link.springer.com/article/10.1007/s13278-020-00670-7 (extended arXiv:1907.06745: https://arxiv.org/abs/1907.06745)  
Directly relevant domain: classifying short messages as urgent/non-urgent with transfer learning; RoBERTa-based classifiers achieve >25 F1-point lift over baselines.

[25] Dutta, A. et al. (2020). *Detecting Urgency Status of Crisis Tweets: A Transfer Learning Approach for Low Resource Languages*. COLING 2020. https://aclanthology.org/2020.coling-main.414/  
Cross-lingual transfer for urgency detection; shows robustness across producers of varying linguistic form — analogous to heterogeneous event sources.

[26] Red Hat / KIE Community. *Event Driven Drools: Complex Event Processing Explained* (2021). https://blog.kie.org/2021/10/event-driven-drools-cep-complex-event-processing-explained.html  
Shows Drools in stream mode with sliding windows; direct candidate for rule layer in the thesis architecture.

[27] HPE Developer. *Better Complex Event Processing at Scale Using a Microservices-based Streaming Architecture*. https://developer.hpe.com/blog/better-complex-event-processing-at-scale-using-a-microservices-based-str/  
Microservice + CEP reference architecture for high-volume event streams.

[28] Microsoft. *RulesEngine — A JSON based Rules Engine with Dynamic Expression Support*. https://microsoft.github.io/RulesEngine/  
Open-source JSON rule engine used in .NET pipelines; practical alternative to Drools for lighter deployments.

[29] Chen, Z. (2019). *A Distributed Rule-Based Expert System for Large Event Stream Processing* (PhD thesis, University of Birmingham). https://etheses.bham.ac.uk/id/eprint/9414/7/Chen2019PhD.pdf  
Academic treatment of scaling rule engines to high-volume event streams — establishes feasibility bounds.

[30] Google Cloud. *CloudEvents format — HTTP protocol binding (Eventarc docs)*. https://cloud.google.com/eventarc/docs/cloudevents  
Shows production usage of CloudEvents envelope in a managed event-routing product.

[31] Microsoft. *CloudEvents Integration with Azure Event Grid*. https://learn.microsoft.com/en-us/azure/event-grid/cloud-event-schema  
Equivalent production reference for Azure; confirms cross-cloud consensus on envelope.

[32] AWS. *Sending and receiving CloudEvents with Amazon EventBridge*. https://aws.amazon.com/blogs/compute/sending-and-receiving-cloudevents-with-amazon-eventbridge/  
Completes the major-cloud trio using CloudEvents — removes adoption risk of the envelope choice.

[33] Liu, W. et al. (2020). *FastBERT: a Self-distilling BERT with Adaptive Inference Time*. arXiv:2004.02178. https://arxiv.org/abs/2004.02178  
Early-exit inside a single BERT model; confidence-gated exit is conceptually identical to a cascade and is deployable as a single-model fast stage.

[34] Community discussion and case studies: *Leveraging Machine Learning for Ticket Classification in Jira*, Atlassian Community (2023). https://community.atlassian.com/forums/Jira-Service-Management/Leveraging-Machine-Learning-for-Ticket-Classification-in-Jira/qaq-p/2919941 ; *Enhancing Jira with Machine Learning for Ticket Prioritization*. https://reintech.io/blog/enhance-jira-machine-learning-ticket-prioritization  
Industry practice showing RF/SVM/DistilBERT used in production for JIRA priority prediction — direct precedent for the JIRA adapter in the thesis.

[35] Aberdeen, D., Pacovsky, O., Slater, A. (2010). *The Learning Behind Gmail Priority Inbox*. NIPS 2010 workshop. https://research.google.com/pubs/archive/36955.pdf  
Classic production priority-classification system with per-user models and delayed-reward learning; the closest large-scale analog to the proposed dispatcher.

[36] *Text Classification in the LLM Era — Where do we stand?* (2025). arXiv:2502.11830. https://arxiv.org/html/2502.11830v1  
2025 survey placing LLM, SLM, and classical text classifiers on a cost-accuracy frontier; useful for dimensioning the cascade stages.

[37] Fang, B. et al. (2024). *Smart Expert System: Large Language Models as Text Classifiers*. arXiv:2405.10523. https://arxiv.org/html/2405.10523v1  
Practical framework for LLM-as-classifier with prompt engineering; useful for the oracle stage.

[38] *Dynamic Model Routing and Cascading for Efficient LLM Inference: A Survey* (2026). arXiv:2603.04445. https://arxiv.org/html/2603.04445v1  
Up-to-date survey of the routing/cascading design space; direct reference for choosing thresholds and routing policies.

[39] *Cascadia: An Efficient Cascade Serving System for Large Language Models* (2025). arXiv:2506.04203. https://arxiv.org/html/2506.04203  
Systems-level realization of cascaded inference — informs the implementation of the cascade in a streaming serving environment.

[40] *Early-exit Convolutional Neural Networks (EENets)*. (2024). arXiv:2409.05336. https://arxiv.org/html/2409.05336v1  
Canonical early-exit design with per-branch confidence thresholds; architectural inspiration for the small-model stage.

[41] *An Efficient Inference Framework for Early-exit Large Language Models* (2024). arXiv:2407.20272. https://arxiv.org/html/2407.20272v1  
Early-exit applied to LLMs at inference; a path to making even step-4 cheaper if needed.

[42] *Cost-Saving LLM Cascades with Early Abstention* (2025). arXiv:2502.09054. https://arxiv.org/html/2502.09054v1  
Abstention-aware cascades: the cheap stage can abstain instead of guessing, which is the right primitive for "route to batch if unsure" semantics.

[43] *Online Cascade Learning for Efficient Inference over Streams* (2024). arXiv:2402.04513. https://arxiv.org/html/2402.04513v2  
Directly relevant: cascades trained online on a streaming workload, matching the thesis setting.

[44] *BEST-Route: Adaptive LLM Routing with Test-Time Optimal Compute* (2025). arXiv:2506.22716. https://arxiv.org/html/2506.22716v1  
Adaptive routing that re-allocates compute per-query; relevant to feedback-driven threshold tuning.

[45] Wilson, J. et al. (2024). *Multi-armed bandit based online model selection for concept-drift adaptation*. Expert Systems. https://onlinelibrary.wiley.com/doi/10.1111/exsy.13626  
f-DSW Thompson Sampling for non-stationary MABs; exactly the algorithm family proposed for stage-5 feedback.

[46] Foster, D.J., Rakhlin, A. (2020). *Beyond UCB: Optimal and Efficient Contextual Bandits with Regression Oracles*. ICML 2020. https://dspace.mit.edu/bitstream/handle/1721.1/138306/foster20a.pdf  
Reduction of contextual bandits to online regression; foundational theory for the routing bandit.

[47] *Scalable and Interpretable Contextual Bandits: A Literature Review and Retail Offer Prototype* (2025). arXiv:2505.16918. https://arxiv.org/html/2505.16918v1  
Practical contextual-bandit deployment review, including off-policy evaluation — addresses the safety-of-exploration open question.

[48] *Universal Model Routing for Efficient LLM Inference* (2025). arXiv:2502.08773. https://arxiv.org/html/2502.08773v1  
Generalizes RouteLLM-style learned routing; useful when the set of downstream classifiers grows.

[49] Ratner, A. et al. (2017/2019). *Snorkel: Rapid Training Data Creation with Weak Supervision*. VLDB Journal 29. arXiv:1711.10160. https://arxiv.org/abs/1711.10160  
Canonical weak-supervision framework: combine noisy labeling functions (including existing rules and LLM outputs) into probabilistic labels for training the small model.

[50] PagerDuty. *Using AIOps for Better Incident Management*. https://www.pagerduty.com/resources/aiops/learn/aiops-incident-management/  
Overview of rule + ML hybrid in the AIOps space; the architectural pattern the thesis is adapting to the context-management setting.

[51] Label Your Data. *SLM vs LLM: Accuracy, Latency, Cost Trade-Offs 2026*. https://labelyourdata.com/articles/llm-fine-tuning/slm-vs-llm  
Current-year industry summary of SLM vs LLM trade-offs; quantitative grounding for cascade cost estimates.

[52] Ho, S. et al. (2023). *A Survey of Time Series Anomaly Detection Methods in the AIOps Domain*. arXiv:2308.00393. https://arxiv.org/abs/2308.00393  
Complements the classification literature with anomaly-detection methods relevant to "urgency-as-anomaly" for numeric-metric event sources.
