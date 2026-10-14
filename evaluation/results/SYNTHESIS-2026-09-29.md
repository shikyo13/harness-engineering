# Zero Code eval synthesis, 2026-09-29

Read-only synthesis of what has been measured. Sources: `docs/evals/*.md` and the JSON files those cite, `long-horizon-2026-09-23/`, `memory-pilot/`, `memory-desktop/`, `docs/handover/2026-09-19-codex-handover.md`, and `git log main..long-horizon`. Paths below are relative to `docs/evals/` unless stated. Numbers are quoted as the docs give them. **INFERRED** marks anything I computed or concluded rather than read. **COMMIT-ONLY** marks a measurement that appears only in a commit message, with no eval doc behind it.

Regime of the evidence, stated once: one model (Qwen3.8-Flash-Next 4-bit MoE on mlx-serve 26.9.2, M4 Max 128 GB). Cross-harness comparisons were all run 2026-09-16/17 on Zero build `0.0.0-main-202609162231` or later 2026-09-17 rebuilds, on tasks of 5 s to 400 s, mostly one to three trials. The only long-horizon (multi-compaction) evidence is 14 runs on 2026-09-22/23 with the window capped at 64k, Zero only, one trial per arm per task (two on inventory). No cross-harness comparison has been re-run on the `long-horizon` branch code.

---

## 1. Measured wins

Strictness key: **1 run** = a single trial; **n=k** = k repeats of the same cell; **bundled** = the change shipped with others so the number is not attributable to it alone.

| # | Change | Number (as documented) | Source | Evidence base | Verdict |
|---|---|---|---|---|---|
| 1 | Long-horizon work bundle (baseline `8c1fb77e16` to "after" `c1049454ec` + `632e803b5d`) | Baseline failed 2 of 3 tasks (inventory timeout at 90 min "still on its first request"; qa-spec missed a hidden sort check); after passed 3 of 4 runs. Recall 9/11 and 8/10 (baseline) vs 11/11, 10/10, 12/12 (after). | `2026-09-23-long-horizon-results.md` | 1 baseline run per task; after: 1 run each on live-bugs and qa-spec, 2 on inventory (pass in 73 min, fail in 88 min). Bundled (compaction reasoning, re-read warning, user-message carry, message-replay fixes, and more). | Direction plausible, size not established. The doc itself says arm-to-arm comparisons "are weaker". Baseline-side confounds in section 5. |
| 2 | Compaction summary uses the model's lightest reasoning variant (`c1049454ec`) | Baseline 9 compactions ~27 min of 90 (136 to 252 s each, from the baseline's session log). After: 7 compactions/12 min, 10/21 min, 4/9 min, 3/7 min, 11/20 min on inventory. | `2026-09-23-long-horizon-results.md`, commit body | Baseline per-compaction figure from 1 run (9 compactions). After from 5 inventory runs. Bundled with other changes but the mechanism is direct. | Real but modest. INFERRED per-compaction averages for after runs: 103, 126, 135, 140, 109 s (min/compactions from the table), i.e. about 1.7 to 2.3 min, not uniformly "1 to 2 minutes" as the doc says. Baseline average 180 s. |
| 3 | Fewer repeat reads after the "warn before compaction, do not re-read" note (`ed3c6537ff`) | Inventory reads (repeats): baseline 100 (75) vs after 50 (20), 61 (29), tool search 46 (8), 41 (17), repo map 32 (10). | `2026-09-23-long-horizon-results.md`; `duplicate_reads` in `long-horizon-2026-09-23/lh-*.json` agrees (75 vs 20/29/8/17/10) | 1 baseline run vs 5 after-family runs on one task. On live-bugs baseline 9 (1) vs 10 (1), qa-spec 31 (8) vs 15 (1). | Consistent direction across 5 runs on inventory; baseline is a single run. |
| 4 | MCP tool search, now automatic (`43e1546883`, threshold lowered to 5% by `2241ea5edb`) | Compactions inventory 7 and 10 to 4 and 3, qa-spec 3 to 1; passed 4 of 4 runs; wall inventory 65 and 48 min vs 73 and 88 for "after"; qa-spec 32 to 14 min. | `2026-09-23-long-horizon-results.md`; per-run JSON | 2 runs on inventory, 1 on each other task. Live-bugs (1 compaction) no difference. | Weak. Same-arm inventory runs already differ by 15 min and by pass/fail. Also see section 5: the tool-search runs never called `tool_search` or `tool_exec` (`by_tool` in the JSONs), so the win is the removal of unused schema, not a demonstration that search works. |
| 5 | Tool search, mechanical effect only | Per-request schema 35,084 to 22,417 bytes; first-turn prompt 11,055 to 8,050 tokens; total prompt tokens 174,771 to 104,210; cache hit 0.943 to 0.959. Wall 26.2 (20.8-43.2) vs 27.2 (25.5-38.0) s, "indistinguishable". | `2026-09-17-tool-search-ab.md` | n=3 per arm, one task (`fix-failing-test`), 24-tool filler MCP catalog | Mechanism proven; no timing benefit and no cost measured. Commit `2241ea5edb` (COMMIT-ONLY): the user's real 66-tool catalog makes a new session's first prompt 32.6k tokens (106 KB tools); with tool search "the same first prompt is 11.8k tokens with 20 tools". |
| 6 | Prompt-cache reuse within a project | Cold harness turn 30,818 ms (0 of 7,833 cached) vs warm same project 1,513 ms (7,802 of 7,833 cached), "roughly 20 times faster". Later: 78.5 s cold vs 1.3 s warm (31,704 of 31,735 cached); 9.2 s vs 1.4 s (see contradiction on "cold"). | `2026-09-14-qwen3.8-flash.md`; `2026-09-16-zero-speed-controlled.md`; `2026-09-17-speed-repeat.md` | Three separate runs, same direction | Solid as an engine property; the win belongs to a stable prefix, not to a Zero change. |
| 7 | `coding-minimal` tool profile (`ZERO_TOOL_PROFILE`, read/bash/edit/write) | `add-feature` median 72.469 s vs 191.077 s (full), "62.1%" lower; schema 21,188 to 10,038 bytes (-52.6%) for the product profile. `fix-failing-test` 18.482 vs 19.089 s. | `2026-09-17-pi-tool-profile.md`; `2026-09-17-pi-profile-add-feature-repeat.json` | n=2 per profile (66.3/78.6 s vs 122.0/260.2 s); `fix-failing-test` n=2; taskflow n=1 | Directional only, and the doc says so. Not on by default. Taskflow did not confirm it (see section 5). |
| 8 | Memory retention through a fresh session | Missing-rule resume: memory off 0/4 correct vs on 4/4 (desktop, 32-arm study, temperature 0). Tool-free diagnostic: off `UNKNOWN` vs on `FLOOR`, 2 of 2 pairs. | `memory-desktop/results.md`; `memory-pilot/results.md` | 4 pairs desktop plus 2 pairs diagnostic; repeated identically in the 2nd 32-arm round (0/4 vs 4/4) | Solid for the narrow claim (a fact absent from the prompt is delivered). Docs explicitly do not claim better reasoning or speed. |
| 9 | Memory dedup (drop lessons already verbatim in the transcript) | Post-compaction mean paired overhead +4.945 s (+18.6%) to +1.636 s (+6.6%); mean extra input tokens +15,673 (+63.3%) to -121 (-0.5%); injected tokens 144-152 to 0. | `memory-desktop/dedup-remeasurement.md`, `dedup-fix.md` | 4 pairs before, 4 pairs after, synthetic fixtures. The token effect is mechanical (deterministic replay in `dedup-replay.json`); the elapsed effect is noisy (the doc notes separate rounds have different cache state). | Token result strong; elapsed result weak. |
| 10 | `memory_correct` bounded correction | Stale-lesson case from 149.53 s / 36 requests / 35 tool calls / 363,534 cumulative input tokens to 21.497 and 26.533 s with 6 requests each (enabled arms) | `memory-pilot/results.md`, `memory-pilot/correction-results.md` | 1 bad run before; 4 runs after (not a controlled comparison: the tool set differed, doc says "a failure example, not a controlled timing baseline") | Detour gone in 2 of 2 enabled runs; no timing claim can be made. |
| 11 | Desktop browser lane fixes | 107 s, 2 failed calls, 0-byte screenshot, to 33 s, 4 calls, 0 failed, 33,666-byte PNG | `2026-09-16-harness-comparison.md` (browser sections); `2026-09-17-zero-desktop-browser-smoke.json`, `...-rebuilt-live.json` | 1 run before, 1 run after | One clean run. Later commit `e6a6e2513b` (COMMIT-ONLY) found more (input to hidden tabs silently dropped, blank tab waiting the 30 s bridge timeout). |
| 12 | Minecraft MCP lane | 5 attempts, 3 runtime passes; latest world join, block apply, NBT inspect, emitter query, 88,480-byte 1708x960 PNG in 57 s | `2026-09-17-zero-desktop-minecraft.json`, scorecard | 1 pass per configuration, defects fixed between attempts | Works once end to end after fixes. |
| 13 | Interrupt and resume | First phase 8.1 s, resumed phase 12.0 s, `paginate.js` fixed, verifier 7/7 | `2026-09-17-zero-recovery.json` | 1 run | One pass. Zero-only, no comparison. |
| 14 | Project-root fix (`PWD` no longer consulted) | First OpenBench cell scored 0 because the agent edited the source task definition under `vendor/openbench/tasks/`; regression test "confirmed to fail without the fix" | `docs/handover/2026-09-19-codex-handover.md` section 3 | 1 failure observed, test-verified fix | Fixed and tested. |
| 15 | Title generation without thinking (`c52fd56cab`) | Title request "held a first prompt back by about 50 seconds and slowed its prefill from ~310 to 196 tok/s"; after: title finishes in 3 tokens | commit body only | COMMIT-ONLY, no eval doc | Not verifiable from the eval corpus; see contradiction with the 2026-09-16 title experiment in section 5. |
| 16 | Engine timings in the ledger (`f1acf655b9`) | Made the wall-time breakdown in section 3 possible | handover section 1 | n/a | Enabling work, not a speedup. |

### Tried and measured not to help (all single trials unless stated)

| Idea | Result | Source |
|---|---|---|
| `experimental.stable_prefix` | Both arms 3,142 input tokens, 60,388 cached, 95.1% hit; TTFT avg 3.18 s (on) vs 2.66 s (off) | `2026-09-17-zero-stable-prefix-ab.json`, harness-comparison |
| Compact system prompt (9.6 KB to 2.1 KB) | `make-it-run` 30.225 s and 27.133 s vs 21.076 s control; add-feature 199.333 s | `2026-09-17-pi-tool-profile.md` |
| Strip prior reasoning from replay | taskflow 465.144 s vs 396.103 s matched full control | `2026-09-17-pi-tool-profile.md` |
| In-memory converted-context cache | taskflow 418.116 s vs 369.732 s control; add-feature 279.433 s; `make-it-run` 23.137 vs 21.952 s | `2026-09-17-pi-tool-profile.md`, `2026-09-17-pi-derived-execution-plan.md` |
| Native LLM transport | taskflow 346.809 s vs 291.289 s (AI-SDK, minimal) and 396.103 s (full); `fix-failing-test` 26.411 s | `2026-09-17-pi-tool-profile.md` |
| Repo map (long-horizon) | inventory timeout, live-bugs 45 min with 3 compactions vs 20 min; fast only on qa-spec (19 min vs 32) | `2026-09-23-long-horizon-results.md` |
| Automatic title generation as latency source (2026-09-16 test) | 16.28 s vs 12.86 s; "no title-related code change is justified" | `2026-09-16-harness-comparison.md` |
| Background shell | "worked when used" (live-bugs repo-map arm: `shell_output` 2, `shell_kill` 1) but "no run showed a time saving" | `2026-09-23-long-horizon-results.md` |

---

## 2. Measured failure modes

Status key: FIXED (commit cited if a doc or commit names one), OPEN, UNCLEAR.

### Context, compaction and long-session behaviour

| # | Failure | Evidence | Status |
|---|---|---|---|
| F1 | Read, compact, "no code written yet", re-read, refill loop on a small window | Baseline inventory: 100 reads (75 repeats), 9 compactions, timeout at 90 min "still on its first request" (turns=1); prose in commit `ed3c6537ff`. `lh-baseline-inventory.md`: 130 tool calls, 5401 s | Mostly fixed by `ed3c6537ff` (warn from 70% of budget, discourage re-reads). Not eliminated: `duplicate_reads` 29 (after rerun), 17 (tool search rerun), 20 (after), in the JSONs. |
| F2 | Compaction is slow | Baseline 136 to 252 s each, ~27 of 90 min; after: 7 to 21 min per inventory run (16 to 24% of wall, INFERRED from table) | Reduced by `c1049454ec`; OPEN as a cost. |
| F3 | A message sent to a full session is folded into the summary and lost | Recall probe answered with "no next steps remain" instead of the user's question; scored 0/12 and 1/12 on the two pre-fix inventory "after" reruns although the replies recalled 12/12 | FIXED `632e803b5d`. |
| F4 | A finished answer is followed by an unrequested extra turn (re-run tests, status report) when the reply left the window over the limit | Same runs | FIXED `7a25304f26`. |
| F5 | Recall probe scored on the wrong message | Eval defect | FIXED `5a7e81ebc5`. |
| F6 | Early user instructions are forgotten across compaction | Baseline recall 9/11 (missed `real-server-child-process`, `port-env`) and 8/10 (missed `test-per-fix`, `report-line-format`) | Carry-forward added in `8b3c9b6f1a`; after runs recalled all. But see section 5 (baseline recall may be partly the F3 bug). |
| F7 | Long task hits the 90-minute cap | Inventory: baseline timeout, repo map timeout at 90 min (11 compactions), after rerun fail at 88 min; 3 of the 6 inventory runs passed | OPEN. This is the failure most relevant to unattended runs. |
| F8 | Hidden-check failure on a task the model believed done | after rerun fail on 2 checks (reservation expiry, "nothing in between that touches the failing feature"); baseline qa-spec missed a hidden sort check | UNCLEAR (variance vs capability; cause not isolated). |
| F9 | Auto-compaction never fires when the provider reports no usage | Commit `487290fd93` (COMMIT-ONLY) | FIXED `487290fd93` (estimate fallback). |
| F10 | Five-minute turn, tool, subagent and compaction timeouts killed healthy long turns and respawned subagents | Commit `83c050a82a` (COMMIT-ONLY); handover: a 27,394-token cold prompt "measured 98 seconds of prefill", the five-minute stream timeout was uncommitted and excluded from the baseline | FIXED `83c050a82a`, `5634d693e6` (30 min defaults for loopback providers: "SSE read timed out"). |
| F11 | Provider retry re-ran tool calls (second set of subagents) | Commit `6ce4f7b644` (COMMIT-ONLY) | FIXED `6ce4f7b644`. |
| F12 | Replayed a large user prompt after every compaction, refilling the window | Commit `83c050a82a` (COMMIT-ONLY) | FIXED `83c050a82a`. |

### Speed, engine and cache

| # | Failure | Evidence | Status |
|---|---|---|---|
| F13 | Cold prefill dominates first turns and large prompts | 8,871 tokens/12 tools 33.5 s cold vs 3.5 s warm (handover); 7,833 tokens 30.8 s (`2026-09-14`); 31,735 tokens 78.5 s with 4,096 cached (`2026-09-16-zero-speed-controlled.md`); "ctx-64k" row is 90,390 to 90,505 prompt tokens: TTFT 324 s, 309 s, 294 s, 235 s across four runs; ctx-16k 22.6k tokens: 61 s, 56 s, 68 s, 97 s. Prefill 233 to 386 tok/s | OPEN (engine-bound). Mitigated for tools by tool search. |
| F14 | Cache does not carry across projects | 2026-09-14: 99.6% same-project vs 0 of 7,833 cross-project (TTFT 20,765 ms) | UNCLEAR: 2026-09-16 and 09-17 show 28,672 of 31,735 cached cross-project. See section 5. |
| F15 | A changed MCP description costs a large re-prefill | "one changed description cost a 92 s re-prefill at 32k" (cited in `2026-09-17-pi-derived-execution-plan.md`, source measurement of 2026-09-15 is not in `evals/`) | Mitigated by tool search keeping the declared set constant (`43e1546883`). |
| F16 | Title request contends with the first prompt on a single-request engine | COMMIT-ONLY `c52fd56cab`: ~50 s hold-up, 310 to 196 tok/s | FIXED `c52fd56cab` (per commit). |
| F17 | Engine keeps working after the client gave up | 91,909-token prompt: "spent about two minutes persisting the prefix; the request became idle only after the client had stopped receiving a response" (Pi client, `2026-09-17-scorecard.md` / `2026-09-17-context-64k-probe.json`) | OPEN (engine side); observed once with a different harness. |
| F18 | Headless run stalls at bootstrap and never creates a session | COMMIT-ONLY `9e3741bfb0`: "intermittently stopped after the formatter's init line ... no child processes or sockets" | UNCLEAR: only diagnostics (phase timing, 10 s/30 s/2 min watchdog) were added; root cause not stated. |
| F19 | A stalled request looks the same as a harness that never called the model | handover section 4: "cost ten minutes to work out once" | Mitigated in the study proxy (`tools/meter.ts` logs start and completion), not in Zero. |

### Tool use, correctness and termination

| # | Failure | Evidence | Status |
|---|---|---|---|
| F20 | Agent keeps running after the task is solved | `webcore`: checker 43/43 but `completed=false` at 900.024 s (trial 1); trial 2 also 900.024 s with 16/24 feature checks; adapter completion guard did not fix it. Commit `a4d4e67ee7` (COMMIT-ONLY) calls "recognizes a finished task" "the one repeated failure in the benchmarks" | OPEN. Executive observer is advisory and unmeasured. |
| F21 | Verification claim does not match the output | `py-wordfreq-cli` final text claimed "cat 4" vs real output "cat 3"; that was the one coding failure, with 0 of 2 edits preceded by a read (`2026-09-14-qwen3.8-flash.md`) | OPEN (n=1). |
| F22 | Passing visible tests, failing hidden edge cases | Desktop `add-feature`: hidden score 0.9000 after 2m 27s (missing include raised `FileNotFoundError` not `ConfigError`), fixed in 27 s after feedback; `long-kv-cli`: verifier passed but `__proto__` key persistence and symlink invocation were broken | OPEN; each n=1. |
| F23 | Tool-call repair did not run | `LLMRepair.deferredSchemas` had no caller | FIXED in `f1acf655b9` (handover). |
| F24 | Slow trajectories: many turns and rereads for the same fix | Taskflow, coding-minimal/medium: 16 main requests, max prompt 54,681 tokens, 16 thinking segments, "repeated read/investigation passes before the same seven fixes" (369.732 s) vs Pi median 165.625 s | OPEN. |
| F25 | Agent writes outside its workspace / touches the wrong tree | PWD bug (agent edited `vendor/openbench/tasks/`); memory-off arms searched parent directories and user config (twice); model created four empty SQLite files under `~/.config/zero` while probing (`memory-desktop/unscored-discovery.md`) | PWD FIXED (`f1acf655b9`). Model behaviour with unrestricted bash: OPEN by design (handover: "the harness must not refuse"). |
| F26 | Workspace pollution | `telemetry.db-wal`, `__pycache__` recorded as changed files | OPEN (cosmetic). |
| F26b | Two Zero `make-it-run` cells ended in ~1.6 s with `success=False`, `completed=False`, `error: exit 1`, empty checker output | `runtime-runs/2026-09-17-optimization/zero-proxy-trace.jsonl` (1.58 s, 1.559 s); no .md mentions them | UNCLEAR (INFERRED: instrumentation/proxy failure from the run name; cause not documented). |
| F27 | Guard refusals / tool-call errors are rare | Guard refusals 0 in every run listed; failed tool calls 0 to 4 per long-horizon run (e.g. 4 of 162) | Not a problem in this data. |

### Desktop, browser, computer use, UI

| # | Failure | Evidence | Status |
|---|---|---|---|
| F28 | Desktop timeline shows only "Thinking" during a long run; Context panel lists finished tools | `2026-09-15-desktop-development-loop.md`; the same user message ID listed twice | UNCLEAR: later status smoke (9 s request) showed timer and Stop control but not a long run. |
| F29 | Browser: navigate/inspect failed on a fresh tab; zero-byte screenshot | 107 s, 2 failed of 6 calls, `screenshot_bytes` 0 | FIXED (source probe 24,187-byte PNG; rebuilt live 33 s, 33,666 bytes). |
| F30 | Agent browser input reported success but did nothing on tabs the panel was not showing; blank tab waited out the 30 s bridge timeout | COMMIT-ONLY `e6a6e2513b` | FIXED `e6a6e2513b`. |
| F31 | Computer-use desktop control loops on stale observations | Minecraft postmortem: only two screenshots 9 min apart, 18 shell calls, 7 computer calls, reused old titlebar coordinate after moving the window, `focus_app(java)` receipt proved nothing; run aborted | UNCLEAR: the postmortem lists "required correction"; no later doc shows it implemented. The MCP route (`mc.*` tools) succeeded instead. |
| F32 | Minecraft MCP: stale endpoint (`fetch failed`), nullable-screen NPE, `flat=true` ignored, generic object schemas making the model guess arguments | `2026-09-17-zero-desktop-minecraft.json` | FIXED (schemas, normalization, endpoint probe); 1 pass. |
| F33 | Packaged-app defects invisible to Bun/source tests | `Bun is not defined` in Node sidecar map scanner; global vs project memory store aliasing; undefined browser `tab` metadata making the permission list return 400 | FIXED (`memory-desktop/desktop-verification.md`); the pattern (CLI or source green, packaged app broken) recurs. |
| F34 | Desktop build from another branch hid the user's history (db named by branch) | COMMIT-ONLY `0693f17257` | FIXED. |

### Memory feature (side effects)

| # | Failure | Evidence | Status |
|---|---|---|---|
| F35 | Stale memory note triggers an unrequested maintenance detour | `obsolete-2-on`: 149.53 s, 36 requests, 35 tool calls, 363,534 cumulative input tokens, 93.01 s after the passing test | Mitigated by `memory_correct`; 2 of 2 enabled reruns clean. |
| F36 | Redundant memory injection after compaction | +18.6% time, +63.3% input tokens | FIXED (dedup, see win 9); residual +6.6%. |
| F37 | Advertising the correction tool costs something even when memory is off | One disabled arm invented an old lesson to call it | OPEN, minor. |
| F38 | Obsolete zero-based note stays in the frozen system prompt after the on-disk fix | `memory-pilot/results.md` | By design (frozen prefix); handled by explicit superseding result. |

---

## 3. Where the time goes

### 3.1 Per-request and per-turn figures

| Figure | Value | Source | Base |
|---|---|---|---|
| Share of in-request time spent in the engine | 88% | handover section 4 | 1 run (`misleading-error`, 7 turns) |
| Gap between deciding to call the model and first token | 40.7 s TTFT vs 16.6 s engine prefill across 7 turns, "roughly 3.4 s per turn"; handover calls it "the most promising thing" for feeling faster | handover section 4 | 1 run. The 3.4 s is INFERRED-safe arithmetic ((40.7-16.6)/7) but includes queueing and possibly cold-start effects. Later commits added request-prep, history-load and context-assembly timing events (`2026-09-17-pi-tool-profile.md`), but no eval doc publishes numbers from them. |
| Prefill throughput | 233 to 386 tok/s in direct probes (22.6k to 90.5k tokens); 369 vs 279 tok/s in the first run | `2026-09-14`, `2026-09-16-zero-speed-controlled.md`, `2026-09-17-speed-repeat.md` | 4 runs; wide spread |
| Decode throughput | short prompts 22.0, 29.3, 51.6, 60.0, 63.9 tok/s; 22.6k-90.5k contexts 34.3 to 57.3 tok/s; long-horizon means 30.4 to 53.2 tok/s | same plus `long-horizon-2026-09-23/*.json` | many; note the long-context probe completions are 23 to 26 tokens |
| Cold first turn of Zero CLI | 7,833 tokens, 30.8 s; 8,871 tokens, 33.5 s; desktop profile 31,735 tokens, 78.5 s | `2026-09-14`, handover, `2026-09-16-zero-speed-controlled.md` | 1 run each |
| Warm first turn | 1.3 to 1.5 s, 99.6 to 99.9% prefix hit | same | 3 runs |
| Prefix hit in long-horizon runs | 0.78 to 0.91 | JSONs | 14 runs |

### 3.2 Long-horizon wall time (INFERRED from per-run JSON)

I computed TTFT time as `requests x ttft_avg_ms` and decode time as `output / tokens_per_second` from `long-horizon-2026-09-23/lh-*.json`. Caveats: `requests` may include compaction summary calls; the definition of `tokens_per_second` (per request mean vs total) is not documented; the two terms may overlap slightly. Treat as a rough split.

| Run | Wall | TTFT share | Decode share | Sum |
|---|---:|---:|---:|---:|
| baseline inventory (timeout) | 5401 s | 39% | 46% | 85% |
| after inventory (pass) | 4362 s | 32% | 51% | 83% |
| after inventory rerun (fail) | 5284 s | 33% | 61% | 94% |
| tool search inventory | 3872 s | 31% | 59% | 91% |
| tool search inventory rerun | 2906 s | 32% | 61% | 93% |
| repo map inventory (timeout) | 5402 s | 32% | 61% | 93% |
| baseline live-bugs | 2815 s | 22% | 46% | 69% (desktop builds were running) |
| after live-bugs | 1202 s | 35% | 51% | 86% |
| after qa-spec | 1942 s | 32% | 60% | 93% |
| tool search qa-spec | 862 s | 35% | 55% | 90% |

INFERRED reading: 80 to 95% of wall time is model time (roughly a third waiting for first token, roughly half to 60% generating), leaving about 5 to 20% for tools, tests, dev servers and harness. That agrees in direction with the handover's 88% engine share but the two are different measurements. Output volume matters: inventory runs generated 67,874 to 130,956 output tokens.

Compaction as a share of wall (from the results table): baseline inventory ~27 of 90 min (30%); after 12 of 73 (16%), 21 of 88 (24%); tool search 9 of 65 (14%), 7 of 48 (15%); repo map 20 of 90 (22%). Live-bugs and qa-spec: 1 to 4 min.

### 3.3 Trajectory, not per-token cost, explains the Pi gap

`2026-09-17-pi-tool-profile.md` (taskflow, rebuilt binary, coding-minimal, `medium`): 16 main requests, 54,681 maximum prompt, 5,980 streamed output tokens, 31 tool messages, 86.19% mean prefix-cache reuse (355,006 reused tokens), 369.732 s. The doc concludes "A context cache alone cannot be credited with a speed win" and points at "unnecessary model/tool turns and redundant repository rereads". One trial.

Other measured components: tool schema 21,188 bytes (full) vs 12,673 (Pi-style) vs 10,038 (product `coding-minimal`); system prompt about 9.6 KB (compact variant 2.1 KB); the user's 64 MCP tools were "about 25k tokens of schema, so roughly half of the usable window" at 64k (`2026-09-23-long-horizon-results.md`).

---

## 4. Comparisons against other harnesses

Model, engine and endpoint identical. Harnesses: Pi 0.73.1, OpenCode 1.18.31, Qwen Code 0.24.0, Aider 0.86.2, Zero `0.0.0-main-202609162231`. OpenBench pinned to `db193457a3d9128cd4d01fd839c2a890c186c9ac`. All cells: sequential, shared warm KV cache (the docs' own caveat), token telemetry not exposed by most harnesses. Source for tables: `2026-09-16-harness-comparison.md` and the JSONs it cites.

| Task | n per harness | Aider | Pi | Zero | Qwen Code | OpenCode |
|---|---|---|---|---|---|---|
| `fix-failing-test` (median wall) | 3 each | 8.371 s | 24.68 s | 27.824 s | 33.069 s | 34.507 s |
| `make-it-run` | 1 each | 11.6 s | 13.6 s | 17.5 s | 16.5 s | 21.5 s |
| `add-feature` | 1 each; Zero also n=2 (105.4, 125.1 s) | 20.57 s | 71.53 s | 73.16 s | 300.95 s, timed out (checker passed) | 114.82 s |
| `taskflow` first cell | 1 each | 180.13 s, score 0.28 (7/25) | 197.39 s | 169.47 s | 171.70 s | 225.23 s |
| `taskflow` repeat | 2 each | 5.771 s, 2.462 s; 0.28 both (adapter fault: "0 files", see section 5 item 12) | 171.081, 160.169 (median 165.625) | 298.788, 168.781 (median 233.785) | 145.861, 158.03 (INFERRED median 151.95) | 186.391, 187.11 (INFERRED median 186.75) |
| `webcore` | Zero only, n=2 | not run | not run | 900.024 s both, `completed=false`; checker 24/24 then 16/24 | not run | not run |
| Speed probe "Reply exactly OK", median of 3 | 3 each | 3.761 s | 1.933 s | 3.225 s (desktop app) | 3.959 s | 8.048 s |
| 16k-word context probe | 3 each; Zero not run | 3.385 s | 42.402 s | n/a | 42.591 s | 5.697 s |
| 64k context probe | 1 each | ~131 s | 184 s | 236 s | ~188 s | 177 s |

Aggregates the docs publish:

- Coding composite (`2026-09-17-scorecard.md`): Pi 0.926, Zero 0.872, OpenCode 0.864, Aider 0.832, Qwen Code 0.713. Median wall over four task cells: Pi 48.1 s, Zero 71.5 s, OpenCode 74.7 s, Aider 16.1 s, Qwen Code 102.4 s. Composite weights: 45% correctness, 35% clean completion, 20% min-max speed. Strict pass 4/4 for Pi, Zero, OpenCode; 3/4 for Aider and Qwen Code. Sample size behind it: one trial per cell except `fix-failing-test` (3) and Zero's `add-feature` (2).
- The earlier one-trial leaderboard (`2026-09-16-openbench-leaderboard.json`) had medians Pi 42.578 s, Zero 45.352 s, OpenCode 68.163 s. The repeat-aware version moved Zero to 71.543 s. The Zero-Pi gap grew from about 2.8 s to about 23 s once Zero was repeated (INFERRED comparison of the two files).
- All-around matrix (`2026-09-17-all-around-leaderboard.md`): `coverage_incomplete`; context, tool use and recovery missing for every candidate; composite deliberately unset. Zero-only rows: browser 42.134 s/5 calls, long-running goal 300.756 s, recovery, desktop status, Minecraft.
- Speed score (`2026-09-17-native-speed-probe.json`): Pi 1.0, Zero 0.7887, Aider 0.7011, Qwen Code 0.6687, OpenCode 0.0, from n=3 wall medians. Zero was measured through the packaged desktop UI with 35,018 tokens cache-read; others through CLI process wall time. Not like for like, as the doc states.
- Zero vs its own profiles vs Pi (taskflow, single runs): coding-minimal 291.289 s, full 396.103 s, Pi median 165.625 s.
- Only Zero has been run on: `webcore`, browser tier, recovery, long-horizon, memory, Minecraft. No other harness has any long-horizon or compaction-crossing result in this corpus.

Sample-size honesty: the docs mark the standings "provisional" and say "single-cell timing should not be used as a stable speed ranking". No confidence intervals anywhere.

---

## 5. Contradictions and weak evidence

**Regime mismatches**

1. **Long-horizon runs say nothing about a 262k window.** Every run used `--context 65536` (about 57k usable) with 64 MCP tools taking about 25k tokens. Compaction frequency, per-compaction cost, tool-search value, and re-read pressure are all functions of that cap. Direct probes show prefill at 233 to 386 tok/s, so compaction or re-prefill of a 100k+ context is a different cost regime (INFERRED: 162k tokens at ~300 tok/s would be about 9 minutes cold). Nothing in the corpus measures behaviour between ~57k and 262k, or memory pressure on the engine. The commit `43e1546883` notes the auto threshold is "44% of a 64k window and 11% of a 262k one for this catalog", i.e. the tool-search benefit was measured where it matters most.
2. **Tool search benefit is measured only where MCP is never used.** Both tool-search A/B (`2026-09-17-tool-search-ab.md`) and long-horizon tool-search runs have no `tool_search`/`tool_exec` calls (the doc: "the treatment arm never called `tool_search`"; the long-horizon JSON `by_tool` maps contain none). The cost of searching, discovery failures, and browser capability when it is needed are unmeasured. On qa-spec the baseline used 39 Playwright MCP calls; the "after" and "tool search" arms used none, and the repo-map arm used 5. The doc says the runs "say nothing about browser skill either way". INFERRED: the unused schema could have been removed by not connecting those servers at all.
3. **Small OpenBench tasks (5 to 300 s) exercise neither compaction nor large context**, yet the leaderboard is presented as a harness ranking. The speed rankings are dominated by process start and cache state (e.g. 16k probe: OpenCode 5.697 s and Aider 3.385 s vs Pi 42.402 s and Qwen Code 42.591 s for the same prompt; Aider reports "19k sent" of a ~24k-token prompt).

**Disagreements between docs**

4. **Tool search: "stays opt-in" (2026-09-17) vs automatic (2026-09-23).** The A/B doc applies a promotion rule ("single-task medians are single-task evidence") and found wall time "indistinguishable"; commit `43e1546883` turned it on automatically from 1 to 2 runs per task with the largest unexplained variance in the study. The promotion rule was not applied to the later decision.
5. **Title generation.** `2026-09-16-harness-comparison.md`: 16.28 s vs 12.86 s, "hypothesis ... unconfirmed; no title-related code change is justified". Commit `c52fd56cab` later claims a ~50 s hold-up of the first prompt. Different regimes (tiny task vs first prompt of a real session, thinking on) could reconcile them, but no eval doc backs the commit.
6. **"Cold" cache rows are not cold.** `2026-09-14`: harness-cold 0 of 7,833 cached, 30.8 s. `2026-09-16`: harness-cold 4,096 of 31,735, 78.5 s. `2026-09-17-speed-repeat.md`: harness-cold 28,672 of 31,735 cached, 9.2 s. The "cold" and "cold-other-project" labels are unreliable, and the 09-14 claim that "the prefix cache does not carry across projects" contradicts 28,672/31,735 cross-project hits on 09-16 and 09-17. INFERRED: the difference is the presence of a project-specific string early in the prompt, or engine cache retention.
7. **Decode speed and context.** Handover: decode "about 23 tok/s at 10k context against 47.9 tok/s at short context. If that holds, context economy buys speed twice." The direct probes show 34.3 to 57.3 tok/s at 22.6k to 90.5k tokens, and short-prompt decode ranges 22.0 to 63.9 tok/s across runs. The claim is not supported by the speed probes (which use 23 to 26 token completions, a coarse basis).
8. **Prefill rate for the same engine varies about 2x.** Zero direct: 22.6k tokens in 56 to 97 s; Pi: 24.2k tokens with cache_read 0 in 42.4 s (INFERRED 570 tok/s). At 64k: Zero 235 to 324 s for 90.4k tokens; Pi 184 s for 91.9k. Not explained in any doc; 64k row cache states differ (0, 20,480, 28,672, 61,440 cached tokens) and the Pi retry followed a terminated attempt that had already made the engine prefill the same prompt.
9. **Taskflow "Pi is faster".** `2026-09-17-pi-derived-execution-plan.md` cites Pi 165.625 s vs Zero 233.785 s. That Zero figure is the median of the two repeat trials only (298.788, 168.781); the first Zero cell (169.469 s) is dropped. Including it, Zero's median is 169.469 s and Pi's (197.388, 171.081, 160.169) is 171.081 s (INFERRED). Trial 2 of Zero (168.781 s) equals Pi's. In the same repeat Qwen Code (median INFERRED 151.95 s) and OpenCode (INFERRED 186.75 s) also beat Zero's 233.785 s. The later rebuilt-binary Zero taskflow runs (291 to 465 s) are slower than every early-binary Zero sample (169 to 299 s), which may reflect build drift (INFERRED); no doc compares builds.
10. **Add-feature profile A/B and build drift.** `coding-minimal` 72.469 s vs `full` 191.077 s (n=2 each; full includes a 260.158 s outlier). Earlier full-profile Zero on the older binary ran 73.16, 105.4 and 125.1 s. So the "62.1%" gap depends on which "full" samples one trusts. The taskflow follow-up did not reproduce a Pi-level result either.
11. **`Scorecard` says Zero 100% clean completion**, but Zero's own `webcore` cells (2 of 2 never terminated) are excluded because "only Zero Code has been run". The exclusion is defensible for ranking but flatters the completion column.
12. **Aider on taskflow.** Repeat wall times of 5.771 s and 2.462 s cannot contain the ~180 s of work of the first cell, yet both scored 0.28. I read the trial-2 transcript (`runtime-runs/2026-09-17-taskflow/aider-qwen-transcripts/aider-qwen/aider-qwen_02e8e9ddb9f6_taskflow_qwen3.8-local_trial2.txt`): "Git repo: .git with 0 files", 948 tokens sent, the model emitted a text `<tool_call>` for `list_files` and stopped after 38 tokens. So the repeat "reproduced 0.28 on 2/2" is an adapter/file-admission fault (Aider was given no files), not evidence about Aider's ability; the first cell's failure had a different cause (`cd /app` after files were injected). Aider's taskflow score should be excluded from any ranking (this is read, not inferred).
13. **Recall scoring on the baseline.** The doc says every run was re-scored from its database, then says "the other two baseline runs have no separate record" (baseline predates per-run isolation). INFERRED: the baseline live-bugs 9/11 and qa-spec 8/10 may come from the old scorer, and the baseline ran before fix `632e803b5d`, so the probe message may have been lost rather than the requirements forgotten.

**Weak evidence**

14. Long-horizon: one trial per arm per task, no repeat except two inventory reruns. The identical "after" arm passed inventory in 73 min and failed it in 88. Tool-search vs after on wall time is inside that noise.
15. Baseline live-bugs wall (47 min vs 20 min) is confounded by desktop app builds running; the results doc flags it.
16. Every model-behaviour claim in the memory studies rests on 4 synthetic fixtures per scenario with seeded memories and one run per condition; the docs say so. Timings in memory pairs change sign between rounds. Baseline runs for the same scenario cite 26.519 vs 24.883 s for the same off arm.
17. `coding-minimal` results are n=2 and disappear on the one long task (taskflow n=1).
18. Handover "88% engine" and "3.4 s per turn" rest on one 7-turn run.
19. Desktop lane results (browser 33 s, Minecraft 57 s, recovery, add-feature 27 s correction) are single runs. "Clean live desktop result" means one clean run.
20. Many numbers exist only in commit messages (COMMIT-ONLY): 50 s title hold-up, 32.6k to 11.8k token first prompt, 5-minute timeout failure, SSE timeout, bootstrap stall, browser input dropped. They cannot be checked from the eval corpus.
21. The `main` tree did not typecheck at the time of the 2026-09-19 handover and 340 files were uncommitted; the corpus mixes results from `main` dirty tree (`d13770b1c71d`), packaged builds `202609162231` to `202609170926`, `a03b629b4e4a` and `8c1fb77e1666` dirty trees. Builds are not held constant between the comparisons in section 4 and the long-horizon runs.

---

## 6. Open problems ranked

Ranking is by likely impact on a user running long unattended tasks. Evidence is quoted where it exists; rank 1 and 2 are the ones the corpus supports best.

1. **Context pressure and compaction cost inside a long session.** Evidence: baseline inventory spent ~27 of 90 min in 9 compactions and timed out; after-family inventory runs still spend 7 to 21 min in 3 to 11 compactions; 3 of 6 inventory runs did not pass (two timeouts at 90 min, one fail at 88 min); duplicate reads persist (17 to 29 in three of five later inventory runs). Unmeasured: behaviour at the real window (262k), where per-compaction and cold-restart prefill costs are larger (INFERRED). This ranks first because it caps how long an unattended task can run.
2. **Run-to-run variance and hidden-check failures (false "done").** Evidence: same arm pass 73 min vs fail 88 min; qa-spec baseline missed one hidden check; desktop add-feature scored 0.9000 until fed the failure; `long-kv-cli` shipped with `__proto__` broken; claimed verification not matching output. Unattended runs have no user to feed the checker output back. No mitigation is measured.
3. **No progress or completion detection (non-termination, over-work).** Evidence: `webcore` 900 s twice with checker 43/43 on one; a finished answer followed by unrequested work (fixed only for the compaction case); stale-observation loops in computer-use (18 shell + 7 computer calls, two screenshots 9 minutes apart); the executive that would detect this is advisory and has produced no measurement yet. Cost is wasted hours, and possibly harmful actions with unrestricted bash.
4. **Engine and transport stalls that end a session.** Evidence: request timeouts (5 min, SSE) fixed but only by raising to 30 min; engine "persisting the prefix" for about two minutes after a 91,909-token prompt; intermittent bootstrap stall with unknown cause; a stalled request indistinguishable from an idle harness. The just-died 162k session is a known instance and is out of scope here.
5. **Cold prefill and cache invalidation on every restart.** Evidence: 30.8 s (7.8k), 33.5 s (8.9k), 78.5 s (31.7k), 235 to 324 s (90k). A restarted session or engine re-pays this; "one changed description cost a 92 s re-prefill at 32k". Tool search shrinks the fixed prompt (32.6k to 11.8k COMMIT-ONLY) but a 100k+ history re-prefill is unmitigated.
6. **Inefficient trajectories relative to Pi on multi-step tasks.** Evidence: taskflow 16 requests, 31 tool messages, repeated rereads (369.732 s); Pi median 165.625 s (n=2, with caveats above). Every candidate mitigation tried so far (stable prefix, compact prompt, strip reasoning, in-memory context, native transport) showed no gain on single trials.
7. **Desktop tool reliability under load.** Evidence: browser input silently dropped on hidden tabs (fixed), packaged-only defects (Bun in Node sidecar, 400 on permission list), Minecraft computer-use failure. These were found by driving the packaged app, not by CLI or source tests. Live timeline "Thinking" stall not re-verified on a long run.
8. **Memory feature overhead.** Evidence: +6.6% residual post-compaction, +3.863 s and +13,617 input tokens in the obsolete-lesson scenario, tool advertised even when disabled. Small compared with 1 to 7.
9. **Measurement gaps that limit future decisions.** No other harness on any long-horizon task; no repeated long-horizon arm; no near-limit or 262k probe; unmeasured search cost for tool search; harness overhead per turn (3.4 s) not re-measured with the new timing events; builds not held constant across comparisons.

---

## Appendix: which commit came from which measurement

| Finding | Commit |
|---|---|
| Read, compact, re-read loop (baseline inventory) | `ed3c6537ff` |
| Compaction 2 to 4 min, mostly thinking | `c1049454ec` |
| Lost message on a full session (recall probe) | `632e803b5d` |
| Answer followed by unrequested work | `7a25304f26` |
| Recall probe scored on wrong message | `5a7e81ebc5` |
| MCP schema half of the 64k window | `43e1546883`, `2241ea5edb` |
| Early user instructions lost in summary | `8b3c9b6f1a` |
| Cancelled turn seen only as bare tool errors | `08fcc7403b` |
| Dev servers cannot be kept alive for live testing | `fc498fd9a6` |
| Benchmark sessions polluting the user's history | `d2ba4c8ee2` |
| Browser input to hidden tabs (packaged-app driving) | `e6a6e2513b`, `fa33ffa511` |
| Title request contention | `c52fd56cab` |
| Five-minute timeouts, SSE timeout, retry re-running tools | `83c050a82a`, `5634d693e6`, `6ce4f7b644` |
| Bootstrap stall (diagnostics only) | `9e3741bfb0` |
| No-usage providers never compact; one context budget | `487290fd93` |
| Engine timings stored per message / shown in UI | `dcdf32884f`, `d24a8de044` |
