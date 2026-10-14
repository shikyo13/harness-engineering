# Experiment design

## What is held fixed

Everything in `MODEL_FINGERPRINT.json`: weights, quantization, backend build, launch
arguments, context limit, chat template, tokenizer, hardware. The harness, or one named
harness component, is the only thing that varies within a comparison.

One thing cannot be held fixed and is therefore recorded instead: sampling parameters.
Harnesses ship their own defaults and Track A measures harnesses as products. See D5.

## Two tracks, kept apart

**Track A, real-world comparison.** Each harness runs as its authors ship it: its own
system prompt, tool set, context policy, compaction, editing mechanism, retries. The
package is the treatment. This answers which harness makes this model perform best. It
does not by itself explain why, and no causal claim is drawn from it.

**Track B, controlled ablation.** One agent loop, one prompt, one task set, with exactly
one architectural property changed at a time. Candidate arms: minimal primitives (read,
write, edit, bash), bash-centric, hybrid (read, patch, bash, search), rich structured
tooling. Which arms actually get built is decided by what Track A's failures point at, not
in advance.

## Stages and budget

Total ceiling roughly 150 to 220 full agent runs. It is a ceiling, not a quota. Each stage
states what it buys.

| Stage | Purpose | Scale |
| --- | --- | --- |
| 0. Qualification | prove each harness reaches the exact model and can read, search, write, edit, run a command, see it fail, recover, handle long output, keep state across turns | 15 to 25 runs |
| 1. Broad screen | 4 to 6 discriminative tasks, one trial per cell, to find failure modes and outliers | about 52 runs |
| 2. Adaptive replication | more trials only where the answer would change a decision | 25 to 50 runs |
| 3. Held-out comparison | Zero Code plus 3 to 5 informative competitors on a fresh task set | 50 to 80 runs |
| 4. Zero Code experiments | focused hypotheses drawn from observed failure patterns | as needed within the ceiling |

Stage 0 is not a performance measurement. Its only job is to catch broken adapters,
protocol translation damage, and instrumentation that is not recording what it claims.

## Measurement

Three clocks, because they disagree and the disagreement is informative.

1. **Engine clock.** mlx-serve reports `prompt_n`, `cached_n`, `prompt_ms`, `predicted_n`,
   `predicted_ms`, `tokenize_ms` on the final streaming chunk. This is the only source for
   the prefill and decode split.
2. **Proxy clock.** A metering proxy in front of the engine timestamps every request and
   response, and records what each harness actually sent: system messages, tool
   definitions, tool choice, sampling parameters, limits, and the model it asked for.
3. **Harness clock.** Whatever the harness reports about itself. Treated as telemetry, not
   as truth.

Token counts come from the proxy for every arm, so they are counted the same way
regardless of what a harness chooses to report.

## Failure classification

A shell command exiting non-zero is not a failed tool call. The two are counted separately.

- **Harness or protocol failure:** malformed tool JSON, schema mismatch, unknown parameter,
  invalid tool name, parser failure, router failure, harness exception.
- **Command or domain failure:** the call was well formed and the world said no. Compiler
  error, failing test, empty grep, merge conflict, wrong output.

Derived measures, defined once here and computed the same way for every arm: tool protocol
failure rate, longest failed-action streak, exact-repeat events, failure-loop episodes
(three or more equivalent failed actions with no new observation, no useful mutation and no
changed strategy), recovery distance, unrecovered tool failures, and harness friction time.

## Scoring

External and deterministic. A checker decides, never the agent. A task is admitted only
after a negative control (untouched workspace must fail) and a positive control (a known
good solution must pass). Requested behaviour is tracked separately from test counts, so a
run cannot pass by adding or deleting tests.

## Execution discipline

Harness order is randomised or blocked within tasks, one model-heavy run at a time unless
concurrency is itself the variable, and the engine's prefix cache state is recorded per run
because a warm cache changes prefill by two orders of magnitude on this machine.

Repeats of one task are stochastic replicates, not independent tasks. Uncertainty across
tasks uses a task-clustered bootstrap; paired binary outcomes use matched task comparisons.
Where the sample cannot separate two harnesses, the report says unresolved at this sample
size rather than equal.
