# Evaluation results

One directory per experiment, holding the run that stands as the result for the paper.

| Experiment | Run | Verdict | Published page |
|---|---|---|---|
| `e1/` | Kafka run 4, `20260907T000315Z-e1-kafka` (2022-01 → 2026-07, 387,403 events, 12 policies, registered in `e1/PREREG-run4.md`) | Global HBOS is the best scorer at the 1–5 % budgets (recall@2 % 0.089 vs random 0.020); IsolationForest ties it; the per-source split reaches parity only above ~5 % because the JIRA signal saturates; no per-entity scorer beats chance; the guarded design (R5) admits nothing. Non-evidentiary by its own gates until the human sanity sample is annotated. | https://claude.ai/code/artifact/64b0f4e2-646e-44fa-9bb9-c9669131a76c |

## `e1/` contents

- `e1-kafka-run4-report.html` — the published report page (self-contained, figures embedded).
- `results_summary.md` — every table in Markdown and LaTeX (recall@b, contrasts, guard funnel, per-source, lateness, label diagnostics, remediation record, gates).
- `fig_recall_at_budget.png`, `fig_rule_floor_saturation.png`, `fig_label_posterior.png` — paper figures.
- `primary_contrasts.csv`, `recall_at_budget_rule_negative_pooled.csv`, `per_source_recall_2pct.csv`, `guard_funnel.csv`, `lateness_rule_negative.csv`, `label_definition_recall.csv`, `label_definition_contrasts.csv`, `metric_remediation.csv`, `label_diagnostics.csv`, `validity.csv`, `calibration.csv`, `robustness.csv`, `delays.csv`, `label_situations.csv`, `rule_floor_breakdown.csv`, `rule_hit_share_by_month.csv`, `metrics.csv` — the run's tables.
- `results.json`, `manifest.json`, `config.yaml` — the run record (primary metric, contrasts, gates, hashes).
- `human_sanity_sample.csv` (blind) and `human_sanity_key.csv` — the 100-item annotation sheet still to be filled in by two annotators.
- `PREREG-run4.md` — the registration committed before the run.

Source of truth for the code and the full run history: `harnext-context-engine`, branch `eval-framework`
(`apps/eval/reports/e1-kafka-run4/`, `docs/HANDOFF-E1-REAL.md`).
