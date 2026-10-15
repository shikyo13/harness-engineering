# High-leverage harness engineering for smaller local coding models

**Research synthesis — 2026-10-15. Status: literature-backed technical recommendations and unrun experiment protocols.**

> **Central hypothesis (not yet tested in Zero Code):** The largest tractable capability gains for a fixed, smaller local coding model may come from (1) acquiring the right evidence, (2) intervening only when a trajectory shows measurable failure signals, (3) verifying completion against external state, and (4) retaining only harness changes that improve held-out outcomes. Successful interventions may later become training examples that reduce the cost of the runtime harness.

This report translates the completed October 2026 Deep Research investigation into **research about coding-agent harness engineering**, not a repository-maintenance plan. All numerical results below are **reported by external researchers** unless explicitly labeled otherwise. No new local inference, fine-tuning, or end-to-end coding benchmark was run for this contribution. The exact *future* checkpoint, quantization, engine, hardware and evaluation budget are unspecified; the repository's historical Qwen/M4 Max setup must not be silently substituted.

## What is already covered—and what is new

The existing [2026-10-14 literature survey](2026-10-14-proven-advances.md) already lists many of the underlying papers, and the [subsystem studies](../studies/2026-10-14-subsystems/) propose a target harness architecture. The first two research contributions now on main cover [tool-call protocol reliability](2026-10-14-local-tool-protocol-reliability.md) and [revision-bound *within-task* execution memory](2026-10-14-execution-state-evidence-memory.md).

The **new contribution here** is the decision-oriented synthesis and falsifiable, cross-cutting experimental program:

1. **Adaptive supervision + grounded completion**: which monitor signals and intervention schedules buy *verified* task success for a small local worker, and when does a reviewer harm an otherwise-correct trajectory?
2. **Repository context acquisition + *cross-task* memory**: does improving file retrieval actually improve patch success, and can small, verified repository facts beat both raw history and no memory?
3. **Regression-gated harness evolution**: which editable runtime surfaces produce held-out improvements, rather than benchmark overfitting or prompt bloat?
4. **Harness-to-model distillation**: can intervention-corrected, externally verified trajectories teach a 7B–14B model to need less runtime supervision?

Detailed studies: [supervision](2026-10-15-adaptive-supervision-verification.md), [repository context/memory](2026-10-15-repository-context-memory.md), [self-evolution](2026-10-15-regression-gated-harness-evolution.md), [distillation](2026-10-15-harness-to-model-distillation.md). Common evaluation design: [local harness research program](../evaluation/methods/2026-10-15-local-harness-research-program.md).

## 1. Comparative evidence matrix

| Mechanism | External result (reported by authors; not replicated here) | What it supports | What it does **not** establish |
|---|---|---|---|
| [LivePlan](https://arxiv.org/abs/2608.06701) | Gains up to **15.2 percentage points** across reported coding-agent settings; on MiniMax-M2.5, **+5.45 pp SWE-bench Pro** and **+5.0 pp Verified** | Deterministic drift detection plus triggered advice can rescue coding trajectories | That a cheap **local** adviser has the same gain; that intervention is harmless |
| [Cadence](https://arxiv.org/abs/2610.12269) | **+25.33 pp** mini-swe-agent and **+15.67 pp** Moatless on 300 SWE-bench Lite tasks; up to **29.93% fewer tokens** than a monitor baseline | Monitor *frequency* and advice-vs-replacement policy matter | That the gain transfers from the paper's stronger monitor configurations to a 7B–14B local adviser |
| [Teaching Agents to Code Reliably](https://arxiv.org/abs/2610.03984) | Differential patch verification and learned verifier results reported for 7B/14B/30B; a test-time variant reports **52.8% resolution** at **48.1% of the steps** of an eight-sample comparator | An agent's own passing tests are insufficient; reverted-tree negative controls can improve discrimination | That every issue admits a simple revert-based test; that the verifier is an independent hidden oracle |
| [Agent Retrieval Bench](https://arxiv.org/abs/2607.24882) | **427 samples / 25 repositories**; some logged agents never touch a gold file on about **27–35%** of cases; controlled retrieval seeding improves file-selection F1 by about **0.075** | Repository evidence acquisition is a measurable bottleneck | **Not** patch success: the seeding experiment omits editing/testing |
| [VibeMemBench](https://arxiv.org/abs/2609.23570) | **111 targets / 90 repositories / 3,634 historical trajectories**; direct useful-experience injection improves four of five solvers **+1.1–4.5 pp**, yet **11 of 12** actual memory-system/solver pairings fail to beat memory-off | Useful cross-task knowledge exists, but memory packaging/retrieval can negate its value | That adding vector memory or larger context is a default win |
| [Self-Harness](https://arxiv.org/abs/2606.09498) | On its 100-case subset, Qwen3.5-35B-A3B **19.5→41.5%** overall; held-out **18.2→39.4%** | A frozen model can improve substantially through failure-mined harness changes and regression gates | That the gain generalizes to a new task distribution or a different local model without validation |
| [Agentic Harness Engineering](https://arxiv.org/abs/2604.25850) | Terminal-Bench 2 **69.7→77.0%**; frozen harness transferred **+5.1–10.1 pp** to three other model families | Tools, middleware, memory and instrumentation can matter more than prompt-only changes | That component gains add linearly or that test-set-guided optimization is unbiased |
| [HarnessEvolve](https://arxiv.org/abs/2609.00829) | Reports stronger evolution with reference-guided error diagnosis and explicit quality/performance gates | An optimizer must be constrained by evidence and held-out gates | A guaranteed safe self-editing loop |
| [Harness-Zero](https://arxiv.org/abs/2609.24974) | Qwen3.5-9B macro-average: **23.3%** base/minimal, **41.7%** base/specialized harness, **44.3%** LoRA student/minimal harness | Some harness-induced behaviors can be transferred into model weights | A coding-agent SWE-bench gain: the reported tasks include spreadsheets, AppWorld and patent workflows |
| [Empirical harness design study](https://arxiv.org/abs/2609.20804) | Controlled study over **four models and 176 matched settings** finds model-dependent planning, action-space and context-management effects | Harness policy should be model-conditional | A universal ranking of harness features |

**Evidence discipline.** All percentage-point changes are within the source's own experimental design. Do not compare rows as if they used identical models, task distributions, hardware or verification. Some are preprints and single-run results; independently reproduce promising effects before promoting defaults. The repository's existing local evaluations are separate evidence and should not be combined with these numbers into a new “leaderboard.”

## 2. Causal research architecture

The useful synthesis is a *closed experimental loop*, not a maximal feature bundle:

~~~text
Task + frozen repository
  → initial retrieval / compact verified repo knowledge
  → fixed local worker and tool interface
  → deterministic failure signals
  → optional same-model / local-model adviser
  → externally grounded completion check
  → immutable trajectory + outcome evidence
  → failure clustering + bounded harness proposals
  → held-out paired regression gate
  → accepted runtime policy (or rejected candidate)
  → [optional] corrected, verified training examples → LoRA student
~~~

Every arrow is a potential failure boundary with its own observable metrics. An intervention may improve call-level validity while reducing task-level resolution; a memory hit may be relevant but stale; a new harness may raise pass@1 while doubling wall time; a student may imitate a successful trajectory but regress on a new repository. **No single aggregate score should hide these regressions.**

### A. Adaptive supervision: first experiment

Use deterministic loop/stall signals (repeated unchanged command, repeated tool error, no new file/test evidence, failed external verification, unresolved obligation at premature “done”) and a *budgeted* adviser. Distinguish **advice** from **replacement** (discarding the worker's step or restarting a branch). Compare monitor-off, deterministic-only, fixed-interval review and adaptive-triggered review at equal worker/tool budgets. The source evidence suggests high potential, but the first question is whether a *local* reviewer can help rather than simply consume additional decode time. [Details](2026-10-15-adaptive-supervision-verification.md).

### B. Retrieval and cross-task repository memory: parallel experiment

Keep *cold context acquisition* distinct from *warm cross-task memory* and from [within-task revision-bound evidence](2026-10-14-execution-state-evidence-memory.md). For cold retrieval, measure relevant-file contact, patch localization and executable resolution under lexical, RepoMap, local embeddings and hybrid seeds. For warm memory, compare memory-off, raw retrieved trajectories, oracle-selected helpful experience (diagnostic upper bound), and compact, verified, revision-scoped facts. Never leak an evaluation task's solution patch through its memory source. [Details](2026-10-15-repository-context-memory.md).

### C. Self-evolution: only after the evaluator is trustworthy

Allow an optimizer to change a **manifest of harness policies** (monitor thresholds, verifier checks, context-seed policy, tool-output budgets, retry rules). Disallow changes to hidden tests, scoring, gold data, network/sandbox policy and split assignments. Propose one bounded change at a time, run paired parent/candidate tasks, accept only if held-out quality and efficiency criteria pass and regressions remain controlled. Reserve an untouched shadow/OOD set until final selection. [Details](2026-10-15-regression-gated-harness-evolution.md).

### D. Distillation: only after obtaining verified corrections

Use corrected trajectories from the supervisor/verifier, not arbitrary “success” transcripts. Train a fixed student with LoRA/SFT; compare a four-cell factorial: base/improved harness × base/distilled model. Split training and evaluation by repository, and count training cost separately from deployment cost. The best outcome may be a *thin* harness around the student, not no harness. [Details](2026-10-15-harness-to-model-distillation.md).

## 3. Priority, dependencies and failure modes

| Rank | Program | Reason to run now | Main risk | Exit criterion |
|---|---|---|---|---|
| 1 | Adaptive supervision + differential completion | Existing long-run failures already implicate stalls, false “done” and repeated ineffective actions | Adviser derails successful trajectories | Paired executable success or false-completion improvement at acceptable token/wall cost |
| 2 | Retrieval + cross-task memory | Clear separation between finding files and completing patches; negative evidence warns against naïve memory | Leakage, stale facts, excessive context | End-to-end patch success or strong search-efficiency gain with memory-off control |
| 3 | Regression-gated harness evolution | Can use failure traces and reliable evaluators from 1–2 | Test-set hill-climbing, bloat, reward hacking | Held-out improvement and stable shadow/OOD performance |
| 4 | Harness-to-model distillation | Converts measured runtime improvements into potentially cheaper inference | Training cost, specialization, unverified labels | Student retains material runtime lift on unseen repositories |

**These ranks are engineering judgments, not measured comparative effects.** An expensive strong-adviser ceiling is useful as a diagnostic but should not be required for the local-worker conclusion. Multi-agent expansion, generic skill libraries and more tools are not first-priority experiments unless traces isolate them as failure causes.

## 4. Proposed three-month decision sequence (not a scheduled commitment)

- **Foundation:** freeze task manifests, model/engine hashes, resource budgets and event schema; ensure independent hidden verification and reproducible local baseline.
- **Month 1:** run monitor and completion-gate pilot; inspect resolved→unresolved regressions; retain only the signals that predict actual failure.
- **Month 2:** run retrieval-seeding and memory-off/on A/Bs; then let a bounded self-evolution loop propose changes to *measured* bottlenecks.
- **Month 3:** confirm winning harness policies on shadow repositories; only then curate intervention-corrected data and attempt a modest LoRA/SFT distillation.

The detailed [experiment contract](../evaluation/methods/2026-10-15-local-harness-research-program.md) defines arms, denominators, statistical reporting, guardrails, reproducibility manifests and proposed sample sizes. Engineer-week and GPU-hour estimates should be revised only after target checkpoint, engine, hardware and budget are fixed.

## 5. What this research does **not** prove

- It does **not** show that local Qwen gains the published Cadence/LivePlan improvements, or that a same-model adviser is strong enough to correct its own worker.
- It does **not** show that a memory system improves completion: VibeMemBench provides substantial counter-evidence.
- It does **not** prove that more planning, more subagents, larger context or more tool definitions are universally better.
- It does **not** establish that a benchmark-specific self-evolution loop transfers to new repositories.
- It does **not** demonstrate a trained model or provide a valid local training-cost estimate.
- It does **not** replace the repository's measured 2026-09 and 2026-10 evaluation results.

The intended scientific deliverable is a **ranked set of falsifiable hypotheses** and methods that can turn these plausible mechanisms into local, externally verified knowledge. A negative or null result is a valuable outcome and belongs in \`evaluation/results/\` with the same visibility as a win.

## Primary-source dossier

The main cited sources are original papers or official artifact repositories, not secondary roundups:

- [LivePlan](https://arxiv.org/abs/2608.06701) · [authors' artifacts](https://github.com/Intelligent-CAT-Lab/Agent-Planner/tree/3c72ab7a965e3189123f7721e7cf8dfbd9a67ccb)
- [Cadence](https://arxiv.org/abs/2610.12269) · [authors' code](https://github.com/LogAnalysisTech/Cadence/tree/b5601c2e45931a8493b6889f2ff2646e37ebbf5a)
- [Teaching Agents to Code Reliably](https://arxiv.org/abs/2610.03984)
- [Agent Retrieval Bench](https://arxiv.org/abs/2607.24882) · [benchmark code](https://github.com/eyuansu62/agent-retrieval-bench/tree/07014c986f3deadb1548c62b32c0ffbe6a81465d)
- [VibeMemBench](https://arxiv.org/abs/2609.23570) · [authors' repository](https://github.com/AlibabaResearch/DAMO-ConvAI/tree/main/VibeMemBench)
- [Self-Harness](https://arxiv.org/abs/2606.09498) · [artifact](https://github.com/qzzqzzb/Self-Harness/tree/2720dbb3f52283684f4b85a1065d642df1779dd8)
- [Agentic Harness Engineering](https://arxiv.org/abs/2604.25850) · [artifact](https://github.com/china-qijizhifeng/agentic-harness-engineering)
- [HarnessEvolve](https://arxiv.org/abs/2609.00829), [Evo-Harness](https://arxiv.org/abs/2608.15071), [multi-task self-evolving harness study](https://arxiv.org/abs/2609.38372)
- [Harness-Zero](https://arxiv.org/abs/2609.24974) · [authors' code](https://github.com/metaevo-ai/harness-zero/tree/a4a6223241479c750fdab827c3a449bb1d41b99a)
- [Open-SWE-Traces](https://arxiv.org/abs/2606.16038), [Empirical harness design study](https://arxiv.org/abs/2609.20804)

**Provenance:** synthesized from the completed Deep Research report on 2026-10-15, checked against this repository's current index and prior studies. Linked artifact commits were resolved on 2026-10-15; these links identify reproducible source snapshots but **are not a claim that every implementation file was line-audited or executed**. No third-party source code is copied.
