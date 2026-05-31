# RQ2 Reframed: From Supervised Urgency Classification to Per-Entity Streaming Anomaly Detection

*Research note responding to supervisor feedback: "the way we handle fast-queue/batch-queue classification is wrong — look at how Cloudflare does anomaly detection and take inspiration from it." Supersedes the supervised-cascade framing in `research/RQ2-event-classification.md` §6 and `proposal.md` "RQ2 — Proposed classification pipeline".*

---

## 1. The flaw in the current framing

The current RQ2 design routes each event to a FAST or BATCH lane via a four-stage supervised cascade ending in a *calibrated supervised classifier* emitting `p(urgent)` plus a *contextual bandit* trained on downstream agent verdicts. The design document itself names the Achilles heel — labels — but the problem is deeper than "labels are noisy." It is **structural**:

- **The target does not exist independently of the decision.** A supervised classifier needs i.i.d. `(event, urgent?)` pairs. The only label source here is downstream agent behavior `{appropriate, too_urgent, too_slow, irrelevant}`, which is (a) *delayed* (revealed only after the agent acts), (b) *selection-biased* — agents only ever see events the router already pushed FAST, so the BATCH lane is never observed (missing-not-at-random / bandit feedback), and (c) *policy-dependent* — the label distribution shifts every time the router is retrained, making the target non-stationary by construction. Training a global "urgent" class on such a signal optimizes a moving, self-referential objective [12, 13].

- **"Urgent" is contextual and per-entity, not a global learnable class.** The same raw value means different things for different entities: 50 failed logins is normal for a 500K-req/s zone and an attack for a small one; an order refund is routine for a churned account and alarming for a VIP. A global classifier cannot represent "urgent-*for-this-entity*" without per-entity features it has never seen. This is precisely the *contextual anomaly* formulation of Chandola et al. [14] — a point is anomalous only relative to its context — and the context key is exactly the thesis's CloudEvent `subject` (`customer:42`).

- **Cold-start.** A supervised classifier is blind on a brand-new Slack channel or JIRA project until enough *labeled* traffic accrues — and labels are the scarcest resource of all.

- **Drift.** A fixed decision boundary degrades under concept drift, forcing a detect-then-retrain loop (DDM/ADWIN) and a lag during which the model is simply wrong.

The honest conclusion: supervised urgency classification is the *wrong primitive*. Not because the engineering is hard, but because the learning problem is ill-posed.

---

## 2. Inspiration: how Cloudflare actually does it

Cloudflare runs the largest production anomaly-detection fleet in the industry, and — marketing stripped out — its design choices answer the label problem directly.

**Per-entity adaptive baselines, no attack labels.** Adaptive DDoS Protection "creates a traffic profile by looking at a customer's maximal rates of traffic every day, for the past seven days," recalculated daily on a rolling 7-day window, using only the 95th-percentile rate to resist short bursts [1, 2]. No "this was an attack" label is ever used: *the per-zone baseline is the ground truth.*

**Deviation SCORING, not labeled classification.** Cloudflare's flagship products emit a continuous 1–99 score and defer the binary decision to a customer-chosen threshold: `cf.waf.score` ("a global score from 1–99") and `bot_score` ("how likely that request came from a bot") are the *model output*; the cut point (`cf.waf.score le 20`) is a separate, operator-owned policy [3, 4]. Their deepest R&D detector trains **one unsupervised model per zone** (~1M/day) on ~12 features deliberately chosen to be normally distributed and *uncorrelated with traffic volume* (canonically, "the proportion of users across the top-5 browsers"), scores each 5-minute window by Mahalanobis distance after PCA-whitening, and applies a **single global threshold λ** across all million models via a final rescaling [5]. The label-free analytics platform uses Histogram-Based Outlier Scoring (HBOS) against a *per-customer* ClickHouse baseline with per-visitor Redis sliding windows, at 500K+ req/s over ~310M visitors [6].

**Volume-decoupling is the killer move.** Cloudflare *explicitly rejected* both global z-score thresholds and seasonal SARIMA models — z-scores false-positive on benign spikes (Black Friday); SARIMA needs years of history (so "new customers wouldn't be protected for multiple years" — cold start) and retraining every 10–20 minutes [5]. By baselining volume-*invariant ratios*, a legitimate surge does not look like an attack.

**Customer-tunable sensitivity.** Thresholds are exposed as a *sensitivity level* (inverse of threshold), rules deploy in *Log mode* first, and a deviation alone is insufficient — an orthogonal Bot Score signal gates the decision [1, 7]. The simpler notification products are a robust **z-score over short-vs-long windows**: 5-minute observation vs ~4-hour rolling baseline, alert when `|z| > 3.5` **AND** the absolute change exceeds a floor (≈200 requests), firing on both spikes and drops [8, 9].

**Rules + heuristics + ML, layered.** Cloudflare never replaced rules with ML. Deterministic fingerprinting (dosd/Gatebot) does fast mitigation by signal-strength weighting; the unsupervised score does *triage*; Gatebot rules override dosd [10, 11]. The ML score surfaces candidates; a deterministic layer takes the irreversible action.

**Verification caveat (authoritative):** only the *DDoS* path is confirmed to add a Bot Score; the cross-product "deviation + orthogonal ML signal everywhere" generalization is not established. We therefore claim the two-signal gate as a *DDoS-validated pattern we adopt*, not as a universal Cloudflare invariant.

---

## 3. The reframe: per-entity streaming anomaly detection as the fast-path primitive

Replace `p(urgent)` with a **per-entity deviation score** `s_anom(event | baseline_subject)`, keyed by CloudEvent `subject`, computed inside the existing Kappa+ Flink job. The toolbox gives a concrete, defensible three-tier design:

- **Tier 1 — O(1) univariate per-entity statistics (the FAST-lane workhorse).** Per `subject`, maintain a rolling **robust z-score** (median + MAD, not mean + σ, for outlier resistance) plus **Page-Hinkley / CUSUM** for change-points and EWMA/PEWMA for recency [15, 16]. Cost is constant memory and sub-microsecond per event; this is Cloudflare's actual notification math (`|z| > 3.5` AND absolute floor [8]). Use Welford's algorithm for online mean/variance in Flink keyed RocksDB state. Volume-invariant *ratios* (error-event fraction, negative-sentiment fraction, P0/severity mix, distinct-actor entropy) are baselined here, **not raw volume**, to avoid flagging benign campaign spikes.

- **Tier 2 — multivariate outlier scoring for feature combinations Tier 1 misses.** **Half-Space Trees** (online isolation forest, River; ~10 trees, height 8, window 250) as the primary, or **Robust Random Cut Forest** (Guha et al., ICML 2016; AWS Kinesis pedigree, 51.7 on NAB) as the heavier, more principled alternative [17, 18]. One small forest per hot entity. Cloudflare's HBOS (linear-time histogram scoring) is the cheapest multivariate option and our default when feature count is small [6].

- **Drift detectors (ADWIN/KSWIN) govern the baseline lifecycle, never the trigger.** A drift event *resets* an entity's baseline; it does not itself fire the FAST lane. Scorer governs routing, drift governs adaptation [19].

**Where it lives.** Flink keys by `subject`; per-entity rolling statistics sit in keyed RocksDB state next to — and are persisted into — the per-entity Graphiti temporal knowledge graph, so "what is normal for `customer:42` over time" is co-located with the entity's existing memory. Iceberg/Paimon replay lets baselines be *deterministically recomputed* when the statistic changes — a Kappa advantage a stateful supervised model cannot match [21, 22]. Defensible default cadence, borrowed from Cloudflare: 5-minute buckets, rolling ~4-week / 7-day window, retrain daily ("very little intraday drift") [5].

---

## 4. The honest synthesis (hybrid)

**Anomaly ≠ urgency, in both directions** — this must headline the chapter, not be buried:

- *Urgent but not anomalous:* a JIRA P0 or a 5xx burst on a service that 5xxes daily is high-importance yet statistically expected; a pure anomaly detector firing on rarity *misses* it.
- *Anomalous but not urgent:* a brand-new low-traffic channel's first message is novel yet trivial; novelty ≠ importance, so a pure detector *false-alarms* [23, 24, 14].

Therefore the answer is **not** "replace the classifier with an anomaly detector." It is a hybrid trigger where each term covers the other's failure direction:

```
route_to_fast =  Rules(e)  OR  [ s_anom(e | baseline_subject) > θ_source ]
```

`Rules(e)` is the existing JsonLogic/Drools layer (JIRA P0, Slack @oncall), now re-cast from "short-circuit for obvious cases" to the **mandatory floor** covering the urgent-but-expected class the detector provably misses. `s_anom` covers urgent-because-deviant. This mirrors production AIOps exactly — "ML-based but with rule-engine support" [25] — and Cloudflare itself layers ML scoring *on top of* deterministic signals rather than replacing them [10].

**Repurpose the bandit — keep the feedback loop, change its job.** Its original task (learn `p(urgent)` from verdicts) is unsalvageable for the structural reasons in §1. Its *new* task is well-posed and one-dimensional: **tune the per-source sensitivity scalar `θ_source`**, exactly Cloudflare's customer sensitivity levels [7, 9]. `too_urgent` verdicts → raise `θ` (less sensitive); `too_slow` → lower `θ`. Verdicts thus become a *slow, noisy control signal* (drift correction), not per-event ground truth. Firing uses the Google SRE **multi-window, multi-burn-rate** scheme — a short (5-min) AND a long (~1–4h) window must both cross — to suppress flapping [26]. Tuning one scalar per source from delayed feedback is a far better-posed control problem than learning a classifier from the same signal; a simple per-source EWMA/PID controller may be more defensible to a committee than full LinUCB.

This keeps everything good about the old design — CloudEvents adapters, the rules layer, the feedback channel, the entity-keyed delivery — while removing the load-bearing component (`p(urgent)`) that depended on labels that do not exist. The supervised classifier is *demoted, not deleted*: it survives as an optional offline auditor, pre-empting the "why not just use a classifier?" objection.

---

## 5. Before / after

| Stage | OLD: supervised cascade | NEW: rules-OR-anomaly hybrid |
|---|---|---|
| 1. Ingest | CloudEvents envelope + per-source adapter | **Unchanged** |
| 2. Rules | JsonLogic/Drools short-circuit for "obvious" cases | **Unchanged mechanism; re-cast as the mandatory floor** for urgent-but-expected events |
| 3. Core scorer | Global calibrated `p(urgent)` (LightGBM/DistilBERT) from labels | **Per-entity `s_anom` vs Flink keyed-state baseline** (robust-z/Page-Hinkley → Half-Space Trees/HBOS), keyed by `subject`, persisted in Graphiti; hierarchical cold-start fallback global→source-type→entity |
| 4. Decision | Contextual bandit *learns urgency* from verdicts | **OR-gate**: `Rules(e) OR s_anom > θ_source`, with multi-window/multi-burn confirmation |
| 5. Feedback | Verdicts = ground-truth training labels | Verdicts = **slow control signal tuning `θ_source`** (Cloudflare sensitivity); classifier demoted to offline auditor |
| Failure modes | No labels; selection bias; non-stationary self-referential target; cold-start blindness; drift forces retrain | Anomaly blind-spot (→ rules floor); novelty floods (→ floor + multi-window + θ); state explosion (→ hot-entity LRU); baseline poisoning (→ robust stats + rules floor) |

**Architecture diagram / data-flow changes.** Replace the "calibrated classifier → bandit (learns urgency)" box with "per-entity baseline (Flink keyed state + Graphiti) → anomaly score → OR-gate with rules → `θ_source` controller fed by verdicts." Cloudflare's pipeline (Kafka → ClickHouse baseline → Redis hot state, three services) is structurally identical to the thesis's Kappa+ stack (Kafka + Iceberg/Paimon → single Flink graph → Graphiti/FalkorDB) [6], so no new system is introduced — only the contents of stage 3–4 change. Add a side-by-side BEFORE/AFTER figure.

---

## 6. New failure modes & mitigations

- **Anomalous-but-unimportant novelty floods.** A new low-volume entity or benign new event type scores high on novelty. *Mitigations:* the absolute-volume floor (Cloudflare's "≥200" [8]); multi-window confirmation (short AND long must cross [26]); and `θ_source` tuning that raises the threshold for sources generating many `too_urgent` verdicts.

- **Baseline poisoning / slow-burn ramp.** A gradual malicious or pathological ramp trains the rolling baseline to accept bad behavior so it never fires. *Mitigations:* robust statistics (MAD, capped EWMA) bound how fast the baseline moves; the deterministic rules layer is a hard floor independent of the baseline; Iceberg/Paimon replay enables retrospective detection of baseline tampering.

- **Seasonality / benign spikes (Black Friday).** *Mitigations:* baseline volume-*invariant ratios* not raw volume (Cloudflare's central design choice [5]); for strongly seasonal sources, upgrade Tier 1 to Cloudflare Radar's label-free kNN-median forecast — bucket into 15-min blocks, forecast as the median of the 6 most-similar prior 24h windows over ~28 days, with a magnitude filter requiring the deviation to exceed ~10% of the (P95−P5) spread and a hysteresis clean-period before resetting [20]. Note (authoritative): do **not** claim Cloudflare uses Holt-Winters — the disclosed method is kNN-median matching.

- **Cold-start.** Hierarchical fallback (global → source-type → entity): a new entity inherits the group baseline on day one (and rules still fire), graduating to its own baseline online — the MNM recipe: conservative absolute floor for ~14–30 days, then switch to rolling robust-z once variance stabilizes [9].

- **Per-entity state explosion.** Hot-entity-only keyed state with LRU eviction and group fallback; Cloudflare's "recency register" cache gave a 10x throughput win doing exactly this [6].

- **Low-volume entities break Gaussian/Mahalanobis baselines.** For a JIRA project emitting a few events/hour, dense 5-min Gaussian buckets are ill-defined; switch such sources to event-count/Bayesian baselines or rules-dominant routing (open question to validate empirically).

---

## 7. Evaluation pointer

Because true urgency has no clean label, the fast-path is evaluated as a **streaming anomaly-detection** problem on a **replayed bursty stream** (prequential, never a static split), with headline metrics **VUS-PR** and **affiliation-F1**, a NAB latency score under the `reward_low_FN_rate` profile, and an explicit prohibition on naive point-adjusted F1. The full four-layer metric framework, experiment matrix, and statistical-rigor protocol are in `research/evaluation-methodology.md`.

---

## References

1. Cloudflare — Introducing Adaptive DDoS Protection. https://blog.cloudflare.com/adaptive-ddos-protection/
2. Cloudflare Docs — Adaptive DDoS Protection. https://developers.cloudflare.com/ddos-protection/managed-rulesets/adaptive-protection/
3. Cloudflare Docs — WAF Attack Score. https://developers.cloudflare.com/waf/detections/attack-score/
4. Cloudflare Docs — Bot scores. https://developers.cloudflare.com/bots/concepts/bot-score/
5. Cloudflare — Training a million models per day to save customers of all sizes from DDoS. https://blog.cloudflare.com/training-a-million-models-per-day-to-save-customers-of-all-sizes-from-ddos/
6. Cloudflare — Lessons Learned from Scaling Up Cloudflare's Anomaly Detection Platform. https://blog.cloudflare.com/lessons-learned-from-scaling-up-cloudflare-anomaly-detection-platform/
7. Cloudflare — How to customize your layer 3/4 DDoS protection settings. https://blog.cloudflare.com/l34-ddos-managed-rules/
8. Cloudflare — Introducing notifications for HTTP Traffic Anomalies. https://blog.cloudflare.com/introducing-http-traffic-anomalies-notifications/
9. Cloudflare Docs — Magic Network Monitoring dynamic threshold. https://developers.cloudflare.com/magic-network-monitoring/rules/dynamic-threshold/
10. Cloudflare Docs — Bot detection engines. https://developers.cloudflare.com/bots/concepts/bot-detection-engines/
11. Cloudflare — A deep-dive into Cloudflare's autonomous edge DDoS protection. https://blog.cloudflare.com/deep-dive-cloudflare-autonomous-edge-ddos-protection/
12. From Zero to Hero: Cold-Start Anomaly Detection. https://arxiv.org/abs/2405.20341
13. Cloudflare — Get notified about the most relevant events with Advanced HTTP Alerts. https://blog.cloudflare.com/custom-alert-features-anomaly-detection/
14. Chandola, Banerjee, Kumar — Anomaly Detection: A Survey (ACM CSUR 2009). https://dl.acm.org/doi/10.1145/1541880.1541882
15. River — PageHinkley. https://riverml.xyz/dev/api/drift/PageHinkley/
16. River — anomaly module / robust statistics. https://riverml.xyz/dev/api/anomaly/HalfSpaceTrees/
17. River — HalfSpaceTrees. https://riverml.xyz/dev/api/anomaly/HalfSpaceTrees/
18. Guha et al. — Robust Random Cut Forest (ICML 2016) / AWS Kinesis. https://proceedings.mlr.press/v48/guha16.html
19. River — ADWIN. https://riverml.xyz/dev/api/drift/ADWIN/
20. Cloudflare — Gone offline: how Cloudflare Radar detects Internet outages. https://blog.cloudflare.com/detecting-internet-outages/
21. Confluent Current 2023 — Anomaly Detection on Time Series Data Using Apache Flink. https://www.confluent.io/events/current/2023/anomaly-detection-on-time-series-data-using-apache-flink/
22. Streamkap — Anomaly Detection in Streaming Data with Flink (keyed state, Welford, z-score). https://streamkap.com/resources-and-guides/flink-anomaly-detection
23. Baeldung — Drift, Anomaly, and Novelty in Machine Learning. https://www.baeldung.com/cs/ml-drift-anomaly-novelty
24. scikit-learn — Novelty and Outlier Detection. https://scikit-learn.org/stable/modules/outlier_detection.html
25. Ho et al. — A Survey of Time Series Anomaly Detection Methods in the AIOps Domain (2023). https://arxiv.org/abs/2308.00393
26. Google SRE Workbook — Alerting on SLOs (multi-window, multi-burn-rate). https://sre.google/workbook/alerting-on-slos/
