# Decision Log

Every entry records the decision, the alternatives, the evidence, the reasoning, the bias
it may introduce, and whether it was made before or after any benchmark result existed.

---

## D1. The fixed model is the ARC4NUM 4-bit build, not a 27B dense build

**Made:** 2026-09-18, before any benchmark result.

**Decision.** The study fixes on `ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit`,
served by mlx-serve 26.9.2 at `http://127.0.0.1:11234/v1`.

**Alternatives considered.** (a) Hunt for a "Qwen 3.8 27B Abliterated" build as the brief
named it. (b) Use the second build present on the machine,
`ddark-il/Qwen3.8-Flash-Next-Uncensored-Mixed-omlx`. (c) Download a different Qwen.

**Evidence.** Inspection of `~/.mlx-serve/models` found exactly two Qwen3.8 builds and no
27B dense build of any kind. The two differ materially: ARC4NUM is uniform 4-bit affine
g64 across 102 shards; ddark-il is mixed precision with 8-bit on selected linear-attention
and vision tensors across 22 shards. Their weight manifest hashes differ. The ARC4NUM
build is the one the engine had loaded and the one `~/.config/zero/zero.json` points at.

**Reasoning.** The brief forbids silently substituting another build, so the discrepancy
between the named model and what exists was raised rather than papered over, and the owner
designated the ARC4NUM build. Quantization is part of model identity, so the ddark-il build
is a different model for this study's purposes and is out.

**Possible bias.** Conclusions are about a 4-bit MoE with MTP and a 262k context. They do
not transfer to a dense 27B, and nothing here should be read as if they do.

---

## D2. The study workspace lives outside the subject repository

**Made:** 2026-09-18, before any benchmark result.

**Decision.** All study artifacts live in `/Users/zero/Documents/Dev/qwen-harness-study`,
its own git repository. Nothing is written into `zero-code-next`.

**Alternatives considered.** A `docs/research/` subtree inside the Zero Code repo.

**Reasoning.** Zero Code is a subject of the experiment, not its container. Raw results and
transcripts are large and would churn the subject repo's history, and a benchmark artifact
landing in the subject tree is exactly the contamination section 3 of the brief warns about.

**Possible bias.** None identified. It does mean the two repositories must be cross
referenced by commit SHA, which the baseline profile does.

---

## D3. The frozen baseline is HEAD plus the measurement apparatus, and nothing else

**Made:** 2026-09-18, before any benchmark result.

**Decision.** `zero-code-baseline` = commit `2b47e4f94b6052c7c76df94c9e27ef454ad97d1e`,
which is `zero-code-next` HEAD `19557a1ae9` plus exactly three things:

1. the engine timings passthrough, so prefill and decode are measured by the engine rather
   than inferred from the harness clock;
2. the `LLMRepair.deferredSchemas` call site, which HEAD left dead: the function was
   committed without its caller, so tool-call repair was not actually running at HEAD;
3. the `stripReasoning` option on `MessageV2.toModelMessages`, without which HEAD does not
   typecheck at all. Its behaviour is gated off by default.

**Alternatives considered.** (a) Freeze at HEAD exactly. (b) Freeze at the current working
tree. (c) Freeze at HEAD plus instrumentation only.

**Evidence.** HEAD does not compile: `packages/zero/src/session/prompt.ts:1404` and `:1411`
pass `stripReasoning` to a function whose type at HEAD has no such field. Separately,
`git grep deferredSchemas HEAD` finds the definition and no call site, so the repair path
committed earlier was inert. The working tree carries roughly 340 changed files of
unfinished work from a parallel session, including new system prompts and a rewritten
native runtime, which is neither reproducible nor attributable.

**Reasoning.** A baseline that does not build cannot be run, so (a) is out. A baseline made
of somebody else's unfinished work is not reproducible, so (b) is out. Instrumentation has
to be in both arms or the comparison is not like for like. The two non-instrumentation
inclusions are each the missing half of something already committed, not a new capability
chosen in response to evidence, and no benchmark result existed when this was decided.

**Explicitly excluded.** A five minute LLM stream timeout present in the working tree. With
a 98 second cold prefill already measured on this machine, that timeout would convert slow
requests into failures. It changes outcomes rather than observing them, so it has no place
in a baseline.

**Possible bias.** Including the repair call site makes the baseline stronger at tool-call
recovery than HEAD would have been, which flatters Zero Code's baseline relative to a
naive "last commit" reading. This is stated here so the final report can say so plainly.

**Superseded on 2026-09-18 by D11, before any Stage 1 result existed.** The commit below is
kept as `zero-code-baseline-2026-09-18` for traceability but is no longer the subject.

**Construction method.** The commit was built in a temporary git index with `commit-tree`,
so the working tree and the user's index were never touched. Verified afterwards:
`git diff zero-code-baseline` against the working tree shows only the deliberately excluded
hunks; typecheck is clean for `zero`, `schema`, `telemetry`, `guard` and `engine`; and
`bun test test/session test/engine test/tool` gives 879 pass, 7 skip, 1 todo, 0 fail.

---

## D4. Weight identity is a shard manifest hash, not a content hash

**Made:** 2026-09-18, before any benchmark result.

**Decision.** `MODEL_FINGERPRINT.json` records `weights_sha256` as UNKNOWN and instead
carries `weights_manifest_sha256`: sha256 over newline-joined `filename:bytes` for all
safetensors shards and the ngram table, filenames sorted.

**Alternatives considered.** A full content hash of 107.3 GB.

**Reasoning.** A content hash costs a multi-minute full read of the array every time
identity is verified, and the point of the field is to detect a swapped or requantized
build between runs. Shard count and per-shard byte sizes already separate the two builds
present on this machine, and the tokenizer, chat template, config and safetensors index
are hashed by content because they are small.

**Possible bias.** A different checkpoint with byte-identical shard sizes would not be
detected. That is unlikely and it is recorded here as a known limit.

---

## D5. Sampling parameters are the engine's defaults, and per-harness deviations are recorded rather than forced

**Made:** 2026-09-18, before any benchmark result.

**Decision.** The engine's own defaults apply (temperature 1.0, top_p 0.95, top_k 20, from
`generation_config.json` and confirmed by the server's `/v1/models` metadata). Zero Code
sends no sampling override for this model id. Other harnesses will send their own
defaults; the metering proxy records exactly what each one sends, and any difference is
reported as a recorded confound rather than silently normalised away.

**Alternatives considered.** Forcing every harness to identical sampling by rewriting
requests in the proxy.

**Reasoning.** Sampling defaults are part of the harness as a product, which is what Track A
measures. Rewriting them would measure a harness nobody ships. Track B, the controlled
ablation track, holds sampling fixed because there the harness is not the variable.

**Possible bias.** Track A comparisons are not sampling-controlled. This is the intended
reading of Track A, and Track B exists precisely to answer the causal questions Track A
cannot.

**Consequence.** Temperature is 1.0, so repeated runs of one task are stochastic replicates.
No seed control was found in the engine's launch arguments. Replication counts must be
read with that in mind, and section 20 of the brief applies: repeats are not new tasks.

---

## D6. OpenBench is reused for execution and scoring, with five named patches, and is not trusted for measurement

**Made:** 2026-09-18, before any benchmark result, on the source audit in `OPENBENCH_AUDIT.md`
(commit `db193457a3d9128cd4d01fd839c2a890c186c9ac`, 2026-08-20).

**Decision.** Adopt OpenBench as the task runner. Reuse its workspace materialisation and
per-trial reset, its checker isolation, its failure taxonomy, its resume-by-run-id and its
admission-time positive and negative controls. Do not use its token counts, its wall-clock
comparisons, its model identity strings or its trial-independence statistics.

**Alternatives considered.** Writing a runner from scratch.

**Evidence.** The audit found the isolation properties genuinely implemented:
`workspace.py:materialize_workspace` stages only `workspace/` into a fresh temp directory,
and `checker.sh` and `solution/` are siblings that are never copied in or handed to the
adapter. It also found real bias risks in the source: no execution-order randomisation
anywhere in the run loop, no Docker `--network` restriction, the `codex` and `cursor`
adapters reusing the real persistent home directory across trials while others isolate,
retry tuning present only for `pi`, and a sampled checker with no fixture-presence check so
an agent could delete the failing test and pass.

**Reasoning.** Rewriting a runner would cost days and would reproduce the same isolation
logic that already exists and already works. The problems found are localised and each has
a named fix. Its measurement, by its own admission in `run.py` comments, is confounded, and
we already have a better instrument: the metering proxy counts tokens and time the same way
for every arm regardless of what a harness reports about itself.

**Patches to apply, each traceable to an audit finding.**
1. Randomise or block harness order within tasks.
2. Restrict container egress so network access cannot contaminate a task.
3. Give every adapter an isolated home directory, matching the ones that already do.
4. Make retry policy identical across adapters rather than tuned for one.
5. Add fixture-presence assertions to checkers so deleting a test cannot pass.

**Possible bias.** Patching a benchmark is a place where a preferred outcome can creep in.
Every patch above is listed before any result exists, is justified by a specific audit
finding, and applies to every arm symmetrically. Any later patch must be recorded here with
the same discipline and marked as made after results were seen.

---

## D7. Tokens and time are measured by our own proxy, not by the harnesses

**Made:** 2026-09-18, before any benchmark result.

**Decision.** Every harness reaches the engine through `tools/meter.ts`, one proxy process
per benchmark cell, labelled from the command line rather than from request headers.

**Alternatives considered.** Trusting each harness's self-reported usage, or parsing each
harness's own telemetry format.

**Reasoning.** Section 7 of the brief requires proving that each harness actually reaches
the intended model, and the OpenBench audit independently flagged self-reported token counts
and unverified model identity strings as unusable for cross-harness comparison. A proxy
answers both with one mechanism. Labels come from the command line because most harnesses
give no way to set a custom header, and a label the harness controls is a label the harness
can get wrong.

**What it records.** Requested model against the model that actually answered, the
instruction surface with its hash, tool names and a hash of the full schema set, tool choice,
sampling parameters, time to first byte and time to first content separately, usage, and the
engine's own timings object. Full request and response bodies are written beside each row.
Secrets are not recorded: the Authorization header is reduced to its presence.

**Proven.** A smoke request through the proxy recorded `model_matches: true`, the tool
schema hash, `ttfb_ms` 49 against `ttft_ms` 2937, and the engine's `prompt_ms` of 2792.6 for
274 uncached prompt tokens. A grep for the bearer token across the recorded output found
nothing.

**Possible bias.** The proxy adds a hop. Measured at 49 ms to first byte against a 2.9 s
request, its cost is within noise, but it is present for every arm equally and the raw
engine timings are recorded alongside so proxy overhead stays visible.

---

## D8. The engine stays warm across runs and the prefix cache is recorded, not defeated

**Made:** 2026-09-18, before any benchmark result.

**Decision.** One engine process serves the whole session. The prefix cache is left enabled.
Every request's `cached_n` and `prompt_ms` are recorded, and harness order is randomised or
blocked within tasks. The Zero Code adapter turns engine management off so no run can stop
the shared engine.

**Alternatives considered.** (a) Restart or evict the cache between cells so every run starts
cold. (b) Let each harness manage the engine itself.

**Evidence.** Measured on this machine: a 27,394 token prompt with no cache cost 98,160 ms of
prefill; the same prompt at 99.9 percent cached cost 390 ms. The model is roughly 75 GB
resident and reloading it per cell would dominate the run budget.

**Reasoning.** (b) is unworkable: a harness stopping the engine between cells would measure
model loading. Forcing cold prefill everywhere would multiply the wall clock by a large
factor and would not remove the bias, only relocate it. More importantly, how a harness
structures its prompt so that the cache survives from turn to turn is a genuine harness
design property, not an artefact, and Track A is meant to measure harnesses as they ship.

**Possible bias.** A harness that runs immediately after itself benefits from its own warm
prefix. Randomised order spreads that across arms rather than concentrating it, and
`cached_n` per request makes any residual visible in the analysis. If a result turns out to
hinge on cache state, that is reported rather than smoothed over.

---

## D9. The pilot task set is OpenBench's eight native tasks, validated locally

**Made:** 2026-09-18, before any benchmark result.

**Decision.** Development and pilot work uses the eight tasks shipped with OpenBench at the
audited commit. A private held-out set is authored separately and is never used for tuning.

**Evidence.** `obench validate --no-imported` on this machine: all eight pass both controls,
untouched workspace fails and the reference solution passes with score 1.0, across
`add-feature`, `build-a-cli`, `fix-failing-test`, `make-ci-green`, `make-it-run`,
`misleading-error`, `taskflow` and `webcore`. Four of the eight have a non-zero base score,
so they are partial-credit tasks rather than all-or-nothing.

**Reasoning.** These tasks already satisfy section 16's admission requirement and cover most
of the heterogeneity the brief asks for: a misleading traceback whose root cause is not where
it fails, a failing CI, a feature addition, a build from scratch. Authoring a fresh set
before the loop is proven would be exactly the elaborate infrastructure section 35 warns
against. Contamination matters less here than in model benchmarking because every arm runs
the same fixed model, but saturation would still destroy discriminative value, so the Stage 1
set is chosen from these eight by what actually separates arms.

**Possible bias.** These tasks were authored by the OpenBench project, which also authors its
own harness adapters. Nothing in them is specific to Zero Code, and the held-out set is ours.

---

## D10. Runs execute in local mode, not Docker, until the daemon is available

**Made:** 2026-09-18, before any benchmark result.

**Decision.** The pilot runs use `--exec local`. Stage 1 onwards should move to Docker.

**Evidence.** Docker is installed at `/usr/local/bin/docker` but its daemon is not running on
this machine, and every imported terminal-bench task fails validation for exactly that reason.

**Reasoning.** Local mode is enough to prove the loop and is identical for every arm, so it
does not bias a comparison between harnesses.

**Possible bias, and it is real.** Local mode has no filesystem sandbox and no network
restriction. The audit's finding stands: a harness with shell access could in principle reach
the checker on the host, and nothing stops an agent leaving its workspace or reaching the
network. The model under test is uncensored and has bash. Two of the five OpenBench patches,
container egress control and home directory isolation, cannot take effect in local mode.
This is acceptable for a loop proof and is not acceptable for results that get reported, so
bringing the daemon up is a prerequisite for Stage 1.

---

## D11. The baseline was re-frozen once, before any result existed

**Made:** 2026-09-18, before any Stage 1 result. Supersedes the freeze in D3.

**Decision.** `zero-code-baseline` now points at `f1acf655b98a107aedaeae098b002925051dfedd`,
tagged `zero-code-baseline-2026-09-18b`. The previous freeze, `2b47e4f9`, keeps its tag
`zero-code-baseline-2026-09-18` and stays reachable.

**What changed relative to the previous freeze.** Two commits, neither an experimental
treatment:

1. The CLI resolved its project root from `$PWD`, which only a shell keeps in step with the
   real working directory. It is a no-op in a shell and under any launcher that sets `PWD`,
   which every adapter in this study now does.
2. The telemetry ledger kept two of the six quantities the engine reports per request and
   dropped the rest. Recording them changes nothing about what the harness does.

**Why re-freeze rather than carry them as a candidate.** Both belong in every arm. Observation
has to be identical across arms or the comparison is not like for like, and the project root
fix is invisible to task outcomes. Carrying them only in a candidate would confound the final
baseline-versus-candidate comparison with an instrumentation difference.

**Why now and never again.** No benchmark result for any harness exists yet, so nothing is
being tuned in response to evidence. This is the one moment where re-freezing costs nothing.
After Stage 1 begins, section 19 of the brief applies without exception: the baseline is not
updated in response to observed weakness, and any further Zero Code change lives on a
`zero-code-experimental/*` branch and is judged against this commit on held-out tasks.

**Possible bias.** The project root bug was found *by* benchmarking, which is close to the
pattern section 31 warns against: inspect a weak result, patch, rerun, claim improvement. Two
things separate it. It was classified as an integration defect rather than a coding result,
so it never counted against Zero Code in the first place. And the benchmark's own adapter
already neutralises it by setting `PWD`, so the fix cannot move a single score either way.

**Verification of the new baseline, with a correction to how verification is done.** The D3
freeze was verified in a worktree whose `node_modules` was symlinked wholesale to the main
checkout, which made `@zero-code/*` resolve to the main tree's sources. A typecheck there was
reading code from outside the worktree, which is worse than no check because it looks like it
passed. `tools/link-worktree-deps.sh` now shares third-party packages but rebuilds workspace
links locally, and resolution is asserted before any check runs. Under that scheme the new
baseline is clean across `zero`, `schema`, `core`, `llm`, `plugin`, `telemetry`, `guard` and
`engine`, with 1,216 pass / 0 fail across the zero suites and 58 pass / 0 fail across the
schema and plugin suites.

## D12. The Zero Code adapter disables engine management only, not the engine plugin

**Made:** 2026-09-30, before any Stage 1 result.

**Decision.** The adapter's config sets `zero.engine.manage: false` and nothing else. It used to
set `enabled: false` as well, meaning "the benchmark owns the engine", but `enabled` is the engine
plugin's switch, not the lifecycle's. With it off, every Zero Code cell ran without the plugin:
no model metadata synced from the engine, no `engine_status` tool in the tool list, no
reachability check before a request, and on later commits no thinking-off for titles and
compaction. That is not the harness a user runs.

**Which results it touches.** The three Stage 0 cells in `RAW_RESULTS/results.jsonl`
(`misleading-error`, trial 1, 2026-09-18). Stage 0 validates the loop, not the harness, so
nothing is re-scored; any comparison that uses them must note that the plugin was off.

**Possible bias.** None toward Zero Code: the fix restores the configuration a user has, and it
applies to the baseline and every candidate alike. It was found by reading the adapter while
fixing an unrelated live test, not by inspecting a weak result.

## D13. The first Stage 1 screen runs in local mode, on four harnesses

**Made:** 2026-09-30, before any Stage 1 result.

**Decision.** The first screen compares current Zero Code, its frozen baseline, OpenCode 1.18.30 (its
upstream), Codex CLI 0.159.0 and Claude Code 2.1.245 on OpenBench's eight tasks, two trials each, in
local mode through the metering proxy.

**Why local, against D10.** The bench image installs only npm-published harnesses, so running Zero Code
in Docker means first building a container image of it from source. The owner asked for real results
over more infrastructure. Local mode is identical for every arm, each run gets a disposable workdir and
an isolated home, and the risk D10 names (an agent reaching the checker or leaving its workspace) applies
to every arm equally. These results direct Zero Code's development; they are not for publication until
rerun in Docker.

**Why these harnesses.** OpenBench ships adapters for these; Qwen Code has none, and writing one is not
worth it for a first screen. Codex now supports local models natively.

**Adapter patches (tools/openbench-patches/adapters/).** Each gets the local model through the proxy
with the settings the Zero Code adapter uses (262,144-token window, reasoning on, medium effort):
OpenCode through its model config and `--variant medium`; Codex as a direct Responses provider, since the
engine serves `/v1/responses` itself, with its window set; Claude Code at the engine's own `/v1/messages`,
with its background model aliases pointed at the one model the engine serves. The proxy now records
`/v1/messages` as well.

## D14. The first screen was stopped early, and the completion review was removed

**Made:** 2026-09-30, after 74 of 80 cells (webcore trial 2 not run).

**Result.** Pass counts: Zero Code baseline 15/15, Zero Code current 15/15, Claude Code 14/14,
OpenCode 14/14, Codex 9/15 (partial scores on taskflow and webcore, fast failures on two easy tasks).
Six of the eight tasks did not separate the arms on success; only taskflow and webcore did, and only
against Codex.

**What made current Zero Code slower.** Its completion review ran after every file-changing session,
so each cell stopped twice, and the model redid work in the review turn: on add-feature 13 and 17
requests and 4,257 and 8,810 output tokens, against 9 and 7 requests and about 1,400 tokens for the
baseline. No task went from fail to pass. The review was reverted (Zero Code `24ed8a206d`).
Current's webcore trial ran to the 1,500 s limit without stopping (34 requests, 46k output tokens,
against the baseline's 29 and 34k in 996 s); the review never ran there, and one trial cannot separate
that from run-to-run variance.

**For the next screen.** These eight tasks are mostly too easy to separate strong harnesses on success,
so differences show up as time and tokens. Harder tasks are needed for success to discriminate.
