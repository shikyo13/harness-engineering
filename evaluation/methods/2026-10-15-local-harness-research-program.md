# Local harness research program: reproducible experiment contract

**Status:** proposed methods; no new trials performed. This file completes the fifth missing reference from PR #3.

## Baseline and task splits
Reuse `evaluation/methods/2026-09-16-harness-comparison.md` and `surveys/2026-10-13-harness-survey/06-EXPERIMENT_DESIGN.md` rather than replacing their measured results. Freeze repository snapshots and task manifests. Create repository-disjoint development, validation and untouched shadow/OOD splits. Include short, medium and compaction-forcing tasks; explicit and implicit requirements; tool-error, dependency-change and restart fixtures. Keep hidden evaluators outside the agent's writable workspace.

## Required per-run manifest
```yaml
run_id: REQUIRED
task_manifest_sha256: REQUIRED
repository_snapshot_sha: REQUIRED
split: dev|validation|shadow
model_checkpoint_sha: REQUIRED
tokenizer_and_template_sha: REQUIRED
quantization: REQUIRED
engine_name_and_revision: REQUIRED
harness_commit_and_policy_sha: REQUIRED
hardware_and_power_state: REQUIRED
context_limit: REQUIRED
sampling_parameters: REQUIRED
seed: REQUIRED
tool_catalog_sha: REQUIRED
max_wall_seconds: REQUIRED
max_model_tokens: REQUIRED
max_tool_calls: REQUIRED
verifier_revision: REQUIRED
```
Missing values are unknown, not zero. Capture cached prompt tokens, prefill and decode timings, host memory pressure, failure reasons and final-state verifier receipts.

## E1 — Adaptive supervision
Arms: stock, detector-only, fixed-interval review, triggered same-model review, triggered stronger-model review. Track trigger precision/false alarms, reviewer harm, verified resolution and wall-hour efficiency. No extra budget for intervention arms unless explicitly analyzing a compute tradeoff.

## E2 — Repository retrieval and cross-task memory
Retrieval arms: lexical, symbol/RepoMap, embedding, hybrid. Memory arms: off, raw transcript, verified concise facts, oracle-selected upper bound. Run retrieval-only and end-to-end coding evaluations separately. Prevent solution leakage through memory. Include stale-fact and irrelevant-similar-memory negative controls.

## E3 — Harness policy evolution
Compare fixed baseline, manually tuned policy and bounded optimizer candidates. Optimize on development tasks, select on validation, evaluate final candidates once on shadow tasks. Every candidate is an immutable policy diff; no modification of evaluator or gold data. Count all failed search runs in optimization cost.

## E4 — Harness-to-model distillation
Four-cell model×harness factorial: base/base, base/improved, distilled/base, distilled/improved. Train only on verified, licensed, repository-disjoint trajectories. Record training energy/GPU-hours and amortization; do not present training loss as coding capability.

## Statistics and decisions
Use paired task-level differences with bootstrap intervals stratified by repository. Report absolute numerator and denominator, timeout and abort counts, per-task wins and regressions, and exact run manifests. Primary outcome: independent verified completions per wall-clock hour. Secondary: pass@1, total tokens, cache hits, tool-call validity, false completion, memory/retrieval overhead, and energy. Hard-fail on unsafe tool effects or evaluator tampering. Treat uncertain differences as inconclusive. Archive null and negative findings under `evaluation/results/`.

## Provenance
Companion technical studies: `../../literature/2026-10-15-adaptive-supervision-verification.md`, `../../literature/2026-10-15-repository-context-memory.md`, `../../literature/2026-10-15-regression-gated-harness-evolution.md`, `../../literature/2026-10-15-harness-to-model-distillation.md`. All proposed, not measured.
