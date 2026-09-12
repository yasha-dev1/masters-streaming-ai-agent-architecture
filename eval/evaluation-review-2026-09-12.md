# Evaluation review — 12 September 2026

Recommendation: begin the merged E3 preparation and pilot. Keep C2 fidelity inside E3, retain E4–E6 as distinct questions, and reduce the initial structure sweep. The current plan is extensive enough, but its cost assumptions and several validity rules need revision before registration or paid builds.

This is an advisory review, not a preregistration or a change to the accepted specification. No LLM evaluation, store build, annotation, or deployment was performed for this review.

**Evidence and present status**

I read the thesis proposal, original evaluation specification, September 8 merged plan, E1 run-4 registration/results/validity tables, handoff and framework limitations, and relevant replay, reader, store, harness, and probe code. I also inspected the ignored local run directories. The supplied Claude artifact URLs were inaccessible through the web tool. The local README identifies `artifacts/05-e3-store-design-plan.html` as the second supplied artifact; I could not establish an exact local match for the first URL, so used `evaluation.md` as the original authoritative six-experiment plan.

| Work | Evidence available | Next obligation |
|---|---|---|
| Shared framework | E1–E6 implementations and offline smoke outputs | Extend the older E2/E3 machinery for the merged design |
| Kafka corpus | Frozen replay exists locally; parent SHA verified | Freeze R-H1 and its initial-state/provenance contract |
| E1 routing | Run 4 completed; 387,403 evaluation events, 12 policies | Close or explicitly scope outstanding validity issues |
| E2 fidelity | Older implementation and smoke runs | Retain the measurements inside merged E3 |
| Merged E3 | September 8 design; explicitly unregistered | Stage A preparation, then cost and baseline pilots |
| E4 envelope | Implementation/smoke evidence | Real action-quality experiment after choosing a store |
| E5 cadence | Implementation/smoke evidence | Cheap schedule analysis now; full freshness/cost study later |
| E6 system | Implementation/smoke evidence | Live broker/load experiment; can proceed independently |

The engine is currently on `main` at `850bcbc`, which merged the evaluation branch. Older instructions that require checking out `eval-framework` are stale.

E1's primary recall@2% on rule-negative events is 0.089 for global HBOS, 0.020 for random, 0.005 for per-entity HBOS, and 0.000 for the guarded design. Global HBOS minus random is +0.069 with subject-cluster CI [0.047, 0.093]. This is evidence against the proposed per-entity/guarded mechanism within this run, subject to its unresolved validity limitations. It is not a validated general claim about urgency, and the action-benefit part of C1 has not been tested.

The run remains explicitly **non-evidentiary**. Both annotator columns in the 100-item sanity sheet are empty. The required random-VUS geometry, rule-floor feasibility, and corpus-preflight checks also fail. The 1% feasibility problem should not be conflated with the 2% primary: the handoff reports all 52 months feasible at 2%. Flink and the registered synthetic situation study remain unrun. Annotating alone will not clear all gates. Preserve run 4's status; document metric/budget scope changes as amendments, with any later analysis clearly distinguished. E3 preparation does not need to wait for E1 annotation.

**What merged E3 actually tests**

At the same observable point in an identical stream, which maintained representation allows a fixed reader to answer questions accurately, at what read and build cost, and with what retention and update behavior?

The five families cover current state, historical state, supersession, cross-source relationships/changed files, and appropriate abstention. For example: an issue changes owner twice and links to a merged PR. Can the reader return its current owner, the owner at an earlier time, the linked PR/files, and UNKNOWN for an unavailable field without mixing old and new values?

The planned corpus is Kafka R-H1, January–June 2026, with January–April warm-up and May–June queries. The plan calls for 300 selection and 300 sealed confirmation probes, 60 per family in each set, with an 8,000-token primary read cap and 2,000/32,000-token secondary caps. Its named primary contrasts are F5 versus F1 and F5 versus dense retrieval. The broader question also requires comparison with the strongest cheap retriever, including hybrid retrieval.

This merges **E2 and E3 only**. It does not establish whether context improves an agent's action (E4), what cadence is economical at acceptable freshness (E5), or whether the running pipeline meets throughput and latency targets (E6). It also does not benchmark Git versus SQLite durability or database throughput merely by comparing file and graph representations.

**Measured cost correction**

I streamed and hashed the local replay, selected timestamps in [2026-01-01, 2026-07-01), validated the selected records as EvalEvent, and ran the existing event-time `run_pipeline` with a counting store whose `fold` performs no writes or model calls. The config was `apps/eval/configs/s3-curated.yaml`. Full results are in `e3-fold-audit-2026-09-12.json`.

| Measurement | Result |
|---|---:|
| R-H1 events | 37,719 |
| GitHub / JIRA / dev-mail events | 22,685 / 11,039 / 3,995 |
| May–June events | 11,905 |
| Unique event subjects | 4,581 |
| Current-config folds | 29,041 |
| Fast / batch folds | 1,266 / 27,775 |
| Single-event folds | 22,688 (78.1%) |
| Folds with rules disabled, same windows | 28,914 |

The current window is per entity, with a 30-second inactivity gap, 20-event cap, and 120-second maximum age. Sparse entity streams almost always close on the gap. Disabling fast routing barely reduces the number of invocations.

At the plan's illustrative 45 seconds per fold, 29,041 folds would take **15.1 serial days per store**, before retries. This is arithmetic, not measured model latency. The plan's 6,000-fold example implies only 3.1 days. Dollar costs remain unknown until real usage is measured; model tiers, caching, output, retries, and graph extraction costs differ.

Move dry schedule counting before any full store construction. Test several prospectively defined microbatch schedules using the same counter. Consider packing independent entity windows into a single invocation within an explicit event-time interval, with an input-token cap and no crossing of required query cutoffs. Apply the chosen schedule to every primary learned layout. Record this as an experiment setting; it changes when context becomes available and is not a cost-free optimization of production semantics. Benchmark native cadence separately in E5.

The paid pilot should sample early, middle, and late stream prefixes: a pilot consisting only of the first 200 small-store folds can underestimate later navigation and rewriting costs. Measure full-replay economics on the eventual finalists. Do not share a curated warm-up across supposedly independent builders or replicates.

**A smaller initial matrix**

| Conditions | Why retain them |
|---|---|
| A0 and an answerability ceiling | Reader prior and grader/answerability checks; not deployment contenders |
| R-tail, F0 | Last-N context and raw files distinguish retrieval from write-time work |
| F1 | Strong deterministic per-entity state and timeline baseline |
| R-bm25, R-vec, R-hyb | Lexical, dense, and combined retrieval baselines |
| F4, F5 | Same curation contract without/with global organization |
| F7 | Structured temporal fact ledger alongside prose |

This is nine store designs plus controls; R-tail's two N values are separate configurations if both are retained. Add F6 free-form as the first exploratory challenger if agent-invented organization is important. Add one temporal graph when representation breadth is a central thesis claim. Defer F2, the second graph stack, and the full reader grid. Move F3's weekly-versus-window comparison to E5, or give it a cadence-matched comparison in E3.

Keep F4 if claiming that an index/cross-links help: dropping it removes the closest ablation for that mechanism. F5 versus F1 changes both synthesis and organization, so it measures their package rather than organization alone. If F7 beats F5, a small deterministic fact-ledger baseline can distinguish the value of exact state replay from the value of LLM writing.

The plan says 17 conditions, but its table has 18 rows including H1; its category counts omit H1. The planned builder experiment is three structures by four builder configurations including the initial baseline, not a crossed 3-by-3 harness/model design. Make these counts explicit before estimating cost.

Screen at 8k first. Run the 2k/32k curve on finalists and fixed baselines using the already-built snapshots. Cache immutable extraction/chunking/embeddings and identical reader requests with keys including snapshot, prompt, model, tool contract, budget, and replicate identity. Cache hits are reused observations, not new independent replicates. Stop early only using registered selection rules, never after inspecting confirmation scores.

**Context storage contract**

Keep the frozen event archive outside every experimental store. It is the replay and provenance source; a learned store is a derived representation. Every arm receives the same allowed event fields, starting state, and ordering. Do not give F1 gold answers or give learned arms richer source text.

For file conditions, reuse the existing versioned Git backend for evaluation. Give each condition/builder/replicate its own store and immutable snapshots. Record input IDs, replay watermark, logical availability time, snapshot ID, prompt/tool/model hashes, fold status, and usage. The reader receives only the allowed snapshot and tools. This choice supports reproducibility; it is not evidence that Git is the best production backend.

F5 retains OVERVIEW, facts, timeline, index, and superseded-history files. F7 adds machine-readable assertions with entity, field/relation, value, valid-from/to, known-from/to (or an equivalent immutable transaction history), and source event IDs. Valid time answers when a fact applied; knowledge time answers when it became available. Merely putting valid-from/to in YAML does not enforce supersession: validate intervals, provenance, and contradictory current values mechanically.

For temporal queries, distinguish reader cutoff T from historical target T'. An edge must have been known at T and valid at T'. Filtering only graph ingestion time is insufficient. For retrieval, filter eligible document versions before top-k; a full-history BM25 index also needs prefix-specific document-frequency statistics if claiming strict historical replay. Current edited text cannot be made historical just by masking its original event timestamp.

Important planned-shell risk: `replay/snapshots.py:materialise` currently clones the repository and checks out the historical SHA. That leaves later history in `.git`. This is not evidence that current deterministic readers leaked, but a future shell reader could access it. Export only the selected snapshot, or isolate its tools from Git history, raw replay, gold, other stores, and prior answers.

**Validity changes before registration**

1. **Keep development separate from confirmation.** Use an unscored development pool for fixing prompts, gold, and baseline construction. Split evaluation by entity or related issue/PR/thread group where feasible; near-duplicate questions on one fact should not span selection and confirmation. Register candidate-selection rules, then freeze candidates, configurations, and replicate builds before opening confirmation. Always confirm F5 as well as its fixed baselines, even if F5 is screened out. Do not choose additional replicate targets after viewing confirmation rankings.
2. **Treat historical text as a potential leak.** E1 documents export-time descriptions/comments. They can reveal outcomes unavailable at the event's timestamp even when derived field gold is correct. Use trustworthy as-of structured records as the primary input or exclude/delay text without defensible historical availability. A separate full-text sensitivity can expose the impact. Also define initialization for issues that predate January; silently initializing stores from a later export is not valid warm-up.
3. **Separate representation from cadence.** Weekly F2/F3/G2 versus per-window F5 answers a whole-system question. For a storage claim, match available event prefixes and update schedule; for a deployment claim, retain native cadence and explicitly attribute freshness differences. Measure event receipt and correct-state availability separately: an ID written into a ledger does not prove that the relevant fact became usable.
4. **Make fairness a cap, not forced consumption.** An agent that answers correctly after 900 tokens should not be forced to read 8,000. Enforce hard cumulative tool-output caps with accounted truncation, and separately measure total billed input/output, tool calls, repeated context, cache use, and latency. Native graph query tools that invoke another model must expose that model and usage or be labeled a different retrieval system.
5. **Do not make baseline separation a validity gate.** The claim that tied cheap baselines imply bad probes is unjustified. A real tie is a legitimate result. Do not reweight an already-frozen selection set until a favored separation appears. Diagnose coverage/difficulty on development data, preserve comparisons, and report no detected gain when appropriate.
6. **Replace significance-based tie breaking.** A confidence interval spanning zero does not demonstrate equivalence or that curation loses. Use superiority, noninferiority within a justified practical margin, or inconclusive. Only choose cheaper storage as scientifically quality-equivalent after an appropriate noninferiority/equivalence criterion. Correct the fixed primaries and any confirmatory challenger family for multiplicity. Use entity/group bootstrap for the mixed exact/F1 macro; ordinary McNemar is only supplementary on appropriately binary paired outcomes and does not address entity clustering.
7. **Replicate both sides that are stochastic.** Three F5 replicates cannot establish the variance of F7, a graph, or another builder configuration. Three total builds are a sensible initial floor for each learned confirmatory contender, not a guarantee of precision. Use repeated identical instructions; the current live harness implements its seed by appending a seed/tie-break sentence, which changes the prompt rather than controlling provider randomness. Label those runs accordingly or change the protocol. Repeat readers on a diagnostic subset to estimate their variation.
8. **Check precision before committing to 600 probes.** Sixty per family is useful coverage, not a power argument. Estimate paired disagreement and entity dependence on development data and simulate power/CI width for the proposed 5-point effect. Illustratively, with independent binary pairs and 20% discordance, a normal approximation needs roughly 627 pairs for 80% power at a 5-point difference before clustering and multiple comparisons. This is not a power calculation for the actual mixed-score design.
9. **Fix metric interpretation.** UNKNOWN accuracy within the abstention family is abstention recall, not precision; precision needs all predicted abstentions, including answerable probes. Closed graph edges show recorded history, not correct supersession. An erosion slope needs uncertainty and an eligible, as-of-correct panel; a negative point estimate alone does not demonstrate degradation. Match difficulty or distinguish retained historical facts from changing current-state tasks.

The answerability ceiling must include linked PR/thread evidence through T, not merely exact-subject events. An oracle evidence bundle is a useful grader/reader check if full entity history exceeds context; keep it separate from deployable retrieval and count exclusions/truncations.

**Harness comparisons that support interpretable claims**

Start with F4/F5/F7 under one builder model, one harness, one tool policy, one scheduling contract, and one reader. Then on two finalists:

- Hold the model and prompts fixed and change file tools versus shell tools: a tool-interface effect.
- Hold harness and tools fixed and change the builder model: a model effect.
- Compare an alternative agent stack only as a whole-stack comparison if it cannot use the same model and equivalent prompts/tools. A Claude-versus-Codex comparison with different models cannot identify a pure harness effect.
- Hold stored snapshots fixed and change reader strength first; add a targeted search-tool ablation only if useful. Apply feasible native interfaces to graphs and retrieval, rather than pretending every arm supports file tools.

The current Codex registry entry raises `NotImplementedError`. The current reader makes one completion call after material selection. Dense retrieval embeds the visible corpus per probe. The main probe CLI uses a `normalised-smoke-adapter` rather than the required independent raw-export gold path. Anthropic and Voyage are absent from the current virtualenv and undeclared in the eval package. These confirm that selecting a real provider in YAML is insufficient to launch the merged experiment.

Resolve tool instructions too: the seeded F5 operating manual currently says not to run shell commands, whereas the harness allows Bash. A shell-only experiment must not inherit contradictory instructions. Pin actual available model IDs and tool/SDK versions and measure their costs in the pilot; the names and prices in the September 8 plan are not validated deployment settings.

**Recommended execution order**

1. Record the E2-to-E3 amendment and keep E4/E5/E6 in the claim map. Close E1 human/validity work alongside E3 preparation; avoid another classifier sweep merely to seek a positive result.
2. Freeze R-H1, allowable historical fields, initialization, development/selection/confirmation groups, metrics, and the observable-time contract. Perform dry schedule counting and set the build ceiling from measured pilot usage.
3. Implement the tool reader, safe snapshot exports, persistent temporal retrieval, independent gold and probe-window plumbing, usage logging, and preregistration enforcement. Run the two-human development pilot and planted leakage/budget/invalid-gold checks before scaling.
4. Register the study, run the inexpensive baselines, and perform the matched F4/F5/F7 comparison. Keep routing fixed; all events eventually reach every store. Do not let an E1 scorer change the input population across E3 arms.
5. Replicate finalists, run targeted builder/reader checks and budget curves, then perform the frozen confirmation comparison. Report unresolved intervals honestly and preserve cost-quality Pareto results.
6. Use the selected store in E4 action tasks and E5 cadence/freshness runs. Add a small controlled stream covering corrections, late events, and missing information; use the existing synthetic machinery where it supports those cases. Run E6 on the live system with declared hardware/load conditions. Broader project replication is more valuable than many extra harness cells once the mechanism is understood.

The study is sufficiently broad for the storage part of a master's thesis. Its distinct contribution should be time-correct maintenance over a heterogeneous stream, derived gold, and measured build/read/freshness tradeoffs. Generic filesystem organization is already studied: Zhou et al. report search-cost benefits without consistent answer-quality gains and show that tools affect store shape. This supports retaining read economy as a meaningful outcome even when accuracy ties. [Primary paper, July 2026](https://arxiv.org/html/2607.26637v1).

Source anchors: `eval/evaluation.md`; `eval/README.md`; `eval/artifacts/05-e3-store-design-plan.html`; `eval/results/e1/{results_summary.md,validity.csv,preflight.csv,human_sanity_sample.csv}`; engine `docs/HANDOFF-E1-REAL.md`, `apps/eval/LIMITATIONS.md`, and the source paths discussed above. The attached fold audit is a planning measurement, not an experiment result.
