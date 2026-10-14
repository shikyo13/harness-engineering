# Pi-style tool profile experiment

Date: 2026-09-17. Model: `ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit`.
OpenBench commit: `db193457a3d9128cd4d01fd839c2a890c186c9ac`.

## Result

A Zero Code `fix-failing-test` task was run twice with the normal tool set and twice with a Pi-style coding profile. Each run used a fresh task workspace and the external OpenBench checker. All four runs scored **1.0** and exited with checker code 0.

| profile | tool schema | trial walls | median |
|---|---:|---:|---:|
| full | 21,188 bytes | 18.252 s, 19.926 s | 19.089 s |
| Pi-style coding | 12,673 bytes | 23.138 s, 13.826 s | 18.482 s |

The Pi-style profile removes 8,515 schema bytes (**40.2%**) by withholding tools that are not needed for a basic coding task. The observed median is 0.607 s faster (3.2%), but this two-trial result is too noisy to claim a general speedup. The reliable result is the substantially smaller request payload with no correctness loss in this task.

Raw rows and transcripts are beside this document under `docs/evals/runtime-runs/2026-09-17-optimization/`.

## Decision

Do not replace Zero's desktop default with a restricted tool set yet. The product now exposes the profile as `ZERO_TOOL_PROFILE=coding-minimal`; `full` remains the default, preserving browser/computer access. The rebuilt product profile passed the external checker at 1.0 in 19.042 s and emitted a 10,038-byte tool schema, a 52.6% reduction from the default 21,188 bytes. The same rebuilt binary with the default `full` profile also passed at 1.0 in 19.846 s and emitted the unchanged 21,188-byte schema. The OpenBench runner now accepts `ZERO_TOOL_PROFILE` directly; a post-revert current-binary `coding-minimal` smoke passed at 1.0 in 18.949 s. Next, test the profile on long-context and multi-tool tasks before considering a desktop setting for it.

## Matched multi-file control (2026-09-17)

The same rebuilt binary was then run twice per profile on OpenBench `add-feature` through the repository-owned runner. All four fresh workspaces passed the external checker (`score=1.0`, `completed=true`, `checker_exit=0`):

| profile | trial 1 | trial 2 | median |
|---|---:|---:|---:|
| coding-minimal | 66.346 s | 78.591 s | 72.469 s |
| full | 121.995 s | 260.158 s | 191.077 s |

This strengthens the directional case for the Pi-style profile: the median is 118.608 s lower (62.1%) on this task, with no correctness loss. The transcripts show different model trajectories and the MLX server has shared prefix caching, so this is still not a universal speed multiplier. The profile remains opt-in; the next decisive test is long-context and multi-tool coverage with cold/warm cache separated.

The machine-readable task-level aggregation is
`docs/evals/2026-09-17-pi-profile-add-feature-repeat.json`.

## Long-running task check (2026-09-17)

The same current binary and `coding-minimal` profile completed OpenBench
`taskflow` with a score of 1.0 and checker exit 0 in 291.289 s. That is slower
than the prior Pi median of 165.625 s and Zero's baseline median of 233.785 s.
This is one trial, so it is not a final ranking, but it prevents promoting the
profile as a universal default; longer tasks need their own repeated cold/warm
measurements.

## Prompt transplant check (2026-09-17)

A short Pi-inspired system prompt was tested with the same four-tool profile.
Both runtime rows passed the hidden checker, but `fix-failing-test` took 25.887 s
and `add-feature` took 199.333 s. The short prompt is therefore dropped; Zero's
existing prompt remains in place while the tool-profile change is evaluated on
additional workloads.

## Native streaming seam (2026-09-17)

Zero's opt-in native runtime was extended to accept API-key-backed local OpenAI-compatible providers such as MLX. The native request adapter now preserves provider-scoped options and emits Qwen's reasoning mode as `thinking=true`. A real `fix-failing-test` run passed the external checker (`score=1.0`, `completed=true`, `checker_exit=0`) in 26.411 s. This is a correctness and wire-fidelity result, not a speed win over the AI-SDK default, so native transport remains opt-in pending repeated provider-level measurements.

## Current full-profile long-task control (2026-09-17)

A current packaged binary was run through the same runner with the default full
profile on `taskflow`. It completed all 25 tests with score 1.0 and checker exit
0 in **396.103 s**. The same-day lean row was 291.289 s, while the prior Pi
median was 165.625 s. This confirms that the four-tool profile reduces one
observed Zero trajectory but does not reproduce Pi's long-task performance;
profile selection and runtime trajectory remain separate optimization problems.

## Reasoning-history replay check (2026-09-17)

Pi's OpenAI-completions adapter omits ordinary prior reasoning text when it
rebuilds a context. Zero now has an opt-in equivalent for OpenAI-compatible
models (`ZERO_EXPERIMENTAL_STRIP_REASONING_HISTORY=true`), retaining visible
assistant text and tool history while omitting prior reasoning blocks.

The same current full-profile binary ran `taskflow` once with that flag. The
external checker still passed all 25 tests (`score=1.0`, `checker_exit=0`), but
wall time was **465.144 s**, versus **396.103 s** for the matched full-profile
control and **291.289 s** for the same-day coding-minimal row. This single
stochastic MLX result does not show a speed benefit; the broad flag remains
opt-in and is not promoted as a Pi optimization. A future experiment should
narrow the behavior by model/provider compatibility and collect repeated cold
and warm trials before changing defaults.

## Native streaming taskflow check (2026-09-17)

Zero's opt-in native `@zero-code/llm` path completed the same `taskflow` cell
with the coding-minimal profile in **346.809 s**, score 1.0, checker exit 0.
It was faster than the matched full-profile control (396.103 s), but slower
than the AI-SDK coding-minimal row (291.289 s) and Pi's repeated median
(165.625 s). Native transport remains useful as a correctness-preserving seam,
but it is not currently the Pi speed solution.

## In-memory context conversion check (2026-09-17)

Zero now has an opt-in `ZERO_EXPERIMENTAL_IN_MEMORY_CONTEXT` path that follows
Pi's live-context pattern for OpenAI-compatible models. It caches the converted
message prefix within one run and converts only the new message tail, with
invalidation for model changes, compaction roots, and mutations to prior
message parts. A fresh `make-it-run` cell passed the external checker at 1.0 in
23.137 s and emitted seven main provider requests. The matched non-cached smoke
was 21.952 s, so this single stochastic result does not show a speed win; the
path remains opt-in pending repeated taskflow and multi-file measurements.

The follow-up `add-feature` cell also passed its external checker (`feature 6/6`,
regression groups `4/4`, score 1.0, checker exit 0) with the cache enabled. It
took **279.433 s**, used nine main provider requests, reached a 14,305-token
prompt, and streamed 5,878 output tokens. The result is a valid long-tail
correctness check but provides no speed evidence; the cache remains gated.

## Matched Qwen thinking control (2026-09-17)

Pi's benchmark manifest selects `--thinking medium`. Pi's Qwen compatibility
adapter translates that level to the top-level `enable_thinking=true` field;
it does not send a medium-specific token budget. Zero's Qwen model definition
already enables the same field by default. The runner now supports
`ZERO_MODEL_VARIANT=medium` so this equivalence can be recorded explicitly.

A fresh pinned-variant `make-it-run` row passed the external checker at 1.0 in
21.930 s (five main provider requests, maximum prompt 5,966 tokens). This is
consistent with the existing unpinned 21.952 s smoke and confirms that the Pi
lead is not explained by a lower Qwen reasoning setting.

The optimization target is therefore Pi's execution architecture: a persistent
in-memory context snapshot, a smaller default tool surface for coding tasks,
and fewer repeated orchestration/reload boundaries. Zero should adopt those
ideas behind measured, reversible flags first, then promote only after repeated
cold and warm long-task cells preserve checker correctness.

## Instrumented packaged control (2026-09-17)

The rebuilt packaged binary (`0.0.0-main-202609170850`) completed a fresh
`make-it-run` cell with the explicit `medium` variant and `coding-minimal`
profile in **21.076 s**, score `1.0`, checker exit `0`. The MLX trace captured
five main requests, a maximum 5,956-token prompt, a 10,038-byte tool schema,
and a 95.32% mean prefix-cache hit ratio. The raw result and provider trace
are retained as
`docs/evals/runtime-runs/2026-09-17-optimization/zero-variant-medium-make-it-run-telemetry-built.{jsonl,mlx.json}`.

Zero now emits `llm request prepared`, `session history loaded`, and `session
context assembled` timing events. These fields are diagnostic evidence for the
next desktop and long-task runs; they are not included in wall-time rankings
until the same runtime logger is captured for every candidate.

## Rebuilt taskflow control with provider trace (2026-09-17)

The current packaged binary (`0.0.0-main-202609170857`) was run through the
same pinned `taskflow` cell with `ZERO_TOOL_PROFILE=coding-minimal` and the
explicit `medium` Qwen variant. It passed all 25 hidden tests (`score=1.0`,
`checker_exit=0`) in **369.732 s**.

The provider trace explains why this is still behind Pi. Zero made 16 main
provider requests, accumulated a maximum prompt of 54,681 tokens, streamed
5,980 output tokens, and reached 31 tool messages. Mean MLX prefix-cache reuse
was 86.19% (355,006 reused tokens), so raw prompt prefill was not the only
cost. The transcript contains 16 thinking segments and repeated read/
investigation passes before the same seven fixes were completed.

Pi's repeated taskflow median is **165.625 s** (two strict-pass trials), while
the prior Zero repeat median is **233.785 s** (also two strict-pass trials).
The current 369.732 s run is a diagnostic control from a rebuilt binary, not a
new ranking sample. Together the traces point to the next optimization target:
reduce unnecessary model/tool turns and redundant repository rereads while
preserving the external checker result. A context cache alone cannot be
credited with a speed win until it reduces wall time in repeated long-task
trials.

Raw artifacts:

- `docs/evals/runtime-runs/2026-09-17-optimization/zero-current-taskflow-medium.jsonl`
- `docs/evals/runtime-runs/2026-09-17-optimization/zero-current-taskflow-medium.mlx.json`


## Legacy in-memory context trial (2026-09-17)

The same rebuilt binary was run once with `ZERO_EXPERIMENTAL_IN_MEMORY_CONTEXT=true`, keeping the actual legacy desktop execution path and the same `coding-minimal`/`medium` settings. It passed all 25 hidden tests (`score=1.0`, `checker_exit=0`) in **418.116 s**. The provider trace contained 13 main requests, a 57,254-token maximum prompt, 7,666 streamed output tokens, and 29 tool messages, with 82.78% mean prefix-cache reuse.

This is slower than the 369.732 s current control and does not justify enabling the flag by default. It confirms Pi's advantage is not explained by message-conversion caching alone; the higher-value target is reducing unnecessary reasoning/tool turns and redundant repository reads.

Raw cache trial artifacts:

- `docs/evals/runtime-runs/2026-09-17-optimization/zero-current-taskflow-cache.jsonl`
- `docs/evals/runtime-runs/2026-09-17-optimization/zero-current-taskflow-cache.mlx.json`


## Compact coding prompt trial (2026-09-17)

A separately gated compact coding prompt reduced Zero's system payload from
about 9.6 KB to 2.1 KB on the same `coding-minimal`/`medium` setup. The fresh
`make-it-run` cell passed the external checker (`score=1.0`,
`checker_exit=0`) but took **30.225 s**, versus the 21.076 s instrumented
current control. This one trial is stochastic, but it provides no promotion
evidence; the compact prompt remains opt-in and the normal Zero prompt stays
the product baseline.

Artifacts:

- `docs/evals/runtime-runs/2026-09-17-optimization/zero-compact-make-it-run.jsonl`
- `docs/evals/runtime-runs/2026-09-17-optimization/zero-compact-make-it-run.mlx.json`

The switch was then routed through `RuntimeFlags` and the rebuilt packaged
binary (`0.0.0-main-202609170926`) passed the same checker cell at **27.133 s**.
This validates the gated path after the refactor; it does not alter the
promotion decision.
