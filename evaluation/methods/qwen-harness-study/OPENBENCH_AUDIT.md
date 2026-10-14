# OpenBench (coding-agent-harness) — Experimental Validity Audit

- **Repo:** https://github.com/minghinmatthewlam/openbench
- **Commit SHA:** `db193457a3d9128cd4d01fd839c2a890c186c9ac`
- **Commit date:** 2026-08-20 17:50:20 -0400
- **Clone date:** 2026-09-18 (`git clone --depth 50` into `vendor/openbench`, gitignored)
- **What it is:** A benchmark that compares coding-agent CLI **harnesses** (`codex`,
  `pi`, `opencode`, `cursor`, `devin`, an open-model `claude` route) against each
  other while holding the underlying model and a self-contained coding task
  fixed ("Harness Bench"); a task is graded only by an independent `checker.sh`
  exit code, never by the harness's own claim of success. A second, separate
  family ("Gateway Bench"/"Gateway Probe") holds the harness/model fixed and
  varies the API gateway/route instead. Confirmed as the intended
  coding-agent-harness target (not the chess-engine or inspect-ai projects of
  the same name) via `obench/adapters/*.py`, `tasks/*/checker.sh`, and the README.

Evidence format below is `path:line` against the cloned tree at the commit above.

---

## 1. Can the agent under test inspect the checker/grader?
**VALID WITH CAVEAT**
- `obench/workspace.py:materialize_workspace` stages only `task_dir/workspace/`
  (or a git-archived ref) into a fresh temp `workdir`; `checker.sh` lives in
  `task_dir`, a **sibling** of `workspace/`, and is never copied in.
- `obench/run.py:905 invoke_adapter(...)` passes the adapter only
  `instruction, workdir` — `task_dir` is never handed to the adapter/agent.
- `obench/run.py:1234 run_checker(task_dir, workdir, ...)` runs
  `task_dir/checker.sh` as a **separate host subprocess after** the adapter
  finishes, with `cwd=workdir` but the script itself sourced from `task_dir`.
- Docker mode bind-mounts only the staged `workdir` at `/work`
  (`obench/docker_exec.py:479-487`), not `task_dir`.
- Caveat: in `--exec local` mode there is no filesystem sandbox; a harness
  with unrestricted shell access could in principle discover `tasks/<name>/checker.sh`
  on the host if it already knows the repo layout. Isolation here is "never
  referenced," not "cannot possibly reach."

## 2. Can the agent inspect a golden/reference solution?
**VALID WITH CAVEAT**
- Same staging path as Q1: `solution/` is a sibling of `workspace/` inside
  `task_dir` and is only ever copied by `obench/workspace.py:450 overlay_solution`,
  which is called exclusively from admission/authoring tools
  (`obench/determinism_check.py:254`, `obench/admission_gate.py:167`,
  `obench/validate_tasks.py:120`) — **never** from `obench/run.py:run_cell`,
  the actual trial path. Same local-mode host-discovery caveat as Q1.

## 3. Is every run started from an identical repository state? How is reset implemented?
**VALID**
- `obench/workspace.py:_clear_dir` + `_copy_snapshot_into`/`materialize_git_workspace`
  wipe and refill a disposable dest per call; git mode uses `git archive`
  export (`obench/workspace.py:_git_archive_extract`) so the source repo/worktree
  is never mutated.
- `obench/run.py:run_cell` creates `workdir = tempfile.mkdtemp(...)` per cell
  (~line 1985) and `shutil.rmtree(workdir, ignore_errors=True)` in the
  `finally` block (~line 2226). Source under `tasks/` is read-only for the
  whole loop.

## 4. Can harness state leak between trials?
**BIAS RISK**
- `pi`/`claude`/`opencode` adapters build an isolated throwaway `HOME`/
  `CLAUDE_CONFIG_DIR` per invocation (`obench/adapters/claude.py:1-30` docstring;
  `obench/adapters/pi.py` isolated `iso_home`).
- Docker mode is one fresh container per cell (`obench/docker_exec.py:10`,
  "One `docker run --rm` per cell (container-per-run)").
- BUT `obench/ADAPTER_SPEC.md:44` states explicitly: *"`codex` / `cursor`: use
  the user's existing login as-is"* — i.e. those two adapters reuse the real,
  persistent `~/.codex` / `~/.cursor` config/session/cache directories across
  **every** trial in local mode, unlike the isolated-HOME adapters.

## 5. Can model-server state or cache create systematic bias (e.g. prefix caching across trials)?
**BIAS RISK**
- `tokens_cache_read`/`tokens_cache_write` are tracked per row
  (`obench/run.py` row dict, ~line 1935), confirming provider-side prompt
  caching is expected to occur, not defeated.
- No shuffle/randomization code exists anywhere in `obench/run.py`,
  `obench/run_matrix.py`, or `obench/suite_run.py` (grep for
  `shuffle|random.Random` is empty). Identical instruction text is resent every
  trial/harness for "the same model," so vendor-side prefix/response caching
  (OpenRouter, Z.ai, Moonshot, etc., all shared multi-tenant endpoints outside
  OpenBench's control) is architecturally possible and unmeasured.

## 6. Are infrastructure failures distinguishable from genuine model/agent failures?
**VALID WITH CAVEAT**
- `obench/failure_class.py:442 classify_failure()` implements a maintained
  taxonomy (`solved`/`infra`/`rate_limited`/`timeout`/`wrong_answer`/`stalled`)
  with explicit heuristics (`has_starved_output_budget`,
  `has_throttle_dominated_replies`, `has_rate_limit_marker`, `has_infra_marker`).
  A stall watchdog in `obench/run.py` (~lines 2015-2060) kills genuinely stuck
  cells using proxy-observed call-arrival liveness (not wall time), so a
  legitimately long turn isn't misclassified.
- The code's own comments document a real, previously-undetected bug: a naive
  rule excluded 78 real checker-judged failures across arms in proportion to
  how often each arm got throttled (`obench/failure_class.py:468-482`), which
  "flattered exactly the comparison we were trying to measure." It's fixed now,
  but the classifier is still transcript-regex/heuristic-based, so novel
  failure shapes can still be misclassified.

## 7. Are retries applied consistently across harnesses?
**BIAS RISK**
- `obench/adapters/pi.py:843-855` hand-tunes pi's model-call retry policy
  (`maxRetries: 5, baseDelayMs: 6000, provider.maxRetries: 1`) "from forensics
  on the laguna 429 storm."
- Grepping `codex.py`, `opencode.py`, `cursor.py`, `devin.py`, `claude.py`,
  `grokbuild.py` for equivalent retry config returns nothing — those 6
  adapters run whatever retry behavior their upstream CLI ships with by
  default, unverified and untuned.
- Separately, `obench/harbor_job.py:121 RetryPolicy` is a cell-level
  re-execution retry used only in the Harbor-native `suite_run.py` path
  (`obench/suite_run.py:311`), applied uniformly per job — that part is
  consistent, but it's orchestration-level retry, not the model-API retry
  behavior above.

## 8. Is model identity verified, or merely configured by string?
**VALID WITH CAVEAT**
- Each adapter's `MODELS`/`OPEN_MODELS` dict (e.g. `obench/adapters/pi.py:256`,
  `obench/adapters/codex.py:159`) is a hardcoded string → `{model_id, base_url,
  env_key, ...}` mapping.
- `obench/harbor_results.py:1326-1360 _validate_atif` cross-checks that a
  trial's self-reported trajectory `model_name` matches the harness's own
  `reported_model_name`/`lock_model`/`canonical_model` — this is **internal
  self-consistency** of the harness's own claims, not independent
  verification against the vendor. No code queries a `/v1/models` endpoint or
  compares against an external ground truth.

## 9. Are token measurements comparable across harnesses (who counts, and where)?
**VALID WITH CAVEAT**
- Primary numbers are self-parsed per-adapter from each CLI's own JSON usage
  output (e.g. `obench/adapters/claude.py` docstring: "Token accounting emits
  TOKEN_PARITY.md split fields... Falls back to top-level usage when
  modelUsage is absent"). `obench/TOKEN_PARITY.md` exists specifically to
  reconcile field-shape differences between harnesses by hand.
- `obench/proxy.py` is a stdlib counting HTTP proxy that can independently
  ledger tokens (`tokens_proxy_*` row fields) when active, but it's opt-in
  (`--proxy`) and gated by `proxy_supported_for_cell(...)`
  (`obench/run.py:~1875`) — not every harness×model cell gets the independent
  cross-check.

## 10. Are time measurements comparable?
**BIAS RISK**
- `wall_time_s`/`t_env_setup_s`/`t_agent_s`/`t_checker_s` are all computed by
  the same `time.monotonic()`-based code in `run_cell` for every harness —
  mechanically uniform.
- But `obench/run.py` (~lines 1903-1908) stores `"host": platform.node()` with
  an explicit unresolved comment: *"the estate is two hosts with different
  container runtimes... which makes cross-arm WALL TIME confounded and was
  impossible to detect from the rows themselves"* — citing a past case
  (tb-mid deepseek ran only on the machine with that API key). This is a
  documented, acknowledged, **still-open** confound in the codebase itself.

## 11. Are harness versions pinned?
**VALID WITH CAVEAT**
- Docker mode: `obench/docker/Dockerfile` bakes CLI version pins, audited via
  `obench/bump_clis.py` (`CliPin` dataclass, `/etc/openbench-cli-versions.json`).
  `obench/run.py:530 harness_version_for_source()` stamps
  `row["harness_version"]` from that in-container file, source="container".
- `--exec local` mode instead stamps whatever CLI version is currently
  installed on the operator's host ("host" source) — not pinned, can drift
  session to session.

## 12. Are prompts and tool definitions captured per run?
**BIAS RISK**
- OpenBench's own instruction text and the harness's final transcript/output
  are captured (`obench/run.py` `row["cmd"]`, `write_transcript`/
  `transcripts_dir`, ~lines 2085-2100, local-only persistence).
- The harness-**internal** system prompt and tool/function-definition JSON
  actually sent to the model (codex's own system prompt, pi's built-in tool
  schema, opencode's tool set, etc.) are each CLI's own black box; OpenBench
  stores only what the adapter chooses to surface in `output_tail`/
  `full_output`/`usage_raw`. Silent upstream prompt/tool changes between CLI
  versions are invisible to the recorded evidence.

## 13. Can execution order introduce thermal/cache/load bias (is order randomised or blocked)?
**BIAS RISK**
- No shuffle/random-order code in `obench/run.py`, `obench/run_matrix.py`, or
  `obench/suite_run.py`. Cells run in the matrix's natural
  (harness, task, trial) enumeration order.
- `obench/docker_exec.py:471-473` does set a per-cell `--cpus` quota
  ("Bound each cell's CPU quota so co-tenant host load cannot starve a cell")
  which bounds co-tenant interference but does nothing about ordering.

## 14. Are repeated runs of one task treated as independent tasks in the statistics?
**BIAS RISK**
- `obench/stats.py:90 wilson_ci(successes, n)` is called with `n` = the pooled
  trial count at a (task, harness, model) group
  (`obench/report.py:420-440`, `obench/leaderboard.py:576`) — repeated trials
  of the identical task are pooled as i.i.d. Bernoulli draws for a binomial CI.
  Standard practice for pass@k-style reporting, but the independence
  assumption is not verified given Q5 (no cache-defeat) and Q4 (shared-home
  leakage for codex/cursor).

## 15. Can one harness receive information unavailable to another?
**VALID WITH CAVEAT**
- The benchmark-provided task information (`instruction` text + staged
  `workspace/`) is symmetric across harnesses — `invoke_adapter` hands every
  adapter the same two things.
- But per-adapter environment differs materially: codex/cursor inherit the
  user's full real login/session (`ADAPTER_SPEC.md:44`) while pi/claude/
  opencode get an isolated throwaway HOME; some model routes go direct to a
  vendor, others (`claude.py:133`, `gpt-5.6-sol`) through a local CLIProxyAPI
  bridge with different latency/caching. Each CLI's own default system
  prompt / repo-context auto-discovery is not equalized by OpenBench.

## 16. Can network access contaminate task validity?
**BIAS RISK**
- `obench/docker_exec.py` (`docker run` argv assembly, ~lines 468-530) never
  sets a `--network` flag — Docker's default bridge network applies, i.e. full
  outbound internet access from inside the container unless the base image
  itself blocks it. `--exec local` obviously has full host network access.
  No egress allowlist/firewall code exists in `docker_exec.py`.

## 17. Are untouched-workspace negative controls validated?
**VALID (admission-time only)**
- `obench/admission_gate.py:154-165 validate_equivalence()` runs
  `determinism_check.run_checker_once(task_dir, overlay_solution=False)` and
  raises a HARD finding if the bare workspace checker passes
  (`validate_tasks.workspace_passed`, ~line 173). This is the untouched-
  workspace negative control, but it is a **task-authoring/admission gate**
  (`obench/admission_gate.py`, `obench/validate_tasks.py`), not something
  `obench/run.py:run_cell` re-checks on every benchmark execution.

## 18. Are known-good positive controls validated?
**VALID (admission-time only)**
- Same function: `sol = ...run_checker_once(task_dir, overlay_solution=True)`
  asserts `sol["exit_code"] == 0` and `sol["parsed_score"] ≈ 1.0`
  (`obench/admission_gate.py:168-175`), using `obench/workspace.py:450
  overlay_solution` to overlay the golden `solution/` dir. Same admission-
  time-only caveat as Q17.

## 19. Can an agent alter files that affect verification?
**VALID WITH CAVEAT**
- `checker.sh` and `solution/` live in `task_dir`, never bind-mounted or
  copied into the agent's `workdir` (see Q1/Q2). Docker mode mounts only
  `workdir` at `/work`, plus read-only runner-code mounts
  (`obench/docker_exec.py:480-487`, `:ro` suffix). Same local-mode host-
  discovery caveat as Q1/Q2 applies.

## 20. Can results be gamed by deleting tests or modifying fixtures?
**BIAS RISK**
- `run_checker()` (`obench/run.py:1234`) always executes the pristine
  `task_dir/checker.sh`, never a copy from inside `workdir`, so the grader
  **script** can't be tampered with by the agent.
- But the checker runs with `cwd=workdir` (`obench/run.py:1268`), so if the
  checker script itself just re-runs a test suite that lives inside the
  workspace (e.g. `tasks/fix-failing-test/checker.sh` is literally
  `python3 -m unittest -v` with **no assertion that the expected test file/
  count still exists**), an agent that deletes or guts the test file inside
  its own workspace copy can trivially make the checker report a false pass.
  The admission-gate polarity check (Q17/Q18) tests bare-workspace-fails /
  solution-passes, not adversarial test-deletion, so this class of gaming is
  not caught by the gate for checkers written this way.

---

## Reusable / Must Change / Must Not Rely On

| Component | Verdict | Notes |
|---|---|---|
| Workspace materialize + reset (`obench/workspace.py`) | **Reusable** | Clean fresh-copy-per-trial, source never mutated |
| Checker isolation from agent workdir (`obench/run.py:run_checker`, `invoke_adapter`) | **Reusable** | task_dir never passed to adapter |
| Failure taxonomy (`obench/failure_class.py`) | **Reusable** | Battle-tested, evidence-driven, still heuristic |
| Resume-by-run_id (`obench/run.py:load_existing_run_ids`) | **Reusable** | Harness-agnostic, deterministic |
| Adapter contract (`obench/ADAPTER_SPEC.md`, `obench/adapters/*`) | **Reusable** | Minimal stdlib contract, easy to extend |
| Admission-time pos/neg controls (`obench/admission_gate.py`, `determinism_check.py`) | **Reusable** | Only runs at task authoring, not per-trial |
| Wilson CI stats helper (`obench/stats.py`) | **Reusable** | Standard, but assumes trial independence |
| Local OpenAI-compatible model support | **Must Change** | Hardcoded per-adapter allowlists; needs a new dict entry |
| Execution order randomization | **Must Change** | None exists anywhere in the run loop |
| Docker network egress control | **Must Change** | No `--network` restriction set |
| Retry-policy parity across adapters | **Must Change** | Only `pi.py` is hand-tuned |
| Checker robustness to test-deletion | **Must Change** | Sample checker has no fixture-presence assertion |
| Cross-host wall-time confound | **Must Change** | Acknowledged unfixed in `run.py` comments |
| Local-mode `harness_version` as "pinned" | **Must Not Rely On** | Host probe, drifts session to session |
| Wall-clock across differently-provisioned hosts | **Must Not Rely On** | Explicitly documented as confounded |
| Token counts as apples-to-apples across harnesses | **Must Not Rely On** | Self-reported unless proxy path confirmed active |
| Trial independence in reported CIs | **Must Not Rely On** | No cache-defeat / no order randomization |
| Model-identity strings as verified | **Must Not Rely On** | Self-reported/config-consistency only, no external check |
| Harness-internal system prompts as captured | **Must Not Rely On** | Not extracted; each CLI's own black box |

---

## Local model support (step 4)

**Can it drive `http://127.0.0.1:11234/v1` with an arbitrary model id today?**
No, not without a code change. Every adapter's `MODELS`/`OPEN_MODELS` dict
(`obench/adapters/*.py`) is a hardcoded allowlist keyed by specific canonical
model strings, each mapped to a fixed `base_url` + `env_key` + vendor compat
flags. An unrecognized model string hits `_unsupported()`
(e.g. `obench/adapters/pi.py:267`) and returns `completed=False`.

**What would have to change:** add one new dict entry to an adapter's
`MODELS`/`OPEN_MODELS` mapping with `base_url="http://127.0.0.1:11234/v1"`,
an arbitrary `model_id`, and an `env_key` supplying a placeholder API key
(most local OpenAI-compatible servers accept any string). There is direct
precedent for exactly this pattern already in the repo:
`obench/adapters/claude.py:133` routes `"gpt-5.6-sol"` through a **local**
bridge daemon at `http://127.0.0.1:8317` (documented in
`docs/subscription-bridge.md`), and `obench/adapters/codex.py:421,436` has an
explicit `-c openai_base_url=...` / `model_providers.{prov}.base_url=...`
override mechanism. Best candidates to extend: `opencode.py` (native
OpenAI-compatible `baseURL` provider field, line ~178) or `codex.py` (native
OpenAI Chat/Responses-compatible base_url flag). No change to the core runner
(`run.py`/`workspace.py`/`docker_exec.py`) is needed — only the adapter-level
dict entry, plus a docker `candidate_pass_env`/env-passthrough allowlist entry
if run in `--exec docker` mode.

**Does it support adding new harness adapters?** Yes, explicitly designed for
it. `obench/ADAPTER_SPEC.md` defines a minimal stdlib-only contract: a module
with `NAME: str`, `MODELS: dict`, and
`run(instruction: str, workdir: str, model: str, timeout_s: int) -> dict`
returning `completed`/`error`/`output_tail`/`tokens`/`turns`/`cmd` (+ optional
`full_output`). `obench/run.py:invoke_adapter` (~line 905) just imports and
calls this contract by harness name.

**How much code does one adapter take?** Shipped adapters range 298-914 lines
(`wc -l obench/adapters/*.py`: cursor.py 298, devin.py 310, opencode.py 419,
claude.py 449, grokbuild.py 475, codex.py 553, pi.py 914); the thin
`codex_v1.py`/`codex_v2.py` wrappers are 35 lines each, delegating to a shared
179-line `_codex_ablation.py`. Most of the line count in the large ones is
auth isolation, per-vendor compat-flag tables, and adapter-specific
token-usage JSON parsing — the bare contract (`NAME` + `MODELS` + `run()`)
could plausibly be satisfied in well under 150 lines for an
already-OpenAI/Anthropic-compatible target.
