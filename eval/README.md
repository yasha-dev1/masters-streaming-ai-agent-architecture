# Evaluation

Everything about evaluating the two-lane streaming context engine lives here.

**Review the [1,000-instance Kafka benchmark SPA](../../harnext-context-engine/apps/eval/benchmarks/kafka-1000-v1/review.html).**
It contains 800 historical MCP-only questions and 200 real issue/PR coding
candidates with shell + MCP contracts, deterministic gold/source evidence,
visible tests and reference patches. [Methodology and remaining admission gates](../../harnext-context-engine/apps/eval/benchmarks/kafka-1000-v1/README.md).
No agent experiment has been run; this is the user-review release.

Current executable merged-E2 strategy route:
[shared YAML configuration and outside-agent MCP evaluation](../../harnext-context-engine/apps/eval/E2-STRATEGIES.md),
with [September 12 verification results](../../harnext-context-engine/apps/eval/STATUS/E2-STRATEGIES-20260912.md).
This implements the configuration infrastructure; full historical-task screening
and held-out confirmation remain pending.

| File | What it is |
|---|---|
| `evaluation.md` | **The implementer specification** (single source of truth): claims, corpora, ground truth, phases, E1–E6 runbooks with exact scores and variations, statistics, gates, `eval/` repo layout, decisions register. |
| `evaluation.html` | Rendered twin of `evaluation.md`. Regenerate with the command below. |
| `artifacts/01-thesis-dossier.html` | Proposal summary, implementation-vs-proposal gap audit, draft outline. Published: https://claude.ai/code/artifact/f54b1f92-204c-4239-964a-c21d09f8a645 |
| `artifacts/02-research-brief.html` | Literature answers to the five design questions (storage responsibility, routing, envelope, processing contract, evaluation), 164 checked references. Published: https://claude.ai/code/artifact/fa606e33-1067-4b59-82f3-4869b580f10f |
| `artifacts/03-evaluation-protocol.html` | Evaluation design: claims → experiments, two corpora, ground-truth catalogue, threats, budget, 20 defense questions with answers. Published: https://claude.ai/code/artifact/e411b524-7b6b-41be-b16e-2a01f84e0c10 |
| `artifacts/04-evaluation-runbook.html` | Decisions D1–D12, phase plan, step-by-step procedures and validity checks per experiment, go/no-go gates. Published: https://claude.ai/code/artifact/5678dc66-ce9d-4229-9502-a9556468d4cf |
| `artifacts/05-e3-store-design-plan.html` | Store-design experiment plan (2026-09-08): E2 merged into E3; 17 store conditions (floors, retrieval, file-based F0–F7, graph G1/G2, hybrid), staged A–F with builder and reader grids, sealed confirmation set, decision rule, cost model, open decisions. Published: https://claude.ai/code/artifact/8ac5e0b0-0a74-4775-be1e-92e92addbee8 |
| `artifacts/06-e3-experiment-blueprint.html` | Comprehensive proposed E3 protocol (2026-09-12): 32-factor register, 12 file / 12 retrieval / 8 graph development cells, deterministic scoring, staged selection and sealed confirmation, 26 planned chart recipes with hypotheses and decision consequences, implementation backlog, cost calculator, and explicit unresolved execution inputs. Local self-contained artifact; not preregistered. |
| `evaluation-review-2026-09-12.md` | Readiness and methodology audit of E1 and merged E3, including confirmed code gaps. |
| `e2-smoke-status-2026-09-12.md` | First live merged-E2 infrastructure check: Codex `gpt-5.6-luna` / medium, 30 valid/budgeted answers, exact snapshot/tool replay audit, actual quality failures, code/runbook links and remaining study gates. |
| [Fresh E2 smoke: 20 graphs](../../harnext-context-engine/.harnext/artifacts/e2-smoke-20260912T100821Z.html) | New stores and readers with the same Codex model/effort. Values: S3 10/10, S1 7/10, S0 9/10; all 30 execution/audit gates pass. Interactive graphs, PDF and figure/data exports; synthetic smoke only. |
| `e3-fold-audit-2026-09-12.json` | Measured R-H1 scheduling counts and hashes: 29,041 invocations under the current E3 configuration. No LLM calls or store construction. |
| `template/eval-template.html` | Pandoc template used to render `evaluation.html`. |

Rendered spec: https://claude.ai/code/artifact/b1d2eac6-46f9-40f6-a9b3-25bf16a4f535

Where documents disagree, `evaluation.md` wins; the artifacts are the reasoning trail that led to it (dossier → brief → protocol → runbook → spec). The store-design plan (05) amends the spec's E2/E3 sections and will be folded into `evaluation.md` once registered.

Blueprint 06 proposes a further revision of 05 based on the September 12 audit and design discussion. It is the latest planning artifact, not an executed experiment or committed preregistration; it does not yet override `evaluation.md`.

Implementation now calls the merged study **E2 (merged E2/E3)**. Its initial
synthetic smoke is recorded in `e2-smoke-status-2026-09-12.md`; this does not mean
the full blueprint matrix or a thesis confirmation run has been executed.

The [implementation and coding-task extension](../../harnext-context-engine/apps/eval/E2-DEVELOPMENT-TASKS.md)
adds a constructed development package (32 QA probes, five coding fixtures),
deterministic span-exposure metrics, and the proposed historical PR admission,
test scoring, matched controls, and chart protocol. It is a development extension;
real historical task selection and thesis confirmation remain pending.

```bash
pandoc eval/evaluation.md -t html5 --template eval/template/eval-template.html --toc --toc-depth=2 -o eval/evaluation.html
```
