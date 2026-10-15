# Execution state as a verifiable, revision-bound memory system

**Research synthesis — 2026-10-14. Evidence: papers and one pinned codebase; not measured on Zero Code.**

This study addresses a question not resolved by the existing [context-engine](../studies/2026-10-14-subsystems/context-engine.md), [agent-kernel](../studies/2026-10-14-subsystems/agent-kernel.md) or [autonomy](../studies/2026-10-14-subsystems/autonomy.md) surveys: **How can a coding agent know whether an observation, test result or completed obligation is still true after hundreds of tool calls, file changes, context refreshes and restarts?** The answer is not simply to summarize transcripts, enlarge context or attach vector memory. Source-backed mechanisms suggest using *stateful evidence with explicit validity and supersession relations*. The combined proposal here is original and has not been benchmarked.

## Key distinction: history is not current state

Suppose an agent reads `src/auth.ts`, confirms a test passes, edits a dependency, compacts its history, and later retrieves “auth test passed.” The statement may be an accurate historical record but an **invalid current verification**. A semantic-search index ranks relevance, not applicability. A stale summary can be worse than forgetting, because the model now acts confidently on incorrect state.

An execution-memory layer should distinguish:

- **Observation:** what file content or command output was actually observed, when, and in what workspace revision.
- **Claim:** what conclusion was inferred from an observation, and which evidence supports it.
- **Work item:** an obligation or decision, with dependencies, done predicate and status.
- **Verification:** a test/check result tied to an artifact revision and the conditions of execution.
- **Supersession:** an event or newer result that explicitly invalidates, revises or replaces earlier evidence.
- **Historical evidence:** still retrievable for explanation or debugging, but *never represented as current* until revalidated.

This introduces an additional axis beyond memory retrieval quality: **evidence freshness / task-state consistency**.

## 1. Three approaches compared using primary sources

| Approach | What is stored | What happens before the next model decision | What happens before action / restoration | Published result (authors' reports) |
|---|---|---|---|---|
| **Ledger** [Wang et al. 2026](https://arxiv.org/abs/2608.00808) | deterministic observations, modifications, commands and freshness counters | short regenerated “inform” state view, not an accumulating prompt fragment | “govern” reuses still-visible safe read results or nudges likely repetitions; does not block arbitrary edits | SWE-bench Verified, 500 cases: GPT-5 mini 56.2→64.2%; MiniMax M2.5 75.8→81.0%, with 28.9%/31.8% lower total cost |
| **MemTrace** [Xu et al. 2026](https://arxiv.org/abs/2610.04838) | immutable evidence traces, a trace graph (dependencies, verify, correct, supersede), repository-state graph, compact anchors | reconstructs execution frontier on context refresh | validates restored trace against current repository state and whether it was superseded | Codex CLI/DeepSWE 35.4→56.6% pass@1; SWE-EVO 65.2→69.6% vs best baseline; SWE-Milestone Score 28.6→46.4% vs native |
| **StructAgent** [Wu et al. 2026](https://arxiv.org/abs/2607.11388) | structured, verifier-backed state and workflow transitions for digital tasks | grounded compact task-progress state | verifier-backed step transitions and recoverable progress | OSWorld-Verified with Qwen3.5-9B 27.0→46.9%, 27B 31.6→62.2% (desktop/computer use, **not** coding benchmark) |

**Do not add these improvements.** Different tasks, models, baselines, harnesses and evaluation policies make their gains noncomparable, much less additive. Papers report single-author/independent-team results; these are not replications on a local M4 Max.

### 1.1 Ledger's most useful ablation

The Ledger paper separates **inform** (what the model is told about current state) from **govern** (what commands are mediated). For MiniMax M2.5 on 500 SWE-bench Verified instances: baseline 75.8%; inform-only 77.0%; govern-only 80.4%; combined 81.0%. Its authors report that inform accounts for most of the token/call reductions while govern produces much of the completion gain. For GPT-5 mini the pattern is less separable: inform-only 64.0%, govern-only 64.2%, combined 64.2% against 56.2% baseline. Exact numbers and caveats: [paper, ablation table and limitations](https://arxiv.org/abs/2608.00808).

**Mechanism detail:** Ledger's Reuse only references earlier output **still present in the model's context**; it does not hallucinate a result that has disappeared. Test commands get a *nudge*, not a suppression, even with unchanged code because rerunning them may be intentional. Unhandled commands default to Allow. These are valuable safeguards, but they mean Ledger alone is not a durable post-compaction historical retrieval system.

**Inference:** the no-extra-model-call nature of this layer is unusually appealing when local inference is slow or decode-bound. The local speed benefit remains a hypothesis until measured in tokens and wall time.

### 1.2 MemTrace's complementary result

MemTrace tracks *immutable evidence plus relations* and grounds evidence against the current repository. Its published Codex DeepSWE ablation progresses from a trace store alone (32.1% pass@1), to trace store + Memory Trace Graph (44.2%), to full graph + repository state graph (56.6%). The same repository graph adds no improvement to the SWE-EVO metric in that ablation (69.6% with or without it): graph cost is **workload-dependent**. See [`4.3, component ablation](https://arxiv.org/abs/2610.04838).

In the authors' analyzed traces, 5,261 of 9,226 (57.0%) file-bound traces were exposed to at least one later modification of a referenced file. This is a measure of **exposure**, not proof that 57% became semantically incorrect. The authors use a more selective applicability check that considers code/test dependencies and supersession.

**Primary code read:** the public [MemTrace repository](https://github.com/Homy-Xu/MemTrace/tree/5508f8be81025d8aa90726b1e06aef4ea484e577) contains runtime, Codex and mini-swe-agent backends, benchmark adapters and tests. Its [`src/memtrace/harness/revision.py`](https://github.com/Homy-Xu/MemTrace/blob/5508f8be81025d8aa90726b1e06aef4ea484e577/src/memtrace/harness/revision.py) defines workspace revision receipts, changed paths and symbol bindings; [`src/memtrace/recall/assembler.py`](https://github.com/Homy-Xu/MemTrace/blob/5508f8be81025d8aa90726b1e06aef4ea484e577/src/memtrace/recall/assembler.py) maps evidence fact types to restored code changes, tests/failures, decisions and unresolved work. The README says the published source includes test and receipt schemas but does **not** claim completed external reproduction for SWE-EVO or SWE-Milestone. Source is licensed Apache-2.0; this document proposes design rather than copying code.

**Inference:** repository-aware validation belongs at the *evidence replay* boundary, not merely when context is compacted; the same stale test could be reused even before any compaction.

### 1.3 StructAgent adds a control concept, not a coding result

StructAgent uses structured execution state and verifier-backed progress transitions in long-horizon desktop tasks. Its Qwen3.5-9B and 27B improvements on OSWorld-Verified are an existence proof for this kind of *externalized execution state* outside source-code editing—not evidence of equivalent gains on local software engineering tasks. A coding implementation should inherit the **verification-backed state transition principle**, not simply copy its OSWorld workflow.

## 2. Proposed unified mechanism: Evidence-Bound Runtime (EBR)

**Original synthesis, not an upstream implementation.** Do not build the full graph by default. Build the *smallest dependable runtime* first and make deeper graph relationships optional.

### 2.1 Event stream + projections

Maintain an append-only event stream as the durability source. Deterministically project:

1. **Workspace revisions:** a monotonically incrementing revision ID + changed files + optional symbol-level fingerprints; watch for out-of-harness edits and git checkout/merge events.
2. **Observations:** file/range/read hash, repository revision, source command, timestamp and whether still accessible in live context.
3. **Execution obligations:** canonical task ID, acceptance predicate, dependency IDs, attempted action IDs, verifier receipts and unresolved status.
4. **Evidence references:** identifier, provenance, scope, supporting result, relevant file/symbol/test dependencies, validity interval, relation to newer evidence.
5. **Context working set:** compact addresses to evidence and current obligation frontier, with full text stored separately.

A minimally sufficient schema:

~~~text
Evidence {
  id, kind: read | test | diagnosis | edit | user_constraint,
  event_id, workspace_revision, repo_identity,
  target_paths, symbol_ids?, dependency_fingerprints?,
  claim?, result_ref, result_visibility: in_context | offloaded,
  validity: current | stale | unknown | superseded | historical,
  superseded_by[], verified_by[], created_at
}
Obligation {
  id, description, dependencies[], done_predicate, status,
  verifier_evidence_ids[], failure_evidence_ids[],
  last_checked_revision, next_action_hint?
}
~~~

**Important:** `current` means the runtime verified the *recorded applicability preconditions*, not that an arbitrary semantic claim is philosophically true. Avoid fabricating certainty; `unknown` should trigger fresh reads/tests.

### 2.2 Read-time and action-time guards

- **Before prompt assembly (inform):** supply the current objective, unsatisfied obligations, changed file summary, verified claims still applicable, stale/unknown notices, and the next executable frontier. Regenerate, rather than append new state views into history; this also reduces unstable-prefix cache churn.
- **Before tool dispatch (govern):** use exact command semantics and known state to choose `allow`, `reuse`, `nudge`, or `require_revalidation`. Never reuse mutating commands by default. Match hash + workspace version + output visibility, or return a properly validated offloaded artifact.
- **Before compaction / restart (checkpoint):** persist a frontier snapshot and immutable evidence references. After restart, compare **actual working-tree state** with the snapshot, replay subsequent durable events, and only then restore evidence. A successful saved test at old revision is not today's pass.
- **After mutations:** mark overlapping observations and downstream verifications `stale` or `unknown`; preserve the originals for audit. Use *conservative* invalidation until dependency mapping proves it can be selective.
- **Before “done”:** consult external predicates (tests, expected changed artifacts, configuration and user constraints) and verify against a defined workspace revision. If the repository changed after a verifier ran, do not silently treat the prior result as fresh.

### 2.3 Two-phase rollout to reduce complexity risk

**Phase A — deterministic observation ledger.** No graph database, no LLM subagents, no new embeddings. Implement revisions, file/range observations, safe redundant-read suppression, short state display, and proof of unchanged external behavior. Compare with baseline.

**Phase B — provenance-backed evidence recovery.** Add immutable test/diagnosis/edit receipts, explicit `supersedes` and `verifies` relations, bounded artifact retrieval and invalidation after edits. Only if Phase A succeeds should we attempt symbol-level dependency graphs. Choose representative workloads where symbols matter; MemTrace's SWE-EVO ablation warns against paying full graph cost everywhere.

## 3. Failure modes a glossy design misses

| Failure | Countermeasure | What to measure |
|---|---|---|
| Out-of-band file edits make cached read “current” | workspace observer + git status and file/hash reconciliation, conservative unknown on missed events | undetected stale-use rate |
| Partial read reused as a full file | record byte/range coverage and output visibility | false reuse rate on overlapping ranges |
| Test result tied only to edited file, not dependencies/config | fingerprint test source, known imports and environment; otherwise unknown | invalidated-pass detection |
| Reuse saves tokens but blocks deliberate re-observation | nudge/allow escape hatch, never suppress mutating commands | newly failed tasks and repair loops |
| Memory graph spends more compute than it saves | defer graph construction; bound lookup traversals and payloads | passes/wall-hour and latency per recovered result |
| Compaction preserves objective but loses next action | explicitly persist execution frontier, blockers and unsatisfied done predicates | resumed task completion |
| Multiple agents edit same workspace | revision receipts/worktree isolation per writer; merge invalidates evidence | cross-agent stale collisions |
| Verification itself modifies state | classify stateful commands; never cache as safe reads | side-effect misclassification |

## 4. Falsifiable experiment plan

Use the repository's [same-model evaluation protocol](../evaluation/methods/2026-09-16-harness-comparison.md) and a held-out fixture set. Test at least three different horizons (e.g., short / medium / compaction-forcing) and at least two intervention-trigger distributions; randomize paired task order to avoid hardware or cache drift.

**Arms:** (A) stock context; (B) plain concise progress summary; (C) deterministic read ledger only; (D) ledger + safe action guard; (E) D + revision-bound evidence receipts; (F) E + symbol dependency projection.

**Interventions:** after a passing test change its direct or indirect dependency; edit a previously read file outside the agent; perform two equivalent reads under unchanged state; force compaction before the next operation; restart mid-task; intentionally supersede an earlier diagnosis; simulate a failed test followed by passing test. The task evaluator must be outside the agent's writable workspace.

**Measure:** true end-state correctness, stale-evidence consumption, repeated reads, mistaken suppression, correctly invalidated verifiers, completion after refresh/restart, prompt/cache tokens, generation time, wall time, disk/memory overhead, and failure modes. A repair-count reduction with no measured improvement in final-state success should be reported as efficiency only.

**Primary gates:**
- Phase A does not introduce wrongful read suppression or stale-state false positives.
- Phase B's benefit survives external code changes and compaction in unseen tasks.
- Compare gains in completions per wall-hour, not merely benchmark pass@1 or prompt-token count.
- If full repository graphs add no meaningful held-out gain, retain a lightweight revision ledger.

## 5. What remains unknown

1. Ledger's authors explicitly note one run per benchmark instance and Python issue-style tasks; generalization to large monorepos or other languages is not demonstrated.
2. MemTrace experiments use deepseek-v4-flash-0731-high with 200K context, not a 27B local Qwen with severe decode constraints. Their work provides an implementation baseline, not directly transferable cost figures.
3. How often conservative dependency invalidation over-invalidates useful evidence on a real coding project is not yet established for the target harness.
4. Whether a persistent semantic memory over *multiple sessions* improves completion beyond single-task trace recovery, without spreading obsolete claims, remains open.
5. Which evidence types should survive a model upgrade, model/engine change or repository rebase needs separate transfer evaluation.

## Evidence and source provenance

- [Ledger paper, 2026-08-01](https://arxiv.org/abs/2608.00808): design, results, inform/govern ablations and limitations.
- [MemTrace paper, 2026-10-04](https://arxiv.org/abs/2610.04838): method, graph ablations, stale-evidence exposure data and limitations.
- [MemTrace public source pinned to `5508f8be`](https://github.com/Homy-Xu/MemTrace/tree/5508f8be81025d8aa90726b1e06aef4ea484e577), Apache-2.0. Inspected README, LICENSE, revision and recall assembler implementation; **not** locally executed.
- [StructAgent paper, 2026-07-13](https://arxiv.org/abs/2607.11388): structured/verifier-backed digital-agent runtime, not coding evaluation.
- [AMA-Bench, ICML 2026](https://proceedings.mlr.press/v306/zhao26bs.html): causal/objective information loss in agent-trajectory memory; potential evaluation baseline.

All numeric gains above are **reported by the cited authors**; none are represented as newly observed improvements in Zero Code.
