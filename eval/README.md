# Evaluation

Everything about evaluating the two-lane streaming context engine lives here.

| File | What it is |
|---|---|
| `evaluation.md` | **The implementer specification** (single source of truth): claims, corpora, ground truth, phases, E1–E6 runbooks with exact scores and variations, statistics, gates, `eval/` repo layout, decisions register. |
| `evaluation.html` | Rendered twin of `evaluation.md`. Regenerate with the command below. |
| `artifacts/01-thesis-dossier.html` | Proposal summary, implementation-vs-proposal gap audit, draft outline. Published: https://claude.ai/code/artifact/f54b1f92-204c-4239-964a-c21d09f8a645 |
| `artifacts/02-research-brief.html` | Literature answers to the five design questions (storage responsibility, routing, envelope, processing contract, evaluation), 164 checked references. Published: https://claude.ai/code/artifact/fa606e33-1067-4b59-82f3-4869b580f10f |
| `artifacts/03-evaluation-protocol.html` | Evaluation design: claims → experiments, two corpora, ground-truth catalogue, threats, budget, 20 defense questions with answers. Published: https://claude.ai/code/artifact/e411b524-7b6b-41be-b16e-2a01f84e0c10 |
| `artifacts/04-evaluation-runbook.html` | Decisions D1–D12, phase plan, step-by-step procedures and validity checks per experiment, go/no-go gates. Published: https://claude.ai/code/artifact/5678dc66-ce9d-4229-9502-a9556468d4cf |
| `artifacts/05-e3-store-design-plan.html` | Store-design experiment plan (2026-09-08): E2 merged into E3; 17 store conditions (floors, retrieval, file-based F0–F7, graph G1/G2, hybrid), staged A–F with builder and reader grids, sealed confirmation set, decision rule, cost model, open decisions. Published: https://claude.ai/code/artifact/8ac5e0b0-0a74-4775-be1e-92e92addbee8 |
| `template/eval-template.html` | Pandoc template used to render `evaluation.html`. |

Rendered spec: https://claude.ai/code/artifact/b1d2eac6-46f9-40f6-a9b3-25bf16a4f535

Where documents disagree, `evaluation.md` wins; the artifacts are the reasoning trail that led to it (dossier → brief → protocol → runbook → spec). The store-design plan (05) amends the spec's E2/E3 sections and will be folded into `evaluation.md` once registered.

```bash
pandoc eval/evaluation.md -t html5 --template eval/template/eval-template.html --toc --toc-depth=2 -o eval/evaluation.html
```
