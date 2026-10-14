# 06 — Experiment design (and results actually obtained)

Goal: isolate **harness** effects from **model** effects, cheaply, using machinery that already exists
(`packages/zero-eval` with hidden-test grading, blind review protocol, `--context` override, `--config` A/B, the
telemetry ledger at `~/.local/share/zero/telemetry.db`, and `~/DevData/Projects/pi-eval`). Every planned run names
its confounds. Nothing decorative: if an experiment cannot change a roadmap decision, it is not scheduled.

## 1. Results actually obtained during this research (not planned — done)

**R1 (pre-existing, reused): same-model difficulty ladder, 2026-10-09/10**
`docs/research/2026-10-09-qwen-vs-sonnet-ladder/` — local Qwen 3.8 Flash Next in Zero Code vs Sonnet 5.5 in Claude
Code via cleanroom copies of identical fixtures; hidden tests + blind randomized review.
- Hidden tests tied at every tier; blind quality gap favored Sonnet and *widened with task size*
  (tier 3: 34 vs 20/35; tier 4: 38 vs 20/40; Mealbook 40 vs 27/40). Reviewer noise ±4/40 — gaps under that are
  inconclusive.
- Failure mode localized: explicit specs followed perfectly (all six `long-csv-sum` builds perfect); failures were
  *unstated pitfalls* (prototype-key collisions, destructive state resets, DST date bugs, truncated pipe output).
- Harness economics: 80–86% of wall time is decode; **Zero Code's own loop overhead <1%**; whole-file rewrites 0–6%.
- Measured harness changes: design-review subagent improved blind scores (tier 4: 29 vs 22) at ~2× time;
  robustness instructions fixed hidden-test misses (kv-cli 14/14+4/4 in 3/3 runs) at ~2.5× time, no gain where the
  spec was explicit; low reasoning effort was *slower and worse* (87.6 vs 69.2 min; 26 vs 30).
- Interpretation: at fixed model, harness changes move quality measurably but pay in tokens/time — the exact axis
  the roadmap prioritizes (autonomous success per unit of inference).

**R2 (this run): Antigravity black-box, 2026-10-13** — full probe log in
`02-COMPETITOR_AUDITS/antigravity-cli.md` (P1–P17): binary is Go-1.28-from-google3, auth-walled for every model
turn; loop internals unknowable without credentials. Owner-deleted binary; `agy.strings` retained. Further probes
**blocked** under the binary-download rule.

**R3 (this run, failure reported honestly): compaction stress rerun, 2026-10-14**
`2026-10-13-context-stress.md/.json` in this directory: `zero-eval --set long --only long-kv-cli --context 32000
--context-output 4096` failed at boot — `session.create` returned HTTP success-with-empty-error after a ~60 s hang
against a freshly spawned `zero serve` (server log shows no error; engine adopted fine). Not diagnosed further
(this run is research-only); logged in ZERO_CODE_ISSUES.md as an eval-boot bug on `long-horizon` at `44714a1abb`.
The forced-compaction cell itself is now measured — see R5 (listed ahead of R4 for chronology).

**R5 (this run): E3 multi-compaction cell, 2026-10-14**
`2026-10-14-e3-qa-spec-24k.md/.json`: `long-qa-spec` (90-min fixture cap, main task + followup feature) at forced
**`--context 24000`** (≈9% of the engine's 262k), output reserve 4096, AC power, engine shared with the research
session (all timings confounded). Result: **timeout at the 5,401 s cap — 0/1 passed**, 207 tool calls (1 failed;
108 read, 71 bash, 11 edit), **72 compactions**, prefix hit **34%** (vs 90% in the no-compaction R4 run),
read-before-edit 11/14, 57.5 tok/s. The run reached followup 1 (main spec done) at 07:33 and was mid-followup when
the cap fired — so the failure is **compaction throughput, not capability**: one compaction every ~75 s of
wall-clock, each costing summarizer prefill+decode on a contended engine, with cache-hit collapsing to a third
despite the cache-continuation machinery. Honest reading: at 24k the harness spends more context budget managing
context than using it, and 0/1 pass under a cap it meets comfortably at full window; the recall-probe cell was not
reached. This *raises* the E3 threshold's importance (a realistic small window like 64k is the make-or-break cell)
and converts two roadmap items from "nice" to "prerequisite for small-window use": measured token-ground-truth
triggering with masking (Gemini pattern, 05 F3) and F3's floor/masking stack before any local-model product story.
Re-run at 64k on a quiet engine overnight is the remaining planned cell (needs the engine not to be shared).

**R4 (this run, rerun after owner fix `dc49a6f04a`): context-stress partial, 2026-10-14**
`2026-10-14-context-stress.md/.json`: same command, now booting in ~2 s. Result: **long-kv-cli pass in 267 s, 1 turn,
10 tool calls (0 failed), 0 guard refusals, prefix hit 90%, read-before-edit 1/4, 59.1 tok/s** (AC power; timing
confounded — research session shared the engine). Two findings: (1) at `--context 32000` this fixture **never
compacted** — too short to stress the window, so E3's compaction cell needs `long-qa-spec`/`long-inventory-api` or
a 16k override to fire; the E3 design is updated accordingly below. (2) `read-before-edit 1/4` is direct in-house
evidence for the freshness-enforcement item (Roadmap Phase 2; crush's read-table precedent in 05 F2). E3 window-fit
+ boot fix measured here; the multi-compaction cell result is R5 (above).

## 2. Principles

1. Same model + same settings across harness arms (engine `qwen3.8-flash` local for open arms; one frontier model
   via already-signed-in CLIs where black-box allows).
2. Same starting repo state (eval fixtures give this), same task text, comparable tool access documented per arm.
3. Budgets expressed in **provider tokens and prefill/decode time**, never dollars (local cost = $0).
4. n≥3 per cell where generation randomness can flip the outcome (grading is deterministic tests + randomized blind
   review with the ±4/40 noise floor acknowledged; a cell conclusion needs the gap outside noise or a test-pass flip).
5. One background job at a time on the shared engine; **all wall-clock timings while this research session is
   active are marked confounded**; clean-timing runs scheduled for a quiet machine.
6. No new frameworks: zero-eval gains an `external` adapter that shells out to installed CLIs (claude/codex/
   opencode/gemini) with per-arm tool-capability notes, nothing more. pi/Aider arms run from source with
   `bun`/`uv` against the local engine **only after owner approval** (dependency downloads).

## 3. Planned experiments (each maps to a roadmap gate)

**E1 — Minimal-vs-rich toolset (decides F4/synthesis S1).** Same fixture (js-pagination-bug, py-lru-feature,
long-kv-cli), same local model, three Zero arms: (a) default surface, (b) `coding-minimal` profile (bash/read/
edit/write), (c) minimal + tool_search deferred. Metrics: hidden-test pass, tool-error rate (ledger `tool_calls`
flags: duplicate read, noop edit, invalid args), tokens in, prefill ms, steps. Threshold: (b) within 1 pass-fail of
(a) with >25% fewer declared tokens ⇒ converge tool surfaces. n=3.

**E2 — Edit-format A/B on the local model (decides which ladder from the matrix to invest in).** Same repair
fixtures (js-cli-json-flag, py-duration-bug), arms: (a) Zero edit cascade (current), (b) cascade with fuzzy tiers
disabled, (c) apply_patch for all models (ungate), (d) whole-file. Metrics: edit-attempt success rate by tier
(telemetry strategy logging), corrupted-region incidents (test diff inspection), wall time. Threshold: fuzzy tiers
pay for themselves only if their rescued-match count × task-success exceeds their corruption incidents; today that
ratio is *unmeasured anywhere in the industry* — this makes it measured for our model family. n=3.

**E3 — Context-pressure stress (partially run — R4 window-fit pass, R5 24k cell timeout; remaining planned cell:
64k on a quiet engine).** `long-qa-spec` +
`long-inventory-api` at `--context` 32k/16k (R4 showed `long-kv-cli` at 32k never compacts — fixture too short),
vs native 262k. Metrics: compactions per run, recall-probe score (`retention` answer key), hidden tests, `prefix_broken`
count, guard refusals, goal iterations. This validates the compaction stack for small-window local models — the
exact condition of E5's target users. Threshold: recall ≥80% and pass parity vs native window at 64k; below that,
phase-2 context work (08) gets priority bump. n=3.

**E4 — Harness-vs-model, same task, two harnesses, one local model (the core isolation).**
Arms: (a) Zero Code (V1 live path), (b) OpenCode (`opencode` CLI 1.18.30 pointed at the local engine — config only,
no download), (c) cleanroom Claude Code subagent with local model (existing `cleanroom.ts` route), (d) blocked
until owner approval: pi and Codex from source (dependency downloads). Fixtures: one bug fix, one multi-file
feature (ladder-t3-split), one self-inflicted-bug recovery (long-live-bugs variant). Same task text, documented
per-arm tool differences. Metrics: pass, steps, tokens, tool errors, compactions. This is the experiment the whole
survey lacks (every competitor claim mixes models); even 2 arms × 3 tasks × n=3 settles several matrix rows.
Timing confounded unless quiet-night; pass/fail is timing-independent.

**E5 — Weak-model lift.** Repeat ladder tier 2/3 on a deliberately weaker local model (already on the machine:
`PocketAiHub/Qwen3.8-27B-Abliterated-MLX-8bit` needs engine load only — no download; or existing smaller mlx
catalog entries) under (a) default harness, (b) harness + the model-adaptation package from 09 phase G
(data-driven per-model tables, format-bar reduction, budget auto-sizing). Threshold: ≥1 tier improvement or ≥30%
fewer failed tool cycles per solve.

**E6 — Done-ness predicate pilot (decides S5).** Arm (b) = `--config` injected completion-criteria checklist +
idle-time verifier subagent (hidden `--verify` command gate, mirroring zero-eval's own `verified_after_timeout`
diagnostic) vs arm (a) current default bundle — corrected 2026-10-14: arm (a) is no longer bare finish=stop,
it includes goal todo-continue + guard unfinished-reply continue + design gate (01 Part D); add arm (a0)
`ZERO_DISABLE_BUNDLE=1` for the bare-loop baseline — on tasks prone to false completion (the ladder's unstated-
pitfall class + a new false-completion fixture). Metrics: false-completion rate (claims vs hidden tests), wasted
continuation turns, time. Threshold: halve false completions at <40% time overhead.

**E7 — Mid-run steering fidelity.** Inject one steer per run at a fixed step during long fixtures; arms V1 (live)
vs V2 (`ZERO_EXPERIMENTAL_SESSION_V2`, using eval config) if E4 shows V2 viable; measure steer honored before next
provider call, lost-input rate, allowance resets (ledger + event log). Regression-tests the delivery vocabulary
before it becomes the default path.

**E8 — Resume-after-interruption.** Kill `zero serve` mid-fixture; resume; compare completed work state vs
uninterrupted run (hidden tests + diff). Arms V1 vs V2. Threshold: V2 recovers eligibility without duplicating
promoted work; V1 must show where it loses turns. Directly tests baseline findings 2–3.

## 4. Blocked (recorded, not fudged)

- Antigravity task runs: needs account sign-in + binary re-download (owner rule) → **blocked**.
- Gemini CLI task runs with the real product line: same auth wall as installed `gemini` 0.36.0 requires key/login
  for model calls; running it against the *local* engine via `GOOGLE_GEMINI_BASE_URL`-style overrides would need
  npm package installs beyond what's cached → **blocked pending owner approval**.
- Long benchmark runs on paid CLIs (claude/codex full ladders): short probes done inside audits; benchmarks need
  the owner's go per operating notes → **deferred**, not counted.
- Any arm requiring `npm/pip install` (aider, pi, openhands, mini-swe from source): **blocked pending approval**;
  each is a one-line ask when the owner is ready.

## 5. Resource plan & scheduling

E1/E2/E6 run sequentially overnight on AC power (battery halves decode — ladder-documented): ~2–4 engine-hours each
at tier-2/3 sizes. E3 is one fixture-set (~3 engine-hours). E4 doubles when a second arm is added; start with
(a)+(b) only. E5 waits for E1–E3 conclusions. All arms write the standard eval report + ledger rows; the ledger's
`prefix_broken`, `duplicate read`, `noop edit`, decode-delta columns are the shared observability substrate —
adding per-component prompt-token breakdown (09 phase O) is a prerequisite for reading E1/E3 token claims at
granularity; E1/E3 proceed with totals meanwhile.

Success criterion for the program: by its end, every Phase-1 roadmap item in 09 has a measured before/after in
this directory, and every "X does it better" claim in 04/05 that this program tested is either confirmed, refuted,
or explicitly still-open in the ledger.
