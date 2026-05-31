# Evaluation Methodology for an Open-Source Context Management System for Streaming AI Agents

*Committee-ready evaluation note. Status: this note fixes the "evaluation is vague" feedback by specifying a layered framework with exact, implementable metrics. It assumes the RQ2 reframe (`research/RQ2-anomaly-detection-reframe.md`) from a globally-supervised `p(urgent)` classifier to a per-entity, label-free anomaly/deviation router (Cloudflare-inspired), with deterministic rules as a hard floor and the contextual bandit demoted to sensitivity-threshold control.*

---

## 1. Evaluation philosophy

The system is a routing layer between heterogeneous event sources and stateless LLM agents. The supervisor's two critiques — (a) the fast/batch framing should mirror Cloudflare's *unsupervised, per-entity anomaly detection* rather than supervised urgency classification, and (b) the evaluation must state *how* it works and *exactly which* metrics it measures — drive a **three-layer** evaluation. Each layer answers a distinct question and is measurable independently, so a failure can be localized.

- **L1 — Routing/anomaly-detection quality (offline, replayed).** *Question: when the router fires "FAST", is the event actually one that warranted immediate push, and is it detected early?* This is a **detection** problem (rare positives, temporal structure), evaluated with imbalance-aware and latency-aware anomaly-detection metrics on a replayed trace with constructed ground truth.
- **L2 — Online behavior under drift and load (streaming, prequential + systems).** *Question: does the router stay correct, calibrated, and economical as the stream drifts, and does the pipeline meet latency/throughput SLAs under that load?* This couples ML-under-drift metrics (prequential, regret, calibration, drift recovery) with streaming-systems metrics (tail latency, throughput, lag, backpressure, recovery).
- **L3 — End-to-end agent task utility (closed-loop).** *Question: does correct routing make the downstream agent more useful — right intervention, right time, at acceptable token cost — and does push delivery beat polling?* Evaluated with timing precision/recall + timeliness, LLM-as-judge content quality, token-waste economics, and a fair push-vs-pull benchmark.

**Central hypotheses (pre-registered):**
- **H1 (anomaly reframe works):** the per-entity deviation router + rules floor achieves higher VUS-PR and affiliation-F1 than (i) a global supervised `p(urgent)` classifier and (ii) a rules-only baseline, on the same replayed trace. *(Answers critique a.)*
- **H2 (engine value is burst-specific):** the context engine's advantage in mean asymmetric reward and timing-F1 over a no-engine baseline **shrinks as burstiness B → 0**, demonstrating the value is specifically in handling bursty streams.
- **H3 (SLA + economy):** the FAST/BATCH split holds FAST-lane p99 event-time latency-to-agent under the SLA at sustainable throughput, and reduces token-waste rate vs an always-FAST baseline.
- **H4 (push beats poll):** push delivery dominates polling on the latency/cost/staleness Pareto across all swept poll intervals.

---

## 2. Benchmark / dataset

**Citation status (independently verified).** *ProAgentBench* (Tang et al., **arXiv:2602.04482**, "Evaluating LLM Agents for Proactive Assistance with Real-World Data", submitted 2026-02-04) is **real and accurate** — title, authors (Yuanbo Tang, Huaze Tang, Tingyu Cao, et al.), the 28,000+ events / 500+ hours of real user sessions, the measured **burstiness B = 0.787**, and the Task 1 = *when-to-assist* / Task 2 = *how-to-assist* decomposition all match the arXiv abstract and HTML. It is cited as a genuine external **metric template and burstiness reference**, *not* as validation of this thesis's downstream-feedback labelling scheme. **Caveat:** verify access licence (de-identified content public; raw screenshots may be restricted) before redistributing data.

**Real, reusable benchmarks.**
- **ProactiveBench / "Proactive Agent"** (Lu et al., arXiv:2410.12361; repo `github.com/thunlp/ProactiveAgent`, **Apache-2.0**) — 6,790 events (Coding 2,275 / Writing 2,354 / Daily Life 2,161) with a reward-model-as-judge (reported F1 ≈ 0.918). Best-licensed directly reusable proactive-agent benchmark; use its 2×2 {need}×{proposed} confusion structure and reward-model-as-judge pattern. *(Org is `thunlp`, not `THUDM`.)*
- **Memory benchmarks** for the context-engine claim: **LongMemEval** (Wu et al., ICLR 2025, arXiv:2410.10813, MIT licence; 500 questions, S/M variants up to ~1.5M tokens) and **LoCoMo** (Snap Research; up to 35 sessions) — used to justify Graphiti's temporal KG over flat RAG (verify LoCoMo's licence in-repo before redistribution).
- **Streaming-systems harness templates:** **Nexmark** (query/resource model), **OpenMessaging Benchmark** (driver/worker + latency-percentile reporting), and the **Karimov sustainable-throughput** framework. Cite **YSB** explicitly as the negative example (Redis bottleneck, untuned defaults) to justify departures.

**Bursty event-stream trace with MEASURED burstiness.** Because no canonical bursty-routing benchmark exists, constructing one is itself a contribution. Procedure:
1. **Record** a real trace from the three source families (Slack, JIRA, e-commerce/observability telemetry) into the Iceberg/Paimon log, preserving original event-time timestamps and CloudEvent `subject` (entity key).
2. **Measure burstiness** per source and per entity using the parameter **B** (Goh & Barabási; finite-sequence estimator Kim & Jo, *Phys. Rev. E* 94:032311, 2016): `B ∈ [−1 (regular), 0 (Poisson), +1 (maximally bursty)]`, computed from the coefficient of variation of inter-event times, plus the memory coefficient M. Use the `bursty_dynamics` package. Report measured B alongside ProAgentBench's B = 0.787 for external validity.
3. **Replay** deterministically from the Iceberg/Paimon store (a Kappa+ capability a stateful supervised model cannot match) so every condition sees the identical stream.

**Ground-truth urgency labels (and why this is hard).** "True urgency" has no clean, independent label — it is only revealed by *delayed, selection-biased, policy-dependent* downstream agent verdicts (the design's Achilles heel). The router only ever observes the lane it chose, so the batch lane is never counterfactually observed (missing-not-at-random). We therefore use a **three-tier label strategy**:
- **Silver labels:** downstream verdicts {appropriate, too_urgent, too_slow, irrelevant}, treated as noisy/delayed.
- **Injected oracle:** a known subset of events deliberately constructed as urgent/anomalous, with **hard negatives** (need-less moments contextually similar to need moments, per ProAgentBench) so precision is not inflated by trivial idle periods. This gives unbiased precision/recall/AUPRC.
- **Gold set:** a stratified human-adjudicated subset (oversampling boundary, too_urgent, too_slow cases) defining the optimal intervention point `t_optimal` and content quality; report silver-vs-gold agreement so reviewers can gauge label noise. Explicitly state which definition of `t_optimal` is used (first human/agent action timestamp vs SLA-derived vs annotator consensus), since each yields a different lead-time zero.

---

## 3. L1 — Fast-path routing as anomaly detection: metrics

The fast path is scored as **time-series anomaly detection**, not point-wise classification.

**PR-AUC vs ROC-AUC (imbalance).** Urgent events are rare, so **ROC-AUC is misleading**: `FPR = FP/(FP+TN)` and the dominant TN count keeps FPR (hence ROC-AUC) optimistically near 1 even with many false alarms. **PR-AUC** (precision vs recall, ignores TN) is the imbalance-aware scalar; **always report it against the prevalence baseline** (random PR-AUC = positive prevalence). State prevalence per source in every table, since it differs sharply across Slack/JIRA/telemetry.

**precision@k.** Precision among the top-k highest-scored events — the operationally relevant regime where agents can only act on a budget of k alerts/window.

**Range-based precision/recall (Tatbul et al., NeurIPS 2018).** Generalizes point P/R to *ranges*, decomposing each match into Existence, Overlap-size, Position bias (front/middle/back — lets us reward *early* detection), and Cardinality (penalizes fragmented detections). Requires choosing bias functions, so report as a *sensitivity check* and pair with a parameter-free metric.

**Affiliation precision/recall (Huet et al., KDD 2022).** **Parameter-free** and PA-resistant: scores by the average temporal distance from each prediction to the nearest true event (and vice-versa), normalized against a random predictor (0 = random, →1 = perfect). This is the **primary event-level metric** because it cannot be gamed by a single lucky in-segment hit.

**VUS — Volume Under the Surface (Paparrizos et al., VLDB 2022).** Adds a buffer of length *l* around each true anomaly with a label decaying from 1 to 0, then integrates range-based PR/ROC over all *l* — removing **both** the threshold **and** the window-length choice. **VUS-PR is the single headline number** (imbalance-aware + robust to boundary misalignment).

**Detection latency + NAB scoring.** Define an anomaly window around each true event; only the **earliest** true positive per window is credited, rewarded by a **scaled sigmoid** of its relative position (earlier → higher; right edge → 0) — *not* a linear decay. FPs outside windows carry negative weight; missed windows are negative FNs; the raw score normalizes to **0–100**. Use the **`reward_low_FN_rate`** profile (weights `{fn: 2.0}`) to encode *missed-urgent ≫ false-alarm*. **Also report raw median and p95 detection delay in seconds**, because NAB's normalization hides absolute latency.

**Cost-sensitive recall@low-FPR.** Report recall at a fixed, low false-alarm operating point (e.g. recall@FPR≤1%), mirroring Cloudflare tuning detection rate subject to an acceptable false-positive rate, and total expected cost `= C_FN·FN + C_FP·FP` with `C_FN ≫ C_FP`, swept over the ratio with the cost-minimizing operating point reported.

> **⚠ Pitfall — naive point-adjusted F1.** **Do NOT report point-adjusted (PA) F1 as a headline metric.** Kim et al. (AAAI 2022) prove that PA — flipping an *entire* ground-truth segment to "detected" from a *single* flagged point — inflates F1 so badly that a **uniformly random anomaly score becomes state-of-the-art**, and an untrained model is competitive once PA is forbidden. **What to report instead:** VUS-PR + affiliation-F1 as primary. If PA numbers are needed for comparability with prior work, report the **PA%K curve** (F1 vs K over K ∈ [0,100], or its area, via the `tadpak` library) — never a single K=0 point — with a one-line Kim-et-al. caveat. **Sanity check:** run a uniform-random scorer and a trivial "always-flag" detector through the *entire* suite; any metric that ranks them near the real model is broken for this setting and must be dropped.

**Recommended L1 metric set.**
- **Primary:** VUS-PR (target ≥ 0.6 at the trace's prevalence; report prevalence); affiliation-F1 (target ≥ 0.70).
- **Secondary:** PR-AUC + precision@k; NAB score under `reward_low_FN_rate` (target ≥ 60/100); recall@FPR≤1% (target ≥ 0.80 on injected-urgent); median detection delay (target within the per-source `too_slow` budget).
- **Reference floor the learned router must beat:** a Cloudflare-style rolling robust z-score (median+MAD, current 5-min vs trailing 4–6h window, fire on |z|>3.5 **AND** an absolute floor).

---

## 4. L2 — Online routing under drift & system load: metrics

Evaluated **prequentially** (interleaved test-then-train) on the replayed stream, **not** a static split.

**Prequential accuracy with fading factor.** Each event is first tested, then trained on. Use the memoryless fading factor: `S_i = x_i + α·S_{i−1}`, `B_i = 1 + α·B_{i−1}`, error `E_α(i) = S_i/B_i`, with α ∈ {0.99, 0.999} stated explicitly. Because FAST/BATCH is imbalanced and bursts are autocorrelated, **report prequential AUC** (imbalance-invariant, doubles as a drift indicator) and **Kappa-temporal** = `(p0 − p'_per)/(1 − p'_per)` against the persistent/no-change classifier — a value ≤ 0 means the router is no better than repeating the last label. Drop bare accuracy as a headline.

**Cumulative regret vs oracle routing.** `regret(T) = Σ_t [ r*(x_t) − r_chosen(x_t) ]`, where the oracle action is reconstructed from logged "appropriate" verdicts. Plot regret vs T against baselines (always-FAST, always-BATCH, rules-only, random, ε-greedy); expect sublinear growth in stationary segments and a recoverable kink at injected drift. State and own the oracle-bias from delayed verdicts.

**Off-policy / counterfactual evaluation.** For offline router comparison on logged data, use the **Li et al. replay (rejection-sampling) evaluator** — unbiased when logging was uniform; cost ≈ K·T retained events (K=2 arms). Because the real logging policy (rules + bandit) is **non-uniform**, also report the **Doubly-Robust (DR)** estimator (consistent if *either* propensity *or* reward model is correct, lower variance than IPS) with IPS as a sanity baseline. **Log per-decision action propensities at serving time now** so this is feasible later.

**Calibration.** Validate the "calibrated `p(urgent)`/score" claim with **ECE** = `Σ_m (|B_m|/N)·|acc(B_m) − conf(B_m)|` (fixed M bins, e.g. 15), a **reliability diagram**, and the **Brier score** (proper scoring rule; ECE is not). Track all three **prequentially** so post-drift miscalibration and online-recalibration recovery are visible.

**Drift detection delay & recovery.** Inject known drift points; measure **detection delay** (instances from true onset to ADWIN/DDM alarm), **false-alarm rate** on stationary windows, and **recovery time** (instances until prequential AUC returns within tolerance of its pre-drift level).

**System metrics (the same event log, open-loop load).** Avoid **coordinated omission**: the replayer injects events on a fixed schedule (wrk2/HdrHistogram methodology), stamps each with its intended arrival time, runs on hardware separate from the system under test, and defines `latency = delivery_time − intended_arrival_time`.
- **End-to-end event-time latency-to-agent**, p50/p95/**p99/p99.9**/max (never just the mean), where `t0 =` CloudEvent event-time arrival and `t_deliver =` commit of the (event, Context Unit) tuple to the entity-partitioned delivery topic such that it is consumable; **decomposed** into ingest+adapter / route / context-assembly (Graphiti+FalkorDB) / delivery. Report FAST-lane and BATCH-lane separately.
- **Sustainable throughput** (Karimov): sweep input rate, report the *knee* where p99 event-time latency stops being flat — not a peak-firehose number.
- **Consumer/watermark lag:** Kafka `records-lag-max`; Flink event-time (watermark) lag.
- **Backpressure / saturation:** Flink `backPressuredTimeMsPerSecond`, `busyTimeMsPerSecond` (target <700/1000 at peak for headroom).
- **Checkpoint & recovery:** `LastCheckpointDuration`; **recovery time** = wall-clock from an induced TaskManager kill until the job re-reaches its pre-failure watermark/lag (validates the Kappa+ replay claim).

---

## 5. L3 — End-to-end agent task utility: metrics

**Timing prediction + timeliness mapped to the verdict vocabulary.** Build the 2×2 confusion over {true-need, no-need} × {intervened, silent}: Correct-Detection = **appropriate**, False-Detection = **irrelevant**, Missed-Need = **too_slow** (omission). Report Intervention **Precision** (= alert-fatigue proxy), **Recall** (= manual-seek-help proxy), **F1**, plus an over-trigger/False-Discovery rate — matching ProAgentBench/ProactiveBench. **Do not collapse timing into point-wise F1.** For each true need, compute a **signed lead time** `Δt = t_intervene − t_optimal` within a pre-registered tolerance window `[−τ_early, +τ_late]`: inside = appropriate; before `−τ_early` = **too_urgent**; after `+τ_late` (or never) = **too_slow**. Report (a) **median signed lead time + IQR**, (b) a latency-aware event-F1 (NAB sigmoid / PATE proximity weighting), and (c) the full **verdict histogram** as the primary diagnostic. **Asymmetric reward:** a cost matrix grounded in HCI interruption-cost research — reward *appropriate*; penalize *too_urgent* by interruption/resumption cost (Adamczyk & Bailey, CHI 2004); penalize *too_slow* by decayed value-of-information (Horvitz bounded deferral); penalize *irrelevant* most (interruption cost for zero info). Report **mean asymmetric reward per event** (the pre-registered primary L3 metric, also what the bandit/controller optimizes) and keep the raw histogram so reviewers can re-weight.

**Assist-content quality via LLM-as-judge (bias-controlled).** Score **Helpfulness** and **Relevance** via **pairwise** judging with **randomized/swapped order** (count a win only if stable under both orders, cancelling position bias), and **Faithfulness/Groundedness** via RAGAS-style entailment against the Context Units the agent received (catches hallucination). Controls: judge from a **different model family** than the generator, temperature 0, rubric + chain-of-thought, reference-guided where a gold assist exists. **Human spot-check:** label a stratified subset (oversampling boundary/too_urgent/too_slow), report **Cohen's κ** (judge-vs-human and human-vs-human); trust the judge only on slices where κ ≥ ~0.6. Frontier judges fail 50%+ of bias stress tests, so this validation is mandatory.

**Token-cost / "token waste."** Count `prompt_tokens` and `generation_tokens` per event (vLLM exposes `vllm:prompt_tokens_total` / `vllm:generation_tokens_total`). Define **token waste counterfactually**: for each event routed to FAST whose verdict ∈ {too_urgent, irrelevant}, `waste = tokens_charged_on_fast_path − its amortized share under the BATCH-aggregated policy`. Aggregate to `token_waste_rate = wasted_tokens / total_tokens` and `wasted_$` under a **frozen cost model** ($/M input + $/M output, model tier, fixed per experiment). Report **input-token waste separately** — assembled-context input tokens dominate mis-routing cost. This converts the noisy "true urgency" label problem into a measurable economic quantity. Pin the BATCH aggregation policy per experiment (the amortized share depends on it).

**Head-to-head delivery benchmark (pull/poll vs push).** Fairness controls: feed **both arms the identical trace at the identical CO-corrected rate**, with identical agent, broker, partition count, and hardware; **sweep the poll interval** (e.g. 1s / 5s / 30s) and report the **latency/cost/staleness Pareto** — a single interval is cherry-picking. Attribute the poll arm's extra latency explicitly to interval-induced staleness vs empty-poll waste, and the push arm's cost to delivery/connection overhead. Report per-arm p50/p95/**p99/p99.9** latency-to-agent plus throughput and cost. (This is the RTCE-pull vs Kafka-push comparison from `proposal.md` RQ4.)

---

## 6. Experiment matrix

Conditions run on the **same replayed event stream** (everything paired), ≥3 seeds each.

| # | Experiment (question) | Condition / ablation | Key metrics | Baseline / comparator |
|---|---|---|---|---|
| E1 | Routing quality (H1) | C1 full engine; C2 rules-only; C3 +anomaly (no bandit); C4 +bandit; **C5 global supervised `p(urgent)`** | VUS-PR, affiliation-F1, PR-AUC, NAB(`low_FN`), recall@FPR≤1%, median delay | Rolling robust z-score floor (\|z\|>3.5 + abs floor); random scorer (sanity) |
| E2 | Online drift (H1) | C1 vs C4 vs C5 across injected drift points | Prequential AUC, Kappa-temporal, cumulative regret vs oracle, DR off-policy estimate, drift detection delay / recovery | always-FAST, always-BATCH, ε-greedy |
| E3 | Calibration | C3, C4, C5 | ECE (M=15), reliability diagram, Brier — prequential, pre/post drift | Uncalibrated raw scores |
| E4 | **Burstiness ablation (H2)** | C1 vs C2, on trace at measured **B** and **flattened B→0** (same event set, Poisson-flat arrivals, Fano→1, per-entity counts fixed) | mean asymmetric reward, timing-F1, over-trigger rate — and **C1−no-engine gap as fn(B)** | No-engine (raw events to agent) |
| E5 | SLA & system (H3) | C1 FAST vs BATCH lanes, swept load | event-time latency p50–p99.9, sustainable throughput, records-lag-max, backpressure, recovery time | Single undifferentiated lane |
| E6 | Economics (H3) | C1 vs always-FAST vs always-BATCH | token_waste_rate, wasted_$, input-token waste, $/1000 events (frozen cost model) | always-FAST upper bound |
| E7 | Agent utility | C1 vs C2 vs no-engine | Intervention P/R/F1, verdict histogram, mean reward, helpfulness win-rate, faithfulness (κ-validated) | ProactiveBench reward-model pattern |
| E8 | **Delivery (H4)** | push vs poll @ {1s,5s,30s} | latency p50–p99.9, staleness, cost (empty-poll waste) Pareto | poll baseline (RTCE-style) |

---

## 7. Statistical rigor

Because all conditions run on the **same stream**, every comparison is **paired**.
- **Binary timing decision (intervene vs silent):** **McNemar's test** on the paired discordant cells.
- **Continuous metrics** (reward, signed lead time, VUS-PR, faithfulness, latency): **paired BCa bootstrap 95% CIs** + a **sign-flip permutation test** on per-seed deltas, across **≥3 seeds**.
- **Multiple comparisons:** when reporting several conditions/metrics jointly, apply Holm–Bonferroni correction and pre-register the **primary metric** (L1: VUS-PR; L3: mean asymmetric reward) to avoid metric-shopping.
- **"Significantly better"** = the **paired difference CI excludes zero** **AND** the effect is practically meaningful (report the delta and its CI, never bare "SOTA").
- **Label-noise robustness:** jitter ground-truth timestamps and flip a fraction of verdicts; show VUS-PR/affiliation metrics remain stable (they already give near-in-time partial credit) — this directly defends the labels Achilles heel.

---

## 8. References

1. Tang et al. *ProAgentBench: Evaluating LLM Agents for Proactive Assistance with Real-World Data.* arXiv:2602.04482. https://arxiv.org/abs/2602.04482
2. Lu et al. *Proactive Agent: Shifting LLM Agents from Reactive to Active.* arXiv:2410.12361. https://arxiv.org/abs/2410.12361 — repo: https://github.com/thunlp/ProactiveAgent
3. Wu et al. *LongMemEval.* arXiv:2410.10813. https://github.com/xiaowu0162/LongMemEval
4. *LoCoMo: Very Long-Term Conversational Memory.* https://snap-research.github.io/locomo/
5. Kim & Jo. *Measuring burstiness for finite event sequences.* arXiv:1604.01125. https://arxiv.org/pdf/1604.01125
6. `bursty_dynamics` package. https://github.com/ai-multiply/bursty_dynamics
7. *ROC vs Precision-Recall for Imbalanced Classification.* https://machinelearningmastery.com/roc-curves-and-precision-recall-curves-for-imbalanced-classification/
8. Tatbul et al. *Precision and Recall for Time Series.* NeurIPS 2018. https://arxiv.org/abs/1803.03639
9. Huet, Navarro & Rossi. *Local Evaluation of Time Series Anomaly Detection Algorithms* (affiliation metrics). KDD 2022. https://arxiv.org/abs/2206.13167 — impl: https://github.com/ahstat/affiliation-metrics-py
10. Paparrizos et al. *Volume Under the Surface (VUS).* PVLDB 15(11), 2022. https://www.vldb.org/pvldb/vol15/p2774-paparrizos.pdf — impl: https://github.com/TheDatumOrg/VUS
11. Lavin & Ahmad. *Evaluating Real-time Anomaly Detection — the Numenta Anomaly Benchmark.* IEEE ICMLA 2015. arXiv:1510.03336. https://arxiv.org/abs/1510.03336 — profiles: https://github.com/numenta/NAB/blob/master/config/profiles.json
12. *nexmark/nexmark.* https://github.com/nexmark/nexmark
13. *The OpenMessaging Benchmark Framework.* https://openmessaging.cloud/docs/benchmarks/
14. Karimov et al. *Benchmarking Distributed Stream Data Processing Systems.* arXiv:1802.08496. https://arxiv.org/abs/1802.08496
15. Kim et al. *Towards a Rigorous Evaluation of Time-Series Anomaly Detection.* AAAI 2022. arXiv:2109.05257. https://arxiv.org/abs/2109.05257 — PA%K impl: https://github.com/tuslkkk/tadpak
16. Cloudflare. *Introducing thresholds in Security Event Alerting: a z-score love story.* https://blog.cloudflare.com/introducing-thresholds-in-security-event-alerting-a-z-score-love-story/ — and *Lessons Learned from Scaling Up Cloudflare's Anomaly Detection Platform.* https://blog.cloudflare.com/lessons-learned-from-scaling-up-cloudflare-anomaly-detection-platform/
17. Gama, Sebastião & Rodrigues. *On evaluating stream learning algorithms.* Machine Learning, 2013. https://link.springer.com/article/10.1007/s10994-012-5320-9
18. Brzeziński & Stefanowski. *Prequential AUC.* KAIS 2017. https://link.springer.com/article/10.1007/s10115-017-1022-8
19. Bifet et al. *Evaluation methods and decision theory for classification of streaming data with temporal dependence.* Machine Learning, 2014. https://link.springer.com/article/10.1007/s10994-014-5441-4
20. Li, Chu, Langford & Wang. *Unbiased Offline Evaluation of Contextual-bandit-based News Article Recommendation Algorithms.* WSDM 2011. https://arxiv.org/abs/1003.5956
21. Dudík, Langford & Li. *Doubly Robust Policy Evaluation and Learning.* https://arxiv.org/abs/1612.01205
22. Guo et al. *On Calibration of Modern Neural Networks.* ICML 2017. https://arxiv.org/abs/1706.04599
23. Bifet & Gavaldà. *Learning from Time-Changing Data with Adaptive Windowing (ADWIN).* SDM 2007. https://www.cs.upc.edu/~abifet/PAKDD2009.pdf
24. Tene. *wrk2 / Coordinated Omission.* https://github.com/giltene/wrk2
25. ScyllaDB. *On Coordinated Omission.* https://www.scylladb.com/2021/04/22/on-coordinated-omission/
26. *Apache Flink Metrics.* https://nightlies.apache.org/flink/flink-docs-master/docs/ops/metrics/
27. Adamczyk & Bailey. *If not now, when? The effects of interruption at different moments within task execution.* CHI 2004.
28. Horvitz et al. *Principles of Bounded Deferral.*
29. Zheng et al. *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.* https://arxiv.org/pdf/2306.05685
30. RAGAS metrics (faithfulness / answer relevancy / context precision). https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/
31. *Judge's Verdict: LLM Judge Capability Through Human Agreement.* arXiv:2510.09738. https://arxiv.org/html/2510.09738v1
32. *vLLM Metrics.* https://docs.vllm.ai/en/latest/design/metrics/
33. *Comparing a polling and push-based approach for live Open Data interfaces.* https://brechtvdv.github.io/Article-Live-Open-Data-Interfaces/
34. *When +1% Is Not Enough: A Paired Bootstrap Protocol for Evaluating Small Improvements.* arXiv:2511.19794. https://arxiv.org/html/2511.19794v1
