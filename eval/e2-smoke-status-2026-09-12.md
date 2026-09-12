# E2 (merged E2/E3): first live infrastructure check

The merged context-engine experiment is now called **E2** in the new implementation
path. The older artifacts call it E3. This naming does not change the preregistered
status: the full configuration-selection study remains proposed, not completed.

The initial live smoke passed with **Codex CLI 0.154.0, `gpt-5.6-luna`, medium
effort**, using Harnext's Git-backed builder and explicit host-controlled file tools.
Native shell execution could not initialize its bwrap sandbox on this machine.

| Store | Valid/budgeted answers | Exact values | Strict evidence |
|---|---:|---:|---:|
| S3, model-curated indexed files | 10/10 | 9/10 | 8/10 |
| S1, deterministic fixed template | 10/10 | 7/10 | 8/10 |
| S0, raw event files | 10/10 | 9/10 | 9/10 |

All 30 saved tool traces reproduced exactly from their snapshot SHAs. The builder
performed two real incremental builds; reader fixes were tested against those
same snapshots without rebuilding. The full builder/eval regression suite passes
363 tests. These are synthetic engineering checks, not thesis-quality estimates
or evidence that one store is optimal.

The check caught and fixed incomplete-build publication, provider-cache identity
reuse, and a reader that could exhaust its call budget without submitting an
answer. It also exposed actual quality limits: S1 omits arbitrary release-date
facts; the complete cross-source file set is missed under the small tool-call
budget; S3 drops an older PR link from its current issue projection while the
link remains elsewhere in the store.

Next: freeze exact tokenizer and citation/abstention accounting, add the strong
lossless deterministic baseline, and construct a small causally audited real-data
development panel. Then implement the file, semantic-retrieval, graph and harness
matrices from blueprint 06. Current retrieval caps use bytes, not exact model
tokens. The full matrix and sealed confirmation remain pending.

- [Verified results and evidence links](../../harnext-context-engine/apps/eval/STATUS/E2-SMOKE-2026-09-12.md)
- [Implementation runbook and reproduction commands](../../harnext-context-engine/apps/eval/E2-CONTEXT.md)
- [Integration handoff](../../harnext-context-engine/apps/eval/STATUS/T11-E2-CODEX.md)
