# Evaluation Strategy — JIRA tickets as scenarios, graded by order-discrepancy vs the golden reference

*Companion to `research/evaluation-methodology.md` (metric layers) and `eval-prototype/` (feasibility). Operationalizes proposal.md Study 3; execution/validation design borrowed from Meta's Agents Research Environments (ARE) [<https://facebookresearch.github.io/meta-agents-research-environments/quickstart.html>], minus wall-clock deadlines.*

## Plan & prep

- **Repo:** apache/spark — most events of all live-JIRA Apache projects; mandated `[SPARK-NNNNN]` keys give a deterministic ticket↔commit join, already verified (**89.7% link recall, ~100% precision**).
- **Dataset:** pull three streams onto one timeline, joining each fixed-bug ticket to its fixing commit/PR by the embedded key (table below).
- **Scenario:** one per fixed-bug ticket (table below).
- **Simulator:** replay the scenario's events in real recorded order into the CMS (system under test); trigger the stateless MCP-only agent on the ticket; it reads context via `context_research`/`context_get_urls` and emits its change sequence (commits/file edits).
- **Run:** one scenario per ticket, deterministic (frozen event log), with paired ablations on the identical stream — **no-CMS**, **naive full-dump**, **CMS-over-MCP (ours)**.

**Dataset — three streams, one timeline:**

| Stream | Source | Content |
|---|---|---|
| JIRA | live REST API | tickets + changelog + comments |
| Commits | treeless clone | messages, dates; changed-file list (on demand) |
| PR review | GitHub API | review / discussion |

**Scenario — one per fixed-bug ticket:**

| Part | Definition |
|---|---|
| Trigger | the fixed-bug ticket |
| Backlog | the repo's prior events, replayed in **real recorded order** |
| Cutoff | the ticket's `created` time — the fix and everything after it are held out |
| Golden | the ticket's fixing commit(s) as an **ordered change sequence** (file changes in commit order; optional interleaved PR/status events) |

## Evaluation — order-discrepancy against the golden reference

We do **not** grade correctness by running tests — SWE-bench's execution harness (apply patch → run `FAIL_TO_PASS`/`PASS_TO_PASS`) **cannot be recreated here**: SPARK is JVM (sbt/Maven), there is no off-the-shelf grader, and deriving per-instance test sets means building and running every fix. Instead we exploit that both our system's output and the golden reference are **ordered sequences of changes**, and measure their **discrepancy** — fully deterministic, no build/run.

The only temporal requirement is a **gate, not a score**: events are fed in real recorded order, up to the cutoff, and we verify no event with `time > cutoff` reached the agent (no future leakage), so the golden sequence is genuinely held out.

**The two sequences.**

| Sequence | Definition |
|---|---|
| Golden `G` | the ticket's fixing commit(s) flattened to an ordered list of file-changes, by (commit-time, then in-diff order): `G = [f₁, f₂, …]` |
| Predicted `P` | the file-changes the agent emits while resolving, in emission/commit order: `P = [f₁', f₂', …]` |

**Two orthogonal discrepancy axes — report both (they trade off independently).**

| Axis | Question | Measure |
|---|---|---|
| Coverage (set) | right things changed? | precision / recall / F1 of `set(P)` vs `set(G)` — file granularity headline, hunk/line as the floor |
| Order (sequence) | in the right order? | on shared elements `P ∩ G`: **Kendall τ** (rank agreement) **+** normalized **LCS** = \|LCS(P,G)\| / max(\|P\|,\|G\|) |

Macro-average each over tickets and report the **(coverage-F1, order-τ) pair** — you can hit the right files in the wrong order, or a correct subsequence that misses files. If one scalar is needed: `D = 1 − [½·F1 + ½·(τ+1)/2] ∈ [0,1]`, where `0` = identical to golden.

**Where order actually bites (measured).** Order is informative when the fix is multi-step (≥2 files or ≥2 commits) — and on SPARK that is **~80% of linked fixes** (sampled: median 2 / mean 3.9 files per fix; 17% multi-commit, 1,586 / 9,557). Only the **~21% single-file single-commit** fixes have a vacuous order axis and score on **coverage only**. We report the order-informative share and compute τ over that subset, so it is not diluted by the trivially-ordered minority.

**Why per-ticket internal order, never cross-ticket.** We measured the global resolution order on the real data: created-order vs resolved-order is **Kendall τ ≈ 0.965** (median fix lag 4 days) — the "which ticket got fixed first" order is ~96% determined by the dataset, not by our system. Scoring it would measure Spark's triage habits, not the CMS. So discrepancy is computed on **each ticket's internal change sequence** only.

**Comparison across arms.** The discrepancy (its coverage and order components, and the scalar `D`) is the paired metric across the **no-CMS / full-dump / CMS-over-MCP** ablations — the headline being whether CMS-served context lowers `D` (more of the right files, in more of the right order) versus the baselines.
