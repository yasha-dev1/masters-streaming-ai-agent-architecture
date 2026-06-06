# Thesis Proposal — Open-Source Context Engine for Streaming AI Agents

## Prior art: Confluent Real-Time Context Engine (closed source)

A commercial system with this exact architectural shape already exists: Confluent's **Real-Time Context Engine (RTCE)**, currently Early Access on Confluent Cloud.

![Confluent RTCE architecture](img_1.png)

*Website/mobile/CDC events stream into Kafka, Flink jobs materialise per-entity context views (CUSTOMER360, ORDERS), and an LLM agent retrieves them at inference time.* RTCE is **proprietary**, Early Access, and its agent interface is **pull-only** over MCP — the agent must ask; it is never notified. Tableflow (the Iceberg/Delta materialisation layer) is likewise proprietary.

## Problem statement

> **AI agents operating over high-volume heterogeneous event streams (Slack, JIRA, e-commerce telemetry, …) need a context management system that delivers the right context at the right time, with business-critical events pushed immediately — and no open-source reference architecture or protocol exists for this.**

## Architectural commitments

Two stances frame the rest of the proposal.

**The CMS builds context from the stream with stateless agents.**

- Internal **builder agents** subscribe to the two event topics (fast + batch). Before processing each event, the latest context filesystem is **mounted** to the agent; it processes the event and **updates** the filesystem.
- Two update modes (set by event classification):
  - **Fast lane** — one event, processed on arrival; context updated immediately.
  - **Batch lane** — the window's accumulated events, processed at window close.
- Builder agents are stateless: no in-RAM state between events. All long-lived state lives in the filesystem + a raw-conversation log (SQLite) inside the CMS (design in `research/RQ5-shared-memory.md`).
- Kafka and the filesystem are **internal** — neither crosses the CMS boundary.

**The only external surface is an MCP context tool.**

- Any coding agent — Claude Code, Cursor, a custom runtime — is given one MCP tool, and nothing else:
  - **Read** — `context_research` (multi-source search → synthesized, cited answer) and `context_get_urls` (fetch URL content).
  - **Write** — `context_update` (spawns an internal write agent that applies the caller's instruction to the store).
- The coding agent never sees Kafka or the filesystem. *Context is a service the CMS synthesizes on request, not a store the caller browses.*
- **This MCP tool is the thesis deliverable**; Study 1 tests whether it raises coding-task accuracy by fetching the right context.

## Proposed architecture

![Proposed Context Management System architecture](architecture.svg)

*Heterogeneous events enter the Context Management System and are first routed by an Event Classification stage onto a fast lane (urgent, signal-grade) or a batch lane (aggregated Context Units) — a latency split, not a Lambda-style dual codebase. The Context Engine indexes events into its bitemporal graph + RAG store, materialises per-entity views, computes the `context` object per outgoing event, and produces `(event, context)` envelopes onto partitioned Kafka topics. Agents are stateless consumers of those topics — events are routed to the right agent by partition key.*

**Processing note.** Both lanes flow through the same indexing path inside the Context Engine; only the *moment of publish* differs (fast = on arrival, batch = at window close). The per-event pipeline — bitemporal graph write + index update (in-memory), context computation, envelope emission — is described in *Context Engine — internal architecture* below.

## Context Engine — internal architecture

The engine ingests the raw event stream, builds the org's context from it, and serves that context to external coding agents over MCP. Concerns:

**Routing.**

- Each event is classified and published to its lane — fast or batch (see *Event classification*) — on internal Kafka topics.

**Context building (internal agents).**

- Stateless **builder agents** subscribe to both lanes. Before processing each event, the latest context filesystem is **mounted** to the agent; it processes the event and **updates** the filesystem.
- Fast lane → one event on arrival; batch lane → the window's accumulated events at window close. No in-RAM state between events.

**Internal store (never exposed).**

- An **agentic filesystem** in SQLite (the AgentFS model — one DB per organization) holds the living, synthesized context as files.
- A **raw-conversation log**, also SQLite, persists each agent conversation **URL-addressable** — an agent can fetch its prior context from a URL.
- Cold/raw event archive (audit, replay) lives separately in Paimon / Iceberg.
- *Scope (v1):* single-node per organization; concurrent-write handling and sharding-by-org are future work (see `research/RQ5-shared-memory.md`).

**Context I/O — the MCP surface (the only external surface).**

- External coding agents never touch the store. They call MCP tools, each served by an **internal agent** (read or write) over the store:
  - **Read — `context_research`** — searches the org's events, files, and sources → a **synthesized, cited** answer.
  - **Read — `context_get_urls`** — fetches the content behind one or more URLs.
  - **Write — `context_update`** — spawns an internal **write agent**; the caller hands it its full conversation/context plus an instruction, and the write agent applies the change.
- Reads return an answer, not raw files; writes go through the write agent. This MCP tool is the thesis deliverable.

**Authorisation.**

- One store per organization = the tenant boundary; every event and every MCP call is scoped to its org — no cross-tenant access.

### The internal event message (CloudEvents v1.0)

Events ride the internal Kafka topics as CloudEvents v1.0; builder agents consume them. The contract is versioned.

```jsonc
{
  "schema": "cms/event/v1",
  "event": {
    // CloudEvents v1.0
    "source":  "...",
    "id":      "...",
    "type":    "...",
    "subject": "customer:42",   // doubles as the Kafka partition key
    "time":    "...",
    "data":    { /* event payload */ }
  }
}
```

The `event.subject` field doubles as the Kafka partition key, guaranteeing per-entity ordering across consumers in the same group.

## Event classification: per-entity anomaly detection

![Event classification pipeline](rq2-classification.svg)

Each event is routed **fast vs batch** by **per-entity, label-free streaming anomaly detection**, inspired by Cloudflare's at-scale DDoS detection: per-entity adaptive baselines, deviation *scoring* (not labelled classification), operator-tunable sensitivity. Supervised urgency classification is rejected — no independent urgency labels exist, and a single global model doesn't fit a partitioned stream (full critique in `research/RQ2-anomaly-detection-reframe.md`).

**Keyed by `(entity × source-type)`:**

- **Entity = the CloudEvent `subject`** (`customer:42`, `service:checkout`) — already the Kafka partition key and Flink `keyBy`, so anomaly state shards on the axis the system already scales on.
- **Source-type = a feature extractor** (Slack / JIRA / telemetry); adding a source is a new extractor, never a global retrain.

**Pipeline:**

- **1 — Features.** Per-source adapter extracts source-agnostic features (inter-arrival gap, event-type mix, actor entropy, novelty, time-of-day) plus source-specific ones — preferring volume-decoupled *ratios* over raw counts so legitimate spikes don't false-positive. ~Free (<1 ms).
- **2 — Rules floor.** A small rule set (JIRA P0, Slack @oncall, Stripe dispute ≥ $N, PagerDuty incident) forces FAST — the mandatory floor for *urgent-but-not-anomalous* events a detector misses.
- **3 — Scorer (HBOS).** Features are normalised per-entity (robust-z `(x−median)/MAD`; JS/KL divergence for compositions), then scored by **HBOS** (`Σ log 1/hist_i` — linear-time, mixed-type, no per-entity covariance, scales to millions of entities). Baselines are **plain Flink keyed state** (5-min buckets, ~4-week window) — the routing decision never touches the context filesystem. PCA+Mahalanobis and Page-Hinkley are future upgrades.
- **4 — Decision.** `fast = Rules(e) OR (s_anom > θ_source)`, gated by an **absolute-volume floor** and **multi-window confirmation** (short 5-min AND long 1–4 h must both cross — SRE multi-burn-rate). `θ_source` is a static per-source setting — **no online feedback loop**. A *point* anomaly sends that event; a *window* anomaly emits one deduped **situation envelope**; the continuous score also ranks salience within the batch lane.

**No online feedback loop.** The threshold is a setting, not a learned quantity. Agents may emit `{event_id, verdict}` (`appropriate / too_urgent / too_slow / irrelevant`) but **only for offline evaluation** of routing quality, never on the hot path.

**Runtime behaviour:**

| Event | Per-entity signal | Guard / rule | Action | Why |
|---|---|---|---|---|
| Stripe dispute $50K on `customer:42` | order-value z ≫ customer's normal | past floor | **fast** (point anomaly) | deviant for this entity |
| Slack @oncall in `#incidents` | — | rule match | **fast** (rule floor) | scorer never consulted |
| JIRA P0 on a daily-P0 service | not anomalous (expected) | rule match | **fast** (rule floor) | detector alone would miss it |
| `#random` chit-chat on `channel:random` | within baseline mix | — | **batch** | normal for this channel |
| Error burst on `service:checkout` | event-type mix shift, both windows cross | past floor + multi-window | **fast** (one situation envelope) | confirmed anomalous burst |
| Verbose log blip on `service:logs` | short window crosses, long doesn't | fails multi-window | **batch** | transient blip suppressed |
| New `channel:launch` first message | novelty high, abs volume tiny | fails absolute floor | **batch** | trivial novelty suppressed |

## Delivery surface for stateless agents

Given the architectural commitments above (CMS owns context; agents are stateless), the delivery question reduces to: *which protocol carries the `(event, context)` envelope from engine to agent?* Three plausible families exist — a streaming push protocol over the LLM-tooling stack (MCP Streaming Resources), a pull/polling protocol over the same stack (Confluent RTCE's actual interface today), or a brokered event bus that stays out of the LLM-tooling stack entirely (Kafka). The proposal commits to the third.

### Decision: Kafka direct

![(event, context) delivery over Kafka](rq4-kafka-delivery.svg)

The Context Engine produces `(event, context)` envelopes onto Kafka topics partitioned by entity key. Concrete conventions:

- **Topics — split by lane.** `cms.events.fast.v1` (24 h retention, urgent events) and `cms.events.batch.v1` (7 d retention, windowed Context Units). Agent → engine feedback optionally flows on `cms.feedback.v1`, where agents write `routing_feedback` verdicts collected for **offline evaluation** of routing quality — not an online control loop (the event-classification threshold is a static setting).
- **Partition key.** The CloudEvent `subject` field (`customer:42`, `order:9931`, `tenant:acme`). Kafka hashes the key to a single partition, so per-entity ordering is preserved across the consumer group. Cross-entity ordering is *not* preserved — agents only ever care about per-entity timelines.
- **Partition count.** 50 for fast, 30 for batch. Sized for peak parallelism; over-provisioned because partition count is hard to change live.
- **Consumer groups — one per agent role.** `cms.agents.triage`, `cms.agents.compliance`, `cms.agents.analytics`. Multiple roles on the same topic give independent fan-out with separate offset cursors. Within a role, replicas share partitions via the cooperative-sticky assignor (Kafka ≥ 2.4) to avoid stop-the-world rebalances.
- **Idempotency.** The CloudEvent `(source, id)` pair is the dedupe key. The CMS guarantees at-least-once; agents dedupe.
- **DLQ.** `<group>.dlq` per role for envelopes that fail N retries.

Developers consume from these topics with whatever runtime fits the workload — a long-lived async consumer process, a Lambda-style ephemeral spawn (one invocation per event), Knative, Modal, anything that speaks Kafka. The CMS does not dictate.

**References:**

- Apache Kafka — partitioning, consumer groups, cooperative-sticky assignor: <https://kafka.apache.org/documentation/>
- CloudEvents v1.0 specification: <https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md>
- Push-protocol literature survey: `research/push-protocol/report.md`
- Confluent RTCE deep-dive: `research/confluent-rtce-deep-dive.md`

## Evaluation

The CMS is evaluated through several **independent studies**, each isolating one claim rather than a single end-to-end score. This section specifies the **memory & context-storage study**. The **burst / scalability study** (event classification + delivery under offered load) is a separate evaluation, and further studies (routing quality, end-to-end delivery) attach here as they are defined.

### Study 1 — Does the CMS store & serve context optimally?

- **Question.** Agents are stateless and the CMS *is* their only memory — read and written over MCP (`context_research` / `context_get_urls` / `context_update`), never by touching the store. Does the CMS store and serve the *right* context for an agent to act correctly — and more efficiently than naive memory strategies? ("Optimally" ⇒ measured against ablation baselines, not in isolation.)
- **Why a coding benchmark.** A real software task carries an *objective, automatic* success signal — the repository's own tests pass or fail — so task success isolates whether the FS surfaced the context the agent needed, with no LLM-judge noise.

**Dataset — SWE-bench-Live** (Microsoft, NeurIPS 2025; MIT-licensed code *and* data):

- Contamination-resistant: only GitHub issues filed after Jan 2024, refreshed monthly — fits a streaming thesis and avoids training-data leakage.
- Each instance ships `repo`, `base_commit`, `environment_setup_commit`, `problem_statement`, gold `patch`, `test_patch`, `FAIL_TO_PASS` / `PASS_TO_PASS`, `created_at` — i.e. an objective grader plus a pre-built, REPOLAUNCH-validated Docker environment.
- Pick the **top 1–10 repos by instance count**, so the validated test environments already exist; reported as a curated subset, *not* leaderboard-comparable to full SWE-bench-Live.

**Setup — replay each repo as an event stream into the CMS** (per instance):

- Check out `repo` at `base_commit` = the world state at the task's point in time (`environment_setup_commit` kept distinct for env/install).
- Replay the commit history *up to* `base_commit` as CloudEvents into the CMS, which indexes them into the agentic FS — **windowed** (last *N* commits / time window before `created_at`) and **ordered** (commit-time or topological); both choices stated explicitly.
- Interleave GitHub-native communication (issue + PR review comments, commit messages) by timestamp as the realistic "messages" layer — **not** Discord/Slack (ToS, privacy, no ground truth, not reproducible).
- Emit `problem_statement` as the triggering event.

**Agent.** Stateless coding agent (Claude Code via the SDK loop) whose *only* memory is the CMS over MCP: it pulls context via `context_research` / `context_get_urls`, implements the task, and writes back what it learned via `context_update` (which runs the internal write agent). It never touches the store directly; no agent-side state between events. The agent's own work — its commits/PR — flows back through the CMS via `context_update`, so the CMS is the system of record the run passes through.

**Scoring.** Apply `test_patch`, run `FAIL_TO_PASS` + `PASS_TO_PASS` in the prebuilt container → **% resolved**.

**Ablations (the "optimally" part — same tasks, swap the memory strategy):**

- *No FS memory* — agent sees only the issue (lower bound).
- *Naive full dump* — entire repo/history in-context, no selective FS (cost ceiling; context-window saturation).
- *CMS context tools (ours)* — context served and updated over MCP (internal agentic FS + raw-conversation log), plus variants of FS organization / indexing strategy.

**Metrics.**

- Primary: **% resolved** (task success).
- Context efficiency: tokens retrieved/served, context precision & recall against the files the gold `patch` touches, MCP read/write tool-call counts (`context_research` / `context_update`).
- Cost & latency per resolved task.

**Honesty notes.** Curated repo subset (not full-leaderboard comparable); commit ordering + windowing stated up front; "messages" are GitHub-native only.

### Study 2 — Does the classifier route correctly? (per-entity anomaly detection)

- **Question.** The router is the HBOS-based, per-entity, *label-free* anomaly detector — no independent "urgency" labels exist (the reason supervised classification was dropped), so we cannot compute a plain "urgent vs. not" AUC on a raw stream. Ground truth is obtained two ways, mirroring the HBOS paper's labeled-benchmark methodology (Goldstein & Dengel) but adapted to streaming.
- **Core claim to prove.** HBOS *fails on local/contextual outliers* (the paper's pen-local AUC 0.77 vs. LOF 0.99 — a global histogram can't model local density). Our per-entity normalization (robust-z `(x−median)/MAD`, compositional JS/KL) turns a per-entity *local* anomaly into a *global* one, where HBOS is strong. The headline result is the ablation **global HBOS vs. HBOS + per-entity normalization** recovering that gap.
- **Ground truth — clean labels.** Standard streaming AD benchmarks (**NAB**, **TSB-AD / TSB-UAD**) plus controlled anomaly injection (volume burst, type-mix shift, novelty) at known timestamps on real per-entity backbones. Gives exact metrics and carries the decisive ablation.
- **Ground truth — weak labels.** Real domain streams with a declared-priority field: the **Public Jira dataset** (Montgomery et al., MSR'22: 16 Jiras incl. Apache, 2.7M issues, 32M timestamped changes; `priority` ∈ {Blocker, Critical, Major, Minor, Trivial}; CC-BY) [<https://arxiv.org/abs/2201.08368>] whose per-issue changelog replays as a per-entity stream, plus **GH Archive** and PagerDuty-style incidents. A correlational check — does the score rank Blocker/P0 above routine traffic? — on the **rules floor**, with label noise / selection bias stated.
- **Baselines.** ECOD / COPOD, Isolation Forest, Half-Space Trees, Robust Random Cut Forest, Page-Hinkley / ADWIN; LOF / k-NN as the local-anomaly reference.
- **Metrics.** **VUS-PR** and **affiliation-F1** (range-aware, threshold-free), **NAB** latency-weighted score, recall@FPR ≤ 1%, detection delay. Naive point-adjusted F1 is avoided — Kim et al. (AAAI 2022) show a random scorer "wins" under it.
- **Entity** = the CloudEvent `subject` (per-project, per-service, per-customer) — the detector is evaluated at the granularity it shards on.

## Deliverables

1. **Thesis report** (this document's parent) synthesising the full design: streaming architecture, event classification, batch aggregation, delivery surface, and the shared-memory filesystem.
2. **`(event, context)` Kafka-delivery contract** — JSON Schema for the envelope plus per-topic conventions (naming, partitioning, consumer-group rules, idempotency, DLQ). The architectural specification an open-source CMS implementation must conform to. Replaces the earlier MCP Streaming Resources profile draft.
3. **Reference Context Engine implementation** — open-source, backed by a free-tier streaming stack (Redpanda + Flink + Paimon) and a custom bitemporal graph + RAG store for the memory layer, producing `(event, context)` envelopes onto Kafka topics.
4. **Python SDK for agent authors** — thin Kafka consumer + envelope decoder + `routing_feedback` helper. Lets developers write a stateless agent against the contract in a few lines; raw Kafka clients in any language remain a first-class option.
5. **Empirical evaluation on ProAgentBench** — Tang et al., *ProAgentBench: Evaluating LLM Agents for Proactive Assistance with Real-World Data*, arXiv:2602.04482 (Feb 2026). <https://arxiv.org/abs/2602.04482>
   - 28,000+ events from 500+ hours of real user sessions with preserved bursty interaction patterns (burstiness `B = 0.787`), not LLM-synthesised.
   - Hierarchical tasks: (i) **timing prediction** — when to intervene — maps directly onto our fast-vs-batch routing decision (*Event classification*); (ii) **assist content generation** — what to deliver — exercises our batch-lane Context Unit shape (batch aggregation) and Kafka-direct delivery (*Delivery surface*).
   - Metrics already match the loop we need: timing appropriateness + assist-content quality ≈ our `routing_feedback` verdict vocabulary (`appropriate` / `too_urgent` / `too_slow` / `irrelevant`).
   - Real-event-stream data (vs. synthetic) also directly validates the paper's finding that long-term memory + historical context lift prediction accuracy — which is exactly what the Context Engine's materialised views are for.
