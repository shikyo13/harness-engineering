# Harness comparison protocol

This benchmark compares Zero Code, Qwen Code, OpenCode, Pi, and Aider using one local Qwen 3.8 MLX server at a time. No harness or model process may overlap another run on the 128 GB host.

## Runtime contract

Each harness receives the same fresh fixture directory, task prompt, model endpoint, and timeout. Before a run, the runner records the harness version, model identifier, engine version, git commit, working-tree state, machine, and fixture hash. After a run it records wall time, time to first output, input/output/cache tokens when exposed, tool-call count by tool, changed files, verification exit code, final status, and the complete raw transcript.

A run is valid only when the harness completes its own runtime loop and the fixture verifier passes. Compilation, typechecks, startup success, or a generated plan are not benchmark success. A timeout, tool error, or missing metric remains a failed or incomplete result and is never silently converted to zero.

## Task tiers

1. **Speed:** exact short answer and 16k/64k/128k context probes.
2. **Coding:** fix a seeded bug, add tests, and pass the fixture verifier.
3. **Long task:** multi-file change with an explicit test command and bounded output.
4. **Tool/browser:** inspect a local page, modify the fixture, and prove the result.
5. **Recovery:** interrupt during a tool call, restart the harness, and resume without duplicate edits.

Run each tier once per harness initially. Repeat only when the run is invalid or the protocol itself changes. Store one JSON result per run and a Markdown aggregate with pass/fail and missing-data reasons.

## Bootstrap matrix

| harness | required runtime setup | current state | acceptance gate |
|---|---|---|---|
| Zero Code | packaged desktop/CLI, local Qwen endpoint, fixture runner | evaluator and signed desktop package available | runtime task + verifier + telemetry row |
| Qwen Code | isolated install, same endpoint/model, fresh config | isolated install exists at `~/.local/share/qwen-code-ab` | native Qwen tool loop + verifier |
| OpenCode | pinned release/source checkout, fresh config | setup not yet verified in this checkout | `--version`, model probe, task + verifier |
| Pi | pinned release/source checkout, fresh config | setup not yet verified in this checkout | `--version`, model probe, task + verifier |
| Aider | pinned release, fresh config, same OpenAI-compatible endpoint | setup not yet verified in this checkout | `--version`, model probe, task + verifier |

The first implementation step is to make each row’s bootstrap and one-task smoke run produce the same result schema. No cross-harness ranking is published until all five rows have valid runtime evidence.

## First valid runtime row

**OpenCode 1.18.31** was run alone with `ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit` through the local OpenAI-compatible MLX endpoint. After adding a project-local provider configuration, it completed the seeded `add.js` repair task, executed the fixture verifier successfully, and exited 0 in **20.56 s**. The raw transcript is `/tmp/zero-opencode-result.txt`. This is a valid runtime row, but it is not yet a ranking: equivalent rows for Zero Code, Qwen Code, Pi, and Aider are required.

**Qwen Code isolated install** completed the same seeded repair task against the same model endpoint after explicit `--auth-type openai` and `--approval-mode yolo` bootstrap. It edited `add.js`, repaired the malformed test script, ran `npm test`, and exited 0 in **64.89 s**. Raw output: `/tmp/zero-qwen-result.txt`; timing: `/tmp/zero-qwen-time.txt`. Two earlier invocations were rejected before model execution (conflicting yolo flags, then missing auth type) and are recorded as setup corrections, not benchmark rows.

**Aider 0.86.2** was run in a fresh Git fixture (Python 3.12 isolated environment) against the same model endpoint. It edited `add.js`, ran the test command successfully, and exited 0 in **12.24 s**. Raw output: `/tmp/zero-aider-result2.txt`; timing: `/tmp/zero-aider-time2.txt`. An earlier non-Git/malformed-fixture attempt is excluded as invalid because its verifier failed.

**Zero Code 0.0.0-main-202609162231** completed the same seeded repair task through its built Darwin CLI with a project-local OpenAI-compatible provider configuration. It read and edited `add.js`, ran `npm test`, and exited 0 in **12.86 s**. Raw output: `/tmp/zero-zero-result4.txt`; timing: `/tmp/zero-zero-time4.txt`. An interrupted earlier run is excluded; it had already changed its temporary fixture before interruption.

**Pi 0.73.1** was configured through an isolated `models.json` provider definition (`openai-completions`, local base URL, Qwen thinking compatibility) and run alone on the same fixture. It edited `add.js`, ran `npm test`, and exited 0 in **9.62 s**. Raw output: `/tmp/zero-pi-result2.txt`; timing: `/tmp/zero-pi-time2.txt`. The initial environment-variable-only attempt is excluded because Pi correctly routed it to OpenAI’s default endpoint.

## Multi-file refactor task

Pi 0.73.1 completed `js-refactor-dedupe`: extracted `slugify` into `lib/slug.js`, updated both consumers, added no other files, and passed `node --test` (2/2) in **15.54 s**. Raw output: `/tmp/zero-pi-refactor-result.txt`; timing: `/tmp/zero-pi-refactor-time.txt`.

Zero Code completed the same `js-refactor-dedupe` task, passed `node --test` (2/2), and respected the one-file addition constraint in **18.67 s**. Raw output: `/tmp/zero-zero-refactor-result.txt`; timing: `/tmp/zero-zero-refactor-time.txt`.

Aider completed the same refactor and passed `node --test` (2/2) in **11.00 s**. It also created ignored `.aider*` session metadata in the fixture, so it passes functional verification but violates the literal “only add `lib/slug.js`” filesystem constraint unless harness metadata is explicitly excluded. Raw output: `/tmp/zero-aider-refactor-result.txt`; timing: `/tmp/zero-aider-refactor-time.txt`.

OpenCode 1.18.31 completed the same refactor, passed `node --test` (2/2), and added only the requested source file (apart from the provider config) in **24.83 s**. Raw output: `/tmp/zero-opencode-refactor-result.txt`; timing: `/tmp/zero-opencode-refactor-time.txt`.

Qwen Code completed the same `js-refactor-dedupe` task, passed `node --test` (2/2), and respected the source-file constraint in **46.34 s**. Raw output: `/tmp/zero-qwen-refactor-result.txt`; timing: `/tmp/zero-qwen-refactor-time.txt`. It emitted a warning that `write_file` was unavailable but completed using its available tools.

## Zero title-overhead experiment

Zero Code repair rerun with an explicit `--title benchmark` still passed but took **16.28 s**, slower than the 12.86 s baseline. The hypothesis that automatic title generation dominated latency is therefore unconfirmed; no title-related code change is justified from this sample.

## Zero Code speed telemetry

The existing evaluator completed a single-harness speed run against the same loaded Qwen engine. Direct rows: 22,646 prompt tokens took 67.8s TTFT (335 prefill tok/s, 57.3 decode tok/s); 90,505 prompt tokens took 293.7s TTFT (308 prefill tok/s, 40.5 decode tok/s). Harness rows exposed prefix-cache behavior: cold harness 78.5s TTFT with 4,096 cached tokens, warm same project 1.3s with 31,704 cached, and cold other project 7.0s with 28,672 cached. Full report and JSON: `docs/evals/2026-09-16-zero-speed-controlled.md` and `.json`. This establishes cache reuse as a primary Zero Code optimization axis.

## OpenBench adoption and Zero adapter smoke

The authoritative harness comparison runner is now the upstream OpenBench checkout
at commit `db193457a3d9128cd4d01fd839c2a890c186c9ac` (MIT). OpenBench's checker
contract matches this protocol: it runs each harness headlessly on a disposable
workspace and decides success from an external checker, with transcripts and
efficiency fields retained separately. The existing LoCoMo/LongMemEval/MemoryAgentBench
launcher remains useful for memory-system experiments, but is not the harness
leaderboard because its adapters are incomplete and its datasets measure different
capabilities.

OpenBench's zero-cost control validated all eight core task checkers (untouched
workspace fails; golden solution passes). Its `null` adapter then failed the
`fix-failing-test` task with checker exit 1, confirming that a no-op cannot be
mistaken for success.

A temporary declarative OpenBench candidate ran the real packaged Zero Code binary
against the local Qwen3.8 MLX endpoint on that same task. The first attempt exposed
a runtime integration bug: Zero uses inherited `PWD` when resolving its project, so
the process attached to the OpenBench checkout instead of the disposable task
workspace. Passing `PWD={workspace}` at the adapter boundary fixed the issue. The
validated run then edited `calculator.py`, ran all four unit tests successfully, and
passed the external checker: **1/1, 35.55 s, Zero `0.0.0-main-202609162231`**.
Raw result: `/tmp/openbench-zero-qwen-20260916.jsonl`; transcript:
`/tmp/openbench-zero-transcripts-20260916/openbench-zero-qwen-20260916/zero-qwen_5d5052424f42_fix-failing-test_qwen3.8-local_trial1.txt`.

### Revised execution plan

1. Use OpenBench's pinned task/checker/result schema for coding-harness comparisons.
2. Keep the maintained Zero candidate adapter in
   `scripts/openbench-zero/run.sh`: it stages tasks from the pinned commit,
   sets the task workspace explicitly, avoids arbitrary environment inheritance,
   and records packaged-runtime provenance in OpenBench's result row.
3. Run the same OpenBench task cells for Zero, Qwen Code, OpenCode, Pi, and Aider,
   one model/harness at a time. The repair and multi-file feature tiers are now
   validated; continue with long-context, browser, and recovery tiers.
4. Keep LoCoMo/LongMemEval/MemoryAgentBench as a separate memory evaluation track;
   do not combine their scores with coding-harness correctness or speed.
5. Treat only external-checker passes with complete runtime evidence as valid
   comparisons; compilation, startup, plans, and agent self-reports remain
   diagnostic data.

## OpenBench local Qwen cell results

The first temporary matrix is **invalid for comparative timing**. OpenBench's
legacy local runner leaves its task `workspace/` directory mutable; after the
first successful Zero run, later harnesses could observe an already-repaired
seed. Their rows are retained as setup evidence but are not used for ranking.

The repository-owned runner and fresh Git-staged task packs were then used for a
clean one-cell-per-harness rerun. Each row below started from the pinned failing
seed, used the same local
`ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit` endpoint, and passed the
external checker:

| candidate | version | wall time | checker | filesystem note |
|---|---:|---:|---:|---|
| Aider | 0.86.2 | 4.85 s | 0 | writes `.aider*` session metadata |
| Pi | 0.73.1 | 10.47 s | 0 | task files only |
| Qwen Code | 0.24.0 | 15.74 s | 0 | task files only |
| Zero Code | 0.0.0-main-202609162231 | 14.94 s | 0 | task files only |
| OpenCode | 1.18.31 | 16.33 s | 0 | task files only |

These remain a shakedown matrix, not a performance leaderboard: there is one
trial per harness, model KV-cache state was warmed across cells, and token usage
was not exposed by the declarative manifests. Every candidate used an isolated
home, explicit local endpoint configuration, and a fresh Git-staged task pack;
Aider also received the source files through `workspace_file_globs`. The clean
rerun proves that all five harnesses can solve the same failing seed when
workspace isolation and file admission are configured correctly.

Clean result records are `/tmp/openbench-aider-qwen-isolated.jsonl`,
`/tmp/openbench-pi-qwen-isolated.jsonl`,
`/tmp/openbench-qwen-code-isolated.jsonl`,
`/tmp/openbench-opencode-qwen-isolated.jsonl`, and
`/tmp/zero-openbench-repo-explicit-config.jsonl` (rerun confirmation:
`/tmp/zero-openbench-repo-final.jsonl`).
The normalized comparison is retained in
`docs/evals/2026-09-16-openbench-clean.json`.

The Aider warning in the attached screenshot is now addressed in the benchmark
manifest with `--model-metadata-file`, registering the Qwen context window and
zero local costs. This removes the warning and makes Aider's accounting explicit;
Aider documents this metadata mechanism for unknown models in its advanced model
settings guide.

## Zero Code desktop-app lane

The same class of repair was then run through the actual signed Zero Code desktop
application (`Zero Code Dev`) with the Qwen3.8 Flash Next model selected in the
composer. The app created a session, streamed tool activity, inspected the
disposable `desktop-task` fixture, edited only `desktop-task/add.js`, ran
`node desktop-task/test.js`, and reported the passing result in **26 s**. An
independent shell invocation of the verifier also returned `PASS`.

The desktop transcript exposed one measurement caveat: Zero Code records its
`telemetry.db-wal` file as a changed file alongside the requested source edit.
That is harness telemetry side effect, not a task failure, and should be excluded
from source-file constraint scoring or reported as a separate cleanliness metric.
Screenshot evidence is saved at `/tmp/zero-code-desktop-openbench-result.png`.

The rerunnable bootstrap and staging contract are documented in
`docs/evals/openbench-zero.md`.

## OpenBench multi-file feature tier

The `add-feature` task adds recursive `@include` support to a Python package
with hidden feature tests and regression groups. Fresh pinned task packs were
used for each cell. Zero Code, Pi, OpenCode, and Aider all completed with score
1.0 and checker exit 0:

| candidate | version | wall time | checker | completion |
|---|---:|---:|---:|---|
| Aider | 0.86.2 | 20.57 s | 0 | complete |
| Pi | 0.73.1 | 71.53 s | 0 | complete |
| Zero Code | 0.0.0-main-202609162231 | 73.16 s | 0 | complete |
| OpenCode | 1.18.31 | 114.82 s | 0 | complete |
| Qwen Code | 0.24.0 | 300.95 s | 0 | **timed out** |

Qwen Code produced a correct workspace, but its `--max-wall-time` budget ended
with exit 55 before a clean process completion. The same behavior occurred at
both 180 s and 300 s, so this is recorded as a harness timeout rather than a
valid pass. The task result is therefore a capability signal (the hidden tests
passed) and a reliability failure (the agent did not terminate).

The normalized tier data is retained in
`docs/evals/2026-09-16-openbench-add-feature.json`.

## Zero Code desktop feature loop

The same `@include` feature was driven through the actual Zero Code desktop app
against a disposable copy of the task. The first pass took **2m 27s**, changed
`miniconf/loader.py`, ran the visible 13-test suite, and reported success. The
independent hidden checker scored it **0.9000** because missing include files
escaped as `FileNotFoundError` rather than `ConfigError`.

That verifier failure was sent back into the same desktop session. Zero Code
then made the targeted one-line exception fix, reran the visible tests, and the
hidden checker reached **6/6 feature tests and 4/4 regression groups** with
`SCORE: 1.0000`. The correction pass took **27 s**. The app also created Python
`__pycache__` files and a `telemetry.db-wal`; these are harness side effects and
should be excluded from source-edit scoring.

Desktop evidence is saved at `/tmp/zero-code-desktop-add-feature-result.png`,
with normalized results in
`docs/evals/2026-09-16-zero-desktop-add-feature.json`.

## OpenBench complex orchestration tier

The `taskflow` task exercises retries, priority scheduling, resource limits,
state transitions, event delivery, and event-sourced history. Fresh pinned task
packs were used for every candidate. Zero Code, Pi, OpenCode, and Qwen Code all
completed the full checker with score 1.0:

| candidate | version | wall time | score | completion |
|---|---:|---:|---:|---|
| Qwen Code | 0.24.0 | 171.70 s | 1.00 | complete |
| Zero Code | 0.0.0-main-202609162231 | 169.47 s | 1.00 | complete |
| Pi | 0.73.1 | 197.39 s | 1.00 | complete |
| OpenCode | 1.18.31 | 225.23 s | 1.00 | complete |
| Aider | 0.86.2 | 180.13 s | 0.28 | complete, incorrect |

Aider's run passed only 7/25 hidden tests. Its transcript shows the model
attempting `cd /app`, which is unrelated to the disposable workspace, after all
task files were injected into its context. This is recorded as an output-quality
and adapter-behavior failure, not a timeout. Qwen Code completed this complex
task cleanly at the 600-second budget, so its earlier add-feature timeout is
task-specific rather than a universal termination failure.

Normalized results are retained in
`docs/evals/2026-09-16-openbench-taskflow.json`.

## OpenBench webcore routing tier

The `webcore` task is the largest current tier: 44 files and 43 external
checks covering typed converters, route precedence, mounted applications,
middleware ordering, redirects, 405/OPTIONS/HEAD handling, and reverse URL
generation. Zero Code produced a fully correct workspace: the checker reported
`regression 19/19`, `feature 24/24`, and `SCORE: 1.0000`.

The first run hit the 900-second wall-time limit (`completed=false`) after
solving the workspace. I added an adapter-level completion guard and validated
it on the repair smoke task (checker 0, clean completion in 15.2 s). A second
fresh webcore run then timed out at the same cap with 16/24 feature checks
passing. The paired evidence shows two separate issues: Zero can solve the
task, but completion is unreliable on this large stochastic prompt and a
single completion suffix is not a substitute for task decomposition or a
progress-aware stop policy. No third retry is justified. Raw results and
transcripts are retained in `docs/evals/2026-09-16-openbench-webcore.json`.

## Next execution plan

1. Keep the repository-owned runner as the single OpenBench admission path and
   require `success=true`, `completed=true`, and `checker_exit=0` for a strict
   pass.
2. Add progress and termination telemetry to Zero's runtime output so a long
   task reports current turn, last tool activity, tokens, and elapsed time, and
   can be stopped when it is making no measurable progress.
3. Split the webcore work into typed-routing, mounts/middleware, and HTTP
   semantics subtasks, then rerun each as a fresh task pack. This tests whether
   the current failure is prompt size, implementation coupling, or termination
   policy.
4. Exercise browser, recovery-after-interruption, context-rotation, and
   long-context tiers through the desktop app as well as the CLI adapter. Keep
   one model and one harness active per run.
5. Promote only improvements that pass an external runtime checker and record
   speed, token usage, completion reliability, filesystem side effects, and
   output quality separately.

The current provisional aggregate is retained in
`docs/evals/2026-09-16-openbench-leaderboard.json`. Pi, OpenCode, and Zero each
pass all four currently comparable task cells; Pi is fastest of those three
by median wall time, with Zero close behind. Qwen Code has correct workspaces in all three but a clean
completion failure on `add-feature`. Aider is fastest on the smallest task but
fails the complex task's hidden checks. These standings remain provisional until
the same cells have repeated trials and token telemetry.

The repeat-aware task-level snapshot is now also retained in
`docs/evals/2026-09-16-openbench-repeat-aware.json` and is generated by
`scripts/openbench-zero/aggregate.py --task-level`. It collapses repeated
trials before calculating latency and requires every trial in a candidate/task
cell to pass for a strict task pass. On the current four-task set, Pi, Zero
Code, and OpenCode are tied at 4/4 strict task cells; their median task-cell
wall times are 48.1 s, 71.5 s, and 74.7 s respectively. Aider is 3/4 at
16.1 s and Qwen Code is 3/4 at 102.4 s. The speed advantage of Aider is
therefore recorded without allowing it to outrank a harness that completed
more tasks.

Zero's first repeated cell is now recorded separately in
`docs/evals/2026-09-16-zero-clean-trials3.json`: three fresh trials of the
repair task all passed and completed, ranging from 26.6 s to 31.8 s. This
supports a stable small-task completion result while leaving larger-task
variance unresolved.

The two-trial `add-feature` repeat is recorded in
`docs/evals/2026-09-16-zero-add-feature-trials2.json`: both cells passed and
completed, at 105.4 s and 125.1 s. The earlier 73.2 s single cell remains
valid evidence, but the repeat shows that multi-file latency varies materially;
future speed comparisons need repeated trials rather than a single timing.

Pi's three-trial repeat is recorded in
`docs/evals/2026-09-16-pi-clean-trials3.json`: all cells passed, with a median
of 24.7 s. Its earlier 10.5 s single cell was faster, confirming that process
startup and model state materially affect this small task; single-cell timing
should not be used as a stable speed ranking.

The repository-owned `scripts/openbench-zero/aggregate.py` now computes these
metrics directly from raw JSONL rows. Task-level cells now retain the sorted
trial timings and their minimum/maximum spread. A smoke aggregation of the
repeated Zero and Pi cells produced matching 3/3 strict pass rates and medians
of 27.824 s and 24.680 s, respectively; Zero's observed spread was 26.556–
31.761 s and Pi's was 20.931–31.238 s.

OpenCode's three-trial repeat is recorded in
`docs/evals/2026-09-16-opencode-clean-trials3.json`: all cells passed and
completed, with a 34.5 s median. Its earlier 16.3 s cell was a single fast
sample, so the repeated result is the more useful speed reference.

Aider's three-trial repeat is recorded in
`docs/evals/2026-09-16-aider-clean-trials3.json`: all cells passed and
completed, with an 8.4 s median after supplying explicit Qwen model metadata.
This confirms Aider's small-task speed advantage while leaving its taskflow
correctness failure unchanged.

Qwen Code's three-trial repeat is recorded in
`docs/evals/2026-09-16-qwen-code-clean-trials3.json`: all cells passed and
completed, with a 33.1 s median. Its add-feature timeout therefore remains a
multi-file termination issue rather than a general inability to finish the
repair task.

With all five candidates repeated on the same clean task, the measured medians
are Aider 8.4 s, Pi 24.7 s, Zero Code 27.8 s, Qwen Code 33.1 s, and OpenCode
34.5 s; every candidate is 3/3 strict-pass. This is a useful speed slice, not a
general harness ranking: the task is tiny, the KV cache was warm, and token
telemetry is unavailable.

## Zero browser-control tier

The existing `zero-eval` browser fixture was run through Zero's real server and
Playwright MCP against a locally served page. Qwen navigated to the page,
captured two snapshots, clicked `Reveal`, read the revealed value, and wrote
only that value to `result.txt`. The hidden verifier passed in **42.1 s** with
five tool calls, zero failed calls, and a 98% prefix-cache hit rate.

This is a runtime capability result for Zero's browser lane. It remains separate
from the OpenBench coding leaderboard because the other harnesses do not yet
have an equivalent browser fixture in this setup. Evidence is in
`docs/evals/2026-09-16-browser-reveal.json` and
`docs/evals/2026-09-16-browser-reveal.md`.

## Zero speed and context tier

The direct engine and harness speed run is recorded in
`docs/evals/2026-09-17-speed-repeat.json`. On this M4 Max, a 56-token prompt
had 2.0 s TTFT. The 22.7k-token and 90.5k-token context rows had 97.3 s and
234.5 s TTFT, respectively. The harness cache rows measured 9.2 s cold, 1.4 s
warm in the same project, and 7.1 s cold in another project; the warm row had a
99.9% prefix-cache hit. These measurements make context admission and cache
reuse first-order performance work, rather than UI polish.

The shared 16k-word retention probe passed three times for Pi, Aider, Qwen Code,
and OpenCode. The first 64k single-trial lane passed for all five candidates,
including Zero Code's packaged desktop: Pi 184 s, OpenCode 177 s, Aider about
131 s, Qwen Code about 188 s, and Zero Code 236 s. Zero's telemetry recorded
96,761 input tokens, 234.7 s TTFT, 28,672 cached tokens, and the UI displayed
`37% used`, confirming that the context meter reports used capacity. These are
runtime rows, but they remain out of the all-around score until three-trial,
cold-cache, near-limit, and context-rotation fixtures are complete. Raw rows
are in `docs/evals/2026-09-17-context-probe.json` and
`docs/evals/2026-09-17-context-64k-probe.json`.

## Zero long-running goal tier

The existing `long-kv-cli` goal fixture completed through Zero's real server in
**301 s**. The model created the ES-module CLI and a 24-test `node:test` suite;
the verifier passed all tests. The bounded goal used eight tool calls across
eight requests, 9,724 output tokens, 25.0 decode tok/s, and a 90% prefix-cache
hit rate. The fixture's advisory expectation was 12 tool calls, so activity was
lower than expected even though correctness and clean completion passed. This
is evidence for measuring useful work separately from raw tool-call volume.

Evidence is retained in `docs/evals/2026-09-17-long-kv-cli.json` and
`docs/evals/2026-09-17-long-kv-cli.md`.

The cross-tier scorecard is generated by `scripts/evals/scorecard.py` and is
stored in `docs/evals/2026-09-17-scorecard.json` with a readable view in
`docs/evals/2026-09-17-scorecard.md`. It keeps the OpenBench coding leaderboard
comparable across all five harnesses and records Zero-only dimensions with
their measured runtime values plus the candidates still missing equivalent
fixtures. It intentionally leaves the composite score unset until those
fixtures exist.

The desktop status smoke also ran through the actual Zero Code Dev window on
the disposable `long-kv-cli` project. During the live request the accessibility
tree exposed `Thinking`, an elapsed timer, the selected Qwen model, and a Stop
control. The request then returned `OK` in 9 s and the composer showed `1% used`.
This confirms the current status/context visibility path at runtime; evidence
is in `docs/evals/2026-09-17-zero-desktop-status-smoke.json`.

A separate desktop browser smoke completed the requested report for
`https://example.com`, but it took 107 s, had two failed browser calls during
tab readiness/recovery, and returned a zero-byte screenshot. The model recovered
by navigating the created tab before inspecting it, so this is classified as
`completed_with_tool_failures`, not a clean browser pass. Evidence and the
follow-up target are in `docs/evals/2026-09-17-zero-desktop-browser-smoke.json`.

The follow-up runtime probe isolated and fixed the screenshot transport failure.
Agent-created tabs are initially hidden, and Electron's native `capturePage()`
and CDP surface capture both fail without a presented surface. The production
bridge now temporarily enables an unpresented view for the CDP capture, restores
its prior visibility, and resizes the PNG to the existing 1280-pixel bound. The
hidden-tab probe then passed navigation, accessibility inspection, and a
24,187-byte PNG; the visible-tab lifecycle probe remained green. This is a
source-level Electron runtime pass. The installed desktop bundle still needs a
fresh app restart and live model task before the earlier desktop evidence can be
reclassified. Evidence is in
`docs/evals/2026-09-17-zero-desktop-browser-fix.json`.

The freshly rebuilt desktop bundle was also launched in an isolated onboarding
profile. Its Browser panel opened, loaded `https://example.com/`, and exposed
the `Example Domain` heading through the native view's accessibility tree. The
selected default provider rejected the agent prompt before tool execution, so
this verifies the rebuilt UI/browser path separately from the hidden-tab bridge
probe. Evidence and screenshot are in
`docs/evals/2026-09-17-zero-desktop-browser-bundle-smoke.json` and
`docs/evals/2026-09-17-zero-desktop-browser-bundle-smoke.jpeg`.

The rebuilt desktop bundle then completed the same browser task with the
configured Qwen3.8 model: it created a tab, navigated to `example.com`,
inspected the page, captured a **33,666-byte PNG**, and reported the tab ID and
`Example Domain` title in **33 s**. All four browser calls completed without
failure. This is the clean live desktop result for the repaired browser lane;
evidence is in `docs/evals/2026-09-17-zero-desktop-browser-rebuilt-live.json`.

## Recovery tier

The repository now includes `scripts/evals/recovery-run.ts`, which creates an
isolated project, starts Zero's real server, interrupts an active Qwen turn,
resubmits on the same session, and runs the hidden verifier. The runtime
result passed: the first turn was interrupted after two projected messages,
the resumed session finished with two user prompts and seven projected
messages, and the pagination verifier passed all seven tests. The first phase
took 8.1 s and the resumed phase 12.0 s. This demonstrates session recovery
for a real edit, while remaining a Zero-only dimension until equivalent
interruption fixtures exist for the other harnesses. Evidence is in
`docs/evals/2026-09-17-zero-recovery.json`.

## Stable-prefix speed experiment

The same two-turn `Reply exactly OK.` session was run with
`experimental.stable_prefix` enabled and disabled. Both variants processed
3,142 input tokens, restored 60,388 cached tokens, and achieved a 95.1% prefix
hit rate. The enabled run averaged 3.18 s TTFT across two requests; the
disabled run averaged 2.66 s. This paired run shows cache reuse remained
available in both modes and does not justify claiming a speed gain from the
flag; it is retained as a baseline for future multi-turn tool workloads.
Evidence is in `docs/evals/2026-09-17-zero-stable-prefix-ab.json`.

## Desktop Minecraft MCP tier

The rebuilt Zero Code desktop drove the real NeoForge 1.21.1 Field Emitters
client through the registered MCP endpoint. After fixing endpoint, nullable
screen, generator, and argument-schema defects, Qwen joined the active world,
applied one `fieldemitters:field_emitter` block, inspected its authoritative
state and NBT, queried loaded emitters, and captured an **88,480-byte
1708x960 PNG** in **57 s**. The authoritative inspection matched the
requested block and `qa.emitters` reported the new emitter. Evidence is in
`docs/evals/2026-09-17-zero-desktop-minecraft.json`.

The fixes were runtime-driven: `mc.screen.current` now permits its normal
nullable screen result, `start-field-emitters.ts` removes stale discovery
metadata and probes the newly published endpoint before registration, and the
generator maps `flat: true` to the flat preset while honoring `creative`.
Minecraft build and QA tools now publish explicit JSON schemas instead of a
generic object, and Zero's native runtime normalizes JSON-encoded array/object
arguments before execution. An earlier run reproduced the stale endpoint,
schema guessing, and uniform `fetch failed` errors; they remain recorded as
prior failure evidence rather than being hidden.

## OpenBench make-it-run matrix

The second pinned repair task, `make-it-run`, passed for every harness with a
fresh task pack and isolated home. Wall times were Aider **11.6 s**, Pi
**13.6 s**, Qwen Code **16.5 s**, Zero Code **17.5 s**, and OpenCode **21.5 s**;
all checker exits were zero and all processes completed. This is a second
runtime slice, not a universal ranking. Normalized evidence is in
`docs/evals/2026-09-16-openbench-make-it-run.json`.

## Pi-style tool-profile experiment (2026-09-17)

Pi's coding runtime exposes a small active set by default (`read`, `bash`,
`edit`, and `write`). To test whether that explains its speed advantage, Zero
was run on the same pinned OpenBench `fix-failing-test` task twice with its
normal tool set and twice with a permission configuration that withheld tools
not needed for basic coding. All four runs passed the external checker with a
score of 1.0.

The restricted profile reduced the serialized tool schema from 21,188 to
12,673 bytes (40.2%). Its median wall time was 18.482 s versus 19.089 s for
the full profile. That 0.607 s difference is too small to call a general
speedup with two trials, but the payload reduction is real and has no observed
correctness cost. The full desktop capability set remains the default while
long-context and multi-tool profile tests are pending. Raw rows and the exact
configuration are recorded in `docs/evals/2026-09-17-pi-tool-profile.json`.

The product implementation now exposes the same idea as
`ZERO_TOOL_PROFILE=coding-minimal`, with exactly `read`, `bash`, `edit`, and
`write` available. On the larger `add-feature` task, two fresh trials per
profile all passed the hidden checker: coding-minimal had a 72.469 s median,
while full had a 191.077 s median. This is strong task-specific evidence that
the lean profile can bring Zero close to Pi's 71.531 s single-cell result, but
the shared MLX prefix cache and different model trajectories mean it remains a
directional result pending cold/warm long-context and multi-tool repeats.
