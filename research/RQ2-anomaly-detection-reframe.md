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

Replace `p(urgent)` with a **per-entity deviation score** `s_anom(event | baseline_subject)`, keyed by CloudEvent `subject`, computed inside the existing Kappa+ Flink job.

### 3.1 The one structural difference from Cloudflare — and how to absorb it

Cloudflare's events are **homogeneous**: every event is an HTTP request sharing one schema (browser, ASN, path, status), so a single per-zone feature vector works. The CMS ingests **heterogeneous** events — a Slack message, a JIRA transition, and a checkout event share no fields — so there cannot be one global feature vector. The design is therefore **two-level**:

- **Entity = the "zone".** The CloudEvent `subject` (`customer:42`, `service:checkout`, `ticket:PROJ-1`) keys the baseline, exactly as Cloudflare keys per zone. This is *already* the Kafka partition key and the Flink `keyBy` — so per-entity anomaly state shards along the boundary the architecture already scales on (see §4.5).
- **Feature schema = per source-type.** One detector per *source-type* (Slack / JIRA / telemetry), its baseline statistics keyed per `subject`. Adding a source means writing a feature extractor + (optional) rules — never retraining a global model.

### 3.2 What to baseline: volume-decoupled features in two layers

Cloudflare's golden rule is the one to copy: **baseline composition/ratio features, not raw counts** — their canonical feature is *"the proportion of users across the top-5 browsers"* [5], chosen precisely because it stays stationary when volume spikes legitimately (flash sale, viral thread). Mirror it in two layers.

**Layer A — source-agnostic envelope/arrival features** (computable for *every* event from the envelope + the per-entity arrival process):

| Feature | Cloudflare analogue | Volume-decoupled |
|---|---|---|
| Inter-arrival gap vs entity baseline | burst signal | ✓ (ratio) |
| Event-type **mix** in rolling window (proportions) | top-5 browser proportion | ✓ |
| Actor/source **diversity** (entropy of distinct actors) | client-IP entropy | ✓ |
| Novelty flags (first-ever event-type / actor / subject) | new ASN | ✓ |
| Hour-of-day / day-of-week | seasonality covariate | ✓ |
| Raw rate in window | request rate | ✗ — kept only behind the absolute-volume guard |

**Layer B — source-specific content features** (per adapter):

- **Slack:** message length, @mention count, `@here`/`@channel`, negativity/sentiment (small model), `?` present, thread-reply velocity, reaction velocity, off-hours flag, embedding distance from the channel's topic centroid.
- **JIRA:** priority (ordinal), issue type, status-transition type (→ `blocked`/`reopened` weighted high), comment-burst rate, assignee change, watcher delta, time-to-SLA-breach.
- **E-commerce / telemetry:** order value **z-scored against the customer's own normal**, refund/dispute amount, payment-failure flag, error-event fraction in window, latency-p95 breach, status-code mix shift, funnel-sequence anomaly.

**Feed deviations, not raw values.** Each feature enters the detector already transformed *relative to the entity's own baseline*: numeric → per-entity robust z `(x − median)/MAD` (or straight into a per-entity histogram); compositional → Jensen-Shannon / KL divergence of the current window's mix from the entity baseline mix. The per-entity keying of those histograms/baselines *is* what makes "urgent" contextual rather than global.

### 3.3 The scorer: one model — HBOS over per-entity-normalized features

Commit to a **single scorer**. The earlier "HBOS + robust-z + Mahalanobis + Page-Hinkley" stack was over-built; collapse it:

- **HBOS — Histogram-Based Outlier Score** [27] is **the** scorer. One univariate histogram per feature; the score is `HBOS(p) = Σ_{i=1}^{d} log(1 / hist_i(p))` — a discrete Naive-Bayes density (log-sum for float stability). It **assumes feature independence → linear time**, handles **mixed categorical + numeric** natively (counting for categoricals), uses **dynamic (equal-area) bins** for long-tailed event features, and normalizes each histogram to max-height 1.0 so features weight equally (`k ≈ √N` bins). Crucially it needs **no per-entity covariance matrix** — which is why Cloudflare runs it over 310M visitors in its analytics platform [6], and why it scales to millions of entities.
- **Robust-z is the per-entity *normalizer*, not a second detector.** Each numeric feature enters HBOS already transformed to a per-entity deviation `(x − median)/MAD` (running median + MAD via Welford / P²-quantiles in keyed state). A per-entity point anomaly — a single 10×-normal refund — then lands in a *sparse* HBOS bin → high score. HBOS thus catches "local" deviations *because* the feature is per-entity-normalized, so **no parallel univariate detector is required**; this also closes HBOS's documented blind spot (it misses local outliers only on *raw* features).
- **Deferred upgrades, not v1.** **PCA + Mahalanobis** [5] (captures feature correlations, single global threshold λ via whitening) reintroduces a per-entity covariance matrix — the very thing HBOS exists to avoid, and the basis of the scaling story — so it is an explicit *future* upgrade, used only if evaluation shows correlated-feature anomalies are missed. **Page-Hinkley / CUSUM** change-point detection [15, 16] is likewise deferred unless slow ramps are shown to slip through.
- **Drift detectors (ADWIN/KSWIN)** [19] govern the **baseline lifecycle, never the trigger**: a drift alarm *resets* an entity's baseline; it does not itself fire FAST. Scorer routes; drift adapts.

### 3.4 Where it lives and how often it refreshes

**Routing needs no context store.** Flink keys by `subject`; the per-entity normalizer state (running median/MAD, counts) and the HBOS histograms sit in **plain keyed RocksDB state** — a few scalars and small histograms per hot entity, nothing more. The custom bitemporal graph + RAG store is the *context-assembly* memory layer on the read side (RQ5); the fast/batch **decision never touches it**, so the routing path carries none of that store's ingest cost. Iceberg/Paimon replay lets baselines be **deterministically recomputed** when a feature or statistic changes — a Kappa advantage a stateful supervised model cannot match [21, 22]. Default cadence, borrowed from Cloudflare [5]: **5-minute buckets, ~4-week rolling baseline, refresh daily** ("very little intraday drift").

---

## 4. The honest synthesis (hybrid) and the routing rule

**Anomaly ≠ urgency, in both directions** — this headlines the chapter:

- *Urgent but not anomalous:* a JIRA P0, or a 5xx burst on a service that 5xxes daily, is high-importance yet statistically *expected*; a pure detector firing on rarity **misses** it.
- *Anomalous but not urgent:* a new low-traffic channel's first message is novel yet trivial; novelty ≠ importance, so a pure detector **false-alarms** [23, 24, 14].

So the answer is not "anomaly → FAST." It is a **layered rule** where rules cover the detector's blind spot and guards cover its over-firing:

```
for event e on entity E (source-type S):
  if   Rules_S(e):                       -> FAST          # known-critical (P0, @oncall) + rule id
  elif point_score(e | base_E) > θ_pt:   -> FAST          # this event is individually deviant
  elif window_anomaly(E) past guards:    -> FAST (once)   # situation envelope, deduped per window
  else:                                  -> BATCH
        # guards = absolute-volume floor (Cloudflare "≥200") AND multi-window confirmation
        #          (short 5-min AND long ~1–4h must both cross — Google SRE multi-burn-rate [26])
        # θ_pt, θ_window are per-source sensitivity SETTINGS — static (operator-set, Cloudflare-style)
        #          or a self-calibrating score quantile; there is NO online feedback loop (see below)
```

i.e. `route_to_fast = Rules(e) OR ( s_anom(e | baseline_subject) > θ_source , past floor + multi-window )`. `Rules(e)` is the existing JsonLogic/Drools layer, re-cast from "short-circuit for obvious cases" to the **mandatory floor** for the urgent-but-expected class the detector provably misses; `s_anom` covers urgent-because-deviant. This mirrors production AIOps — "ML-based but with rule-engine support" [25] — and Cloudflare itself layers ML scoring *on top of* deterministic signals rather than replacing them [10].

**Granularity — the subtle part.** Cloudflare detects anomalies at the **window** level ("is this 5-min window of zone X abnormal?"); the CMS routes **per event**. When a *rate/window* anomaly fires, do **not** flood the agent with 500 raw events — the anomaly is a **situation**: emit a single FAST **situation envelope** (the anomalous window as a Context Unit + the most-deviant triggering events), deduped per window. A *point/content* anomaly sends that one event. The continuous score (Cloudflare-style 1–99) is **reused** beyond the binary gate — it also ranks salience *within* the batch lane and weights the Context Unit.

**No online feedback loop — the threshold is a setting, not a learned quantity.** This is the key simplification over the earlier draft. Cloudflare has *no* closed loop in which a downstream consumer reports "that detection was wrong"; sensitivity is an operator-set knob or a globally-calibrated cut on the score distribution [7, 9]. The CMS does the same: `θ_source` is a **static per-source sensitivity** (low/med/high), optionally a **self-calibrating quantile** of the per-entity score (flag the top X%, or a fixed `|z| > 3.5`-style cut) that drifts with the distribution **without labels**. This **deletes** the `cms.feedback.v1` control topic, the controller, and the per-agent instrumentation burden — the expensive, slow, agent-coupled part of the previous design. Adaptivity now comes from the rolling baseline + the quantile threshold + ADWIN baseline resets, not from a delayed loop. Agent verdicts `{appropriate, too_urgent, too_slow, irrelevant}` may still be collected, but **only for offline evaluation** of routing quality (L1/L3 metrics) — never on the control path. The supervised classifier is gone from the system entirely, surviving only as an optional offline auditor.

### 4.5 Why this scales — and why the supervised cascade did not

This is the decisive axis (and the one the supervisor pressed):

| | OLD: global supervised classifier | NEW: per-entity anomaly detection |
|---|---|---|
| **Partitioning** | One global model — a central component every event must traverse; cannot be sharded by entity | State keyed by `subject` = **the existing Kafka partition key / Flink `keyBy`**; shards along the axis the system already scales on |
| **Adding a source** | Re-collect labels, retrain, re-validate the global model | Drop in a feature extractor (+ optional rules); other sources untouched |
| **Label pipeline** | Needs labels at the event rate (LLM-oracle sampling cost grows with volume) | **Label-free**; baselines self-maintain from the stream |
| **Cold start** | Blind on a new entity until labelled traffic accrues | Hierarchical fallback global → source-type → entity; rules fire from day one |
| **Cost per entity** | Amortized, but the model is a single bottleneck and single point of staleness | HBOS is linear-time, no covariance matrix — Cloudflare runs it over **310M** entities [6]; ~1M models retrained daily for DDoS [5] |
| **Drift** | Detect-then-retrain lag during which the global boundary is wrong | Per-entity baselines move with the entity; ADWIN resets only the affected key |

The supervised cascade is a *centralized* learner bolted onto a *partitioned* streaming architecture — an impedance mismatch that surfaces precisely at scale. Per-entity anomaly detection is **partition-native**: it is the same shape as the rest of the pipeline, which is why Cloudflare can run a million models a day on it. State growth (one baseline per hot entity) is the new cost; it is bounded by hot-entity-only keyed state with LRU eviction and the group-fallback baseline (§6), Cloudflare's "recency register" pattern [6].

---

## 5. Before / after

| Stage | OLD: supervised cascade | NEW: rules-OR-anomaly hybrid |
|---|---|---|
| 1. Ingest | CloudEvents envelope + per-source adapter | **Unchanged** |
| 2. Rules | JsonLogic/Drools short-circuit for "obvious" cases | **Unchanged mechanism; re-cast as the mandatory floor** for urgent-but-expected events |
| 3. Core scorer | Global calibrated `p(urgent)` (LightGBM/DistilBERT) from labels | **One model: HBOS** over per-entity-normalized features (robust-z = the normalizer), **plain Flink keyed state — no context store**, keyed by `subject`; cold-start fallback global→source-type→entity; Mahalanobis/Page-Hinkley deferred |
| 4. Decision | Contextual bandit *learns urgency* from verdicts | **OR-gate**: `Rules(e) OR s_anom > θ_source`, with absolute-floor + multi-window confirmation |
| 5. Feedback | Verdicts = ground-truth training labels | **No online loop.** `θ_source` = static per-source sensitivity / score quantile; verdicts (if collected) are for **offline evaluation only** |
| Failure modes | No labels; selection bias; non-stationary self-referential target; cold-start blindness; drift forces retrain | Anomaly blind-spot (→ rules floor); novelty floods (→ floor + multi-window + θ); state explosion (→ hot-entity LRU); baseline poisoning (→ robust stats + rules floor) |

**Architecture diagram / data-flow changes.** Replace the "calibrated classifier → bandit (learns urgency)" box with "per-entity baseline (plain Flink keyed state) → HBOS anomaly score → OR-gate with rules → fast/batch." Cloudflare's pipeline (Kafka → ClickHouse baseline → Redis hot state) is structurally identical to the thesis's Kappa+ stack (Kafka + Iceberg/Paimon → single Flink graph) [6], so no new system is introduced — only the contents of stage 3–4 change. Add a side-by-side BEFORE/AFTER figure.

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
27. Goldstein, M. & Dengel, A. (2012). *Histogram-based Outlier Score (HBOS): A fast Unsupervised Anomaly Detection Algorithm.* KI-2012 (Poster & Demo Track). https://www.goldiges.de/publications/HBOS-KI-2012.pdf
