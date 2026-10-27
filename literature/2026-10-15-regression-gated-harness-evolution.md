# Regression-gated self-improvement of coding harnesses

**Status:** proposed optimization methodology; no autonomous improvements measured here.

## Evidence and existing work
Self-Harness (https://arxiv.org/abs/2606.09498), Agentic Harness Engineering (https://arxiv.org/abs/2604.25850), and HarnessEvolve (https://arxiv.org/abs/2609.00829) report harness adaptation. Their reported gains use different models and tasks; none proves a universal optimum. Zero Code already has a controlled evaluation framework and subsystem architecture. The novel contribution is a **bounded change-selection protocol** that treats benchmark overfitting as a first-class failure.

## Editable search space
Version a typed policy manifest covering tool-description length, output truncation, retry budget, loop-break threshold, context-seeding policy, monitor escalation, and verification cadence. Permit one change at a time initially. Never let the optimizer modify hidden tests, evaluator code, gold patches, sandbox permissions, task splits or scoring rules. Save every candidate as an immutable parent hash plus policy diff.

## Evolution cycle
(1) Cluster failures using external verifier and tool telemetry; (2) state a falsifiable mechanism; (3) generate a minimal policy edit; (4) check schema and safety invariants; (5) compare candidate with parent on paired development tasks; (6) confirm on distinct validation repositories; (7) promote only when preregistered quality and efficiency gates pass; (8) finally evaluate on untouched shadow/OOD repositories. Preserve rejected candidates and negative results.

## Metrics
Externally verified task completion, paired per-task changes, confidence intervals, completions per wall-hour, token/cache cost, false completion, unauthorized/duplicate tool effects, and worst-case per-task regressions. Record every model, engine, quantization, template, hardware and evaluator revision. Account for multiple comparisons: repeated validation-set peeking makes a seemingly strong candidate less credible.

## Falsifiers
A development-set win that disappears on unseen repositories is overfitting, not a proven harness improvement. A higher pass rate purchased by doubling compute is not an efficiency gain. A policy that works only on one checkpoint should be marked model-specific, not silently installed as the default. Stop self-editing if safety or verifier integrity is compromised.

## First implementation
Run a narrow tool-error-recovery policy search on a frozen local-model baseline. Do not introduce a large self-editing agent until a simple manually controlled search demonstrates stable held-out gains.
