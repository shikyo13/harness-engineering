# Adaptive supervision and grounded completion

**Status:** research design, not an experiment performed on Zero Code.

## What is new
The existing `studies/2026-10-14-subsystems/autonomy.md` already covers `/goal`, verifier commands, continuation and restart; `multi-agent.md` covers a hidden design-review subagent and its measured quality/time tradeoff. The missing question is whether **event-triggered, selectively invoked advice** is better than fixed-frequency review or a deterministic stall detector alone.

## Evidence
LivePlan (https://arxiv.org/abs/2608.06701) and Cadence (https://arxiv.org/abs/2610.12269) report improvements from monitoring coding trajectories and intervening when necessary. Their results are not measurements of a local Qwen worker, and they do not establish that a same-model reviewer will be useful.

## Proposed runtime
Use a deterministic monitor that observes repeated identical commands, repeated tool failures, lack of new file/test evidence, unresolved acceptance criteria, verification failures and attempts to finish. Do not treat slow local decoding as a stall. Maintain states `healthy`, `suspected_stall`, `unverified_completion`, `escalated`, `recovered`. Advice must include exact evidence and an explicit next-step suggestion. Replacement of the worker's plan or patch requires isolated state or a rollback path; never allow two simultaneous writers by default. Persist each trigger, advice, worker response and eventual verifier result.

## Controlled experiment
Compare five arms on paired frozen tasks: A stock Zero Code; B monitor logging only; C fixed-interval review; D triggered same-model review; E triggered stronger-model review (an upper-bound study, not a local-only solution). Keep worker model, task, tool catalog, token/wall budget, context and hidden external verifier fixed. Record interventions per task, trigger false positives, reviewer-caused regressions, externally verified completion, false-completion rate, cache hits, model tokens and completed tasks per wall-hour.

## Failure tests
Include deliberate loops, false success claims, dependency edits after a passing test, ambiguous tool errors, long silent decodes and tasks already progressing correctly. A reviewer that interrupts correct progress or increases wall time without improving external completion is not a win. If detection helps observability but local advice fails, retain detection and disable automatic intervention. Pre-register trigger thresholds on development tasks; hold out repositories for final testing.

## Decision
First implement a read-only event detector. Only enable intervention after a held-out comparison demonstrates positive *verified completions per wall-hour* and acceptable harm rate. External results are hypotheses, not locally proven effects.
