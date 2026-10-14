# mini-swe-agent — source audit

HEAD `04d809ceab9d` (2026-09-03), `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/mini-swe-agent`.
All claims from reading implementation; docs-only claims are flagged. Comparison frame: Zero Code baseline
(01-ZERO_CODE_BASELINE.md, HEAD 123cb9262f).

## Architecture

Three injectable roles wired by config YAML (fire CLI in `src/minisweagent/run/mini.py`, 109 lines):

- **Agent** — `agents/default.py` (190 lines total): `DefaultAgent` owns the message list, cost counter, limits.
- **Model** — `models/litellm_model.py` (164 lines) + text-based/XML variants, retry util, Anthropic cache-control/thinking-block utils.
- **Environment** — `environments/local.py` (92) / `docker.py` (161) / singularity / bubblewrap / modal extras.

The famous loop: `run()` `agents/default.py:96-124` — a `while True` calling `step()` (= `query()` +
`execute_actions()`, `:126-157`), breaking when the last message has `role == "exit"` (`:122`). Core loop
machinery ≈ 70 lines; the file adds config, template rendering, and trajectory serialization.
**Verdict on "100 lines": source-verified in spirit** (README.md:26, docs/index.md:33 claim "~100 lines of
python for the agent class"); the honest number is ~70-line loop + ~120 lines of config/serialization in the
same file, with the surrounding package (models, 6 environments, benchmark YAMLs, tests) of normal size.

## Key techniques (file:line)

- **Single-tool contract.** Exactly one tool, `bash {command: string}` — `models/utils/actions_toolcall.py:11-27`.
  Every response MUST contain ≥1 bash call; multiple calls allowed when independent
  (`config/benchmarks/swebench.yaml`). Any other tool name or missing `command` → `FormatError` with a
  rendered corrective template (`:40-74`). No edit/read/search tools at all: file work is `sed`, heredoc
  `cat <<EOF`, `nl -ba | sed -n` — taught by prompt examples (`config/mini.yaml:55-100`, incl. a Darwin
  `sed -i ''` conditional).
- **Termination is an output sentinel, not a tool.** `echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` — detected
  by the *environment* on stdout (`environments/local.py:45-56`), raising `Submitted` → `role:"exit"` with the
  trailing stdout as `submission`. Multi-file "final answer" is just the working tree; the harness diffs it.
- **Cost-based limit is a hard exit, not truncation.** `cost_limit`/`step_limit`/`wall_time_limit_seconds`
  checked before each query → `LimitsExceeded`/`TimeExceeded` exit messages
  (`agents/default.py:132-147`). There is **no compaction at all**: full history goes out every call;
  `ContextWindowExceededError` is in the *abort* list (`models/litellm_model.py:50-57`) — a lost run, not a
  recovered one. (v1.x-era "drop old messages on cost limit" is gone at this head; `git log -S` finds nothing.)
- **Truncation happens per-observation only.** Output >10,000 chars → head 5000 + tail 5000 +
  `elided_chars` + `"Output too long"` warning, entirely in the Jinja observation template
  (`config/mini.yaml:112-128`). Zero-equivalent: their spill-file strategy replaces this with a pointer.
- **Format-error loop with truncation-awareness.** Billed-but-unparsed responses keep their cost (`:100-103`,
  `models/litellm_model.py:87-98`); the error template branches on `finish_reason == "length"` to tell the
  model it was cut off rather than malformed (`config/mini.yaml:129-149`); 3 consecutive format errors →
  exit `RepeatedFormatError` (`agents/default.py:100-114`).
- **Budget countdown prompts.** ProgramBench config injects into observations: <20 steps left → "abort
  debugging, ensure it compiles, write AGENT_REPORT.md handoff"; <10 min wall time → same wrap-up
  (`config/benchmarks/programbench.yaml:241-258`). Cheap, effective long-horizon pressure note.
- **Weak-model path.** `config/mini_textbased.yaml` + `models/openrouter_textbased_model.py` +
  `models/utils/actions_text.py`: exactly one action per response in a unique-fence
  ` ```mswea_bash_command ` block — deliberately *lowering* the format bar instead of repairing formats.
  Anthropic thinking-block reordering and `set_cache_control` markers for prompt-cache stability
  (`litellm_model.py:76-79`); exponential retry with typed abort list (`models/utils/retry.py`).
- **Benchmark discipline.** Trajectory format `mini-swe-agent-1.1` serializes full agent/model/env config,
  per-call cost, and raw responses into one JSON per run (`agents/default.py:159-180`); per-instance and
  aggregate cost tracked (`:108-126`, `GLOBAL_MODEL_STATS`); benchmark prompts are versioned config, not code
  (`config/benchmarks/swebench*.yaml`, incl. xml/backticks modal variants). **">74% on SWE-bench Verified"
  and the leaderboard are documented-only** (docs/index.md:35) — no eval code or scores in the repo.
  **Contamination: zero mentions anywhere in src/ or docs/** — contamination awareness is not an
  in-repo concern; reproducibility (pinned prompt + trajectory) is what the repo actually ships.

## Strengths

- The whole control-flow spec fits in your head: exit-by-sentinel, limits-as-exceptions, format errors as
  re-prompts, everything else is Jinja templates. Forkability is the actual product.
- Accounting honesty: cost attributed even when parsing fails; abort-vs-retry exception taxonomy; per-run
  config snapshot makes every trajectory reproducible — the eval-harness discipline Zero's own eval package
  should envy.
- Prompt-level budget countdowns and handoff-document instructions are near-free long-horizon machinery.

## Weaknesses

- No context management strategy beyond per-observation head/tail: dies at context overflow (documented
  behavior — the abort list). Unusable for long interactive sessions.
- Stateful-shell prohibition (`every action runs in a new subshell`, `config/mini.yaml:39`) is pushed onto
  the model; no cwd/env continuity service.
- No permissions, no approvals, no interruption semantics worth the name (interactive.py is a stdin REPL
  novelty), no resumability: a crash loses the run (mitigated only by after-each-step trajectory save,
  `default.py:121`).
- Multi-file work is entirely model responsibility — no atomic edit, no read-before-edit, no diff validation;
  on weak models `sed`-based edits are exactly where silent corruption happens.

## Works vs unproven

- **Source-verified working (as an eval harness):** loop, sentinel submission, cost/step/time limits,
  format-error loop, text-based weak-model mode, docker/local exec, trajectory serialization.
- **Documented-only:** SWE-bench score claims, "starts faster than Claude Code", contamination posture.
- **Unproven (and out of scope here):** anything interactive, resumable, or multi-session.

## The thesis, honestly graded

"Most complexity in bigger frameworks is unnecessary" is **true for the loop skeleton and false as a general
claim**. mini-SWE can be minimal because of five eval-harness privileges Zero does not have:

1. **One process = one disposable task**, ground-truth exit (patch diff) — so durable `session_input`
   admission, crash repair, and V2 context epochs are load-bearing for Zero but dead weight here.
2. **No human in the loop** — so no approval/auto-approval policy, no steering vocabulary, no
   background-job notifications; Zero's permission and steer/queue machinery is answering a question mini
   never gets asked.
3. **Fixed short horizon** — so no compaction needed; Zero's compaction/cache-continuation work exists
   precisely because sessions outlive the context window.
4. **Sandboxed by construction** (docker) — so no tool-permission parsing, no sandbox-vs-Local duality;
   Zero's tree-sitter bash permissions buy what mini's Dockerfile buys.
5. **Failure is cheap and re-runnable** — abort-on-overflow is acceptable when you rerun the sweep; in a
   product a lost session is lost work, so Zero's overflow recovery/compaction is load-bearing.
   Conversely, Zero's checkpoint system has no mini equivalent because the eval harness resets the container.

## What Zero Code should learn / reject

- **Adopt:** head/tail + `elided_chars` observation envelope (Zero has spill files but should carry the
  elision *count and "redirect output to a file" hint* into the observation itself); the
  `finish_reason==length`-vs-format-error classification in edit/tool failure guidance (Zero's output-length
  handling should feed the same distinction into the near-miss matcher hints); step/time countdown
  pressure-notes with an explicit handoff artifact (AGENT_REPORT analogue — Zero's pressure notes are
  token-based; adding "N provider turns left, write a handoff" is cheap); billed-cost-on-format-error
  accounting in the telemetry ledger.
- **Adopt as eval-harness policy:** pin agent prompts as versioned config artifacts alongside
  `zero-eval` runs, snapshot full config into every trajectory (partially true already — verify).
- **Reject:** single-bash-tool minimalism for the product (Zero's edit-matcher reliability layer exists
  because weak/local models mangle sed round-trips, and interactive users need read-before-edit guards);
  abort-on-context-overflow; environment-side sentinel submission (fine for evals, wrong UX); zero-resume.
