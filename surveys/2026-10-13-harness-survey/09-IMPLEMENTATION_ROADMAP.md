# 09 — Implementation roadmap

Phased, dependency-ordered plan to evolve Zero Code per 08. Priority metric everywhere: **successful autonomous
coding per unit of inference, time, compute** — not feature count. "Eng" = conventional engineering; "Res" =
requires running the 06 experiments first. Every phase leaves Zero Code functional; every phase names its
regression risk and its measurable exit criterion. Effort classes: S ≤ few days, M ≈ 1–2 eng-weeks, L ≈ a month+.
File paths point at the audited code (01).

**Quick wins (independent, start immediately, no phase dependencies — all Eng, all ≤S):**
- W1 fix the eval-boot `session.create` hang (`06 §R3`, repro in `2026-10-13-context-stress.json`) — unblocks all
  eval-driven gating. *Success: long-kv-cli boots and runs end-to-end.*
- W2 surface the compaction archive path on *every* compaction, not only overflow (compaction.ts:668,783; 01 Part C
  risk 1). *Success: new test asserts archive pointer present post-compaction; recall probe improves or ties.*
- W3 literal `"(no output)"` + reclaimable-bytes accounting parity in the truncate path; pagination (or explicit
  `continue` token) for grep's hard 100 cap (grep.ts:71-113). *Success: existing tool tests + 1 new each.*
- W4 wire the telemetry ledger into one feedback loop: log `prefix_broken` count in the loop's structured log and
  expose it in eval reports (already computed, unread — Part C §6). *Success: eval report gains a column.*
- W5 FTS5 index, ranking and jump-to-message over the message-text search that already ships (LIKE +
  json_extract, `session/session.ts:203-213`, wired to home/app) — plus `@conversation:`-style citation of child
  transcripts (correction pass C3-P4: this is an upgrade, not a greenfield feature).
- W6 adopt aider's refresh-cost lesson in the flag-off repomap: change-triggered rebuild only (repomap.ts exists;
  flag runtime-flags.ts:79). *Success: map rebuild count per 100-turn session drops ~10× at equal recall.*

## Phase 1 — Finish the durable core, port reliability (V2 becomes feature-complete)

- **Limitation/opportunity**: the shipping loop is the 1,920-line V1 monolith; V2 lacks retry, doom-loop,
  output-length, MCP/plugin tools, status, and a typed overflow error; steering vocabulary is dead code in
  production (01 Part A findings 1–5).
- **Changes**: (1) port `SessionRetry.policy` + side-effect-aware canRetry semantics into
  `core/session/runner/llm.ts`; (2) port length-guard + capped synthetic continues; (3) overflow/transition
  defects → typed `RunError` channel incl. the post-compaction-overflow case (baseline Part A §11.4); (4) port
  doom-loop (with oh-my-pi's argument canonicalization) as a first-class runner policy; (5) per-call tool-status
  rows (gemini scheduler.ts:309-313 pattern) written through the event publisher; (6) unanswered-tool-call closure
  at admission (geminiChat.ts:528-535 pattern); (7) loop-detect LLM double-check + recovery turn (gemini
  loopDetectionService/client.ts:1300 pattern); (8) MCP + plugin tools materialize in V2; (9) async/streaming V2
  HTTP surface replacing the blocking `promptV2`; (10) startup sweep: re-arm + failInterruptedTools for open turns.
- **Informing implementations**: OpenCode (shipped V2 twin), Gemini CLI, oh-my-pi, Codex persist-policy.
- **Dependencies**: none beyond each other's PR ordering; W1 early helps gating. **Difficulty**: M–L (bulk of it is
  disciplined porting, all with existing V1 tests to port).
- **Expected benefit**: V1/V2 divergence ends; every later phase lands once, not twice.
- **Success criteria**: V2 test suite + ported V1 behavior tests green; E7/E8 harnesses built and passing; V2 vs V1
  on the existing eval long/ladder set within 1 pass of parity at equal-or-better tokens.
- **Regression risks**: V2 subtle projection bugs reaching default users → ship behind same flag, flip
  progressively; tool-status rows adding write load → batch in publisher (single-permit semaphore precedent).

## Phase 2 — Tool-surface convergence + context portfolio (S1, S7)

- **Limitation/opportunity**: four tool-reach layers, ~6k-token declarations (tools.ts:41-45), flag-off repo
  intelligence, estimate-driven context, cache policy implicit (04 §B/C, F3/F4).
- **Changes**: one deferral seam (pi ToolExposure pattern) replacing layers; repair aliases demoted to error
  recovery only; repomap on (aider constants, W6 refresh); LSP tool on with output projections +
  `lsp_replace_symbol` (crush); upgrade guard's shipped per-session read-table from its advisory `[zero-guard]`
  nudge (01 D2) to mtime-aware freshness enforcement for edit/write (crush edit.go:288-302 pattern);
  `ContextPortfolio` service (floors/reserve/fit as data; ground-truth triggering; masking to spill files with
  protection window); cache-as-invariant placement table + per-provider cache accounting incl. implicit proxy;
  compaction-as-replayable-event.
- **Informing**: aider repomap.py constants; gemini contextManager/masking; crush lsp/edit; pi ToolExposure;
  antigravity budget floors (1.2.7).
- **Dependencies**: Phase 1 (single loop to instrument); W4 for cache signal. **Difficulty**: M (largest: portfolio
  service; rest S/M each).
- **Expected benefit**: declared tokens −25–40% at parity (E1); prefix_broken halved (E3/E5 readouts); edit-corruption
  class measurably reduced (E2).
- **Success criteria**: E1/E2/E3 results in `docs/harness-research/` meeting 06 thresholds; repomap/LSP flags
  removed (on).
- **Regression risks**: over-deferral hiding tools from weak models (pin core four; measure tool-not-found rate);
  freshness enforcement friction (allow `force` with recorded rationale, watch rate).

## Phase 3 — Asynchronous autonomy (S2 core)

- **Limitation/opportunity (corrected 2026-10-14)**: the autonomy organs are NOT absent — the goal plugin ships a
  real done-ness driver (`/goal` + `--verify` exit-0 oracle + marker-vs-todos + continue/todos-continue,
  default-on; 01 D1), guard ships unfinished-reply continues and pitfall lints (D2), the design plugin is a shipped
  auto-routed review gate (D4), and desktop ships a recurrence scheduler that starts saved prompts as new sessions.
  What's missing is the **durable core version**: goal state is a closure Map (restart kills it and resets the
  cap), everything is `session.idle`-gated (no wake for an existing session after process exit), the oracle isn't
  a durable row, budget counts continues not costs, and the core Executive is dead code (01 Part A §6). The phase
  **starts from the goal plugin: port, harden, make durable — do not build from scratch.**
- **Changes**: `session_wake` table + scheduler fiber (timer/process/event/manual; fences + coalesce — qwen MQ2c
  watermark pattern, Kilo caps); `objective` + `done_predicate` rows (cleanroom `--verify` promoted);
  Executive wired advisory at idle boundaries (logged `would:` priced); recipe artifact (goose format subset:
  prompt+params+model pin+response schema+sub-recipes) + session→recipe extraction; startup re-arm of wakes.
- **Dependencies**: Phase 1 durability; Phase 2 ledger feedback for pricing. **Difficulty**: M (kernel S–M;
  recipes S). **Research**: E6 gates arming the Executive; done-ness pricing decides default-on.
- **Expected benefit**: multi-hour/daily autonomy loop closes: park→wake→check-oracle→continue/escalate; nightly
  automated chores without babysitting processes.
- **Success criteria**: new `long-park-resume` fixture survives kill+restart with zero duplicated provider calls;
  E6 halves false completions at <40% time overhead; wake storms bounded (max wakes/session/hour under load test).
- **Regression risks**: wake loops burning tokens (caps + ledger kill-switch); oracle gaming (cleanroom copies);
  scheduler startup sweep racing live sessions (fences).

## Phase 4 — Verification + model-aware orchestration (S5, S4)

- **Limitation/opportunity (corrected 2026-10-14)**: false completion only partly measurable — `/goal --verify`
  and the design review gate already verify/review at idle (01 D1/D4) but non-durably, same-model, without claim
  auditing; the *model-tier* routing (cheap reviewer vs frontier reviewer) and written acceptance contracts are
  the open parts; model quirks still live in prompt forks (05 §3, ladder table).
- **Changes**: acceptance-contract artifact (commands + pitfall checklist) with cleanroom verifier; claim-audit
  second-model pass; Executive armed (E6 evidence); ModelTable-as-data (traits/formats/budgets — aider/kimi/qwen
  pattern) with the dead `SessionRunnerModel.resolve(override)` seam now fed by the Executive; frontier boundary
  client (approve/verify/triage; aider empty-history pricing); per-model format-bar + budget auto-sizing
  (kimi `completion = window − used`); E5 weak-model package.
- **Dependencies**: Phase 3 Executive; Phase 2 model-flag data from ledger. **Difficulty**: M. **Research-heavy**:
  E5/E6 decide which boundaries are worth arming; anything not confirmed stays advisory.
- **Expected benefit**: local-primary/frontier-at-boundaries within blind-review noise of frontier-only at <40%
  frontier tokens (07-S4 threshold); measurable lift on weaker local models (E5).
- **Success criteria**: E5 ≥1 ladder tier or ≥30% fewer failed tool cycles; E6 thresholds; ModelTable covers ≥90%
  of the cases the nine prompt `.txt` forks special-case (fork fade begins).
- **Regression risks**: expensive-model routing bugs (caps + per-repo opt-in + privacy note); verifier gaming
  (cleanroom); contract overhead (auto-skip small tasks).

## Phase 5 — Parallelism + PTC + sandbox (S6, S3, sandbox lane)

- **Changes**: declared file accesses → conflict-aware intra-turn scheduler (kimi); worktree `branch|share` modes
  + durable child transcripts cited by id + anti-injection envelope (Antigravity/Claude); PTC presentation mode
  over the existing codemode interpreter with DeepSeek discipline (caps, executor-enforced mode, dispatch event
  pairs, curated output, generated typed prompt); E1(d) decides PTC default posture; optional OS-sandbox lane
  (seatbelt/bubblewrap + execpolicy-style rules + classified denials + protected subpaths; denial-respect rule).
- **Dependencies**: Phase 1 status rows (dispatch events), Phase 2 single dispatch seam. **Difficulty**: M each;
  sandbox L if landed with UX.
- **Success criteria**: two-writer fixture zero lost updates; E1(d) fan-out tasks +20% or tokens −40% at parity
  else PTC stays experimental; sandbox: zero unclassified denial reports in eval runs.
- **Regression risks**: merge friction (default share for read-only, worktree only for background writers);
  sandbox CPU-quirk tail (antigravity audit §7.10 — budget it); program-debuggability (journal dispatch events).

## Ordering rationale and what can proceed independently

Phases 1→2→3 are strictly ordered (each instrumented/flagged on the previous). Within 2 and 4, items are
independent PR-sized units (freshness table, masking, ModelTable row by row). 5's three items are independent of
each other. Quick wins W1–W6 land anytime. The genuinely *research* items (E1–E8) interleave as gating experiments;
nothing downstream of a gate proceeds on vibes.

**Vertical slice (first demonstrable win, ~end of Phase 2 + W5/W6)**: a long task (`long-qa-spec`) at forced 64k
context runs to hidden-test pass with fewer compactions, ≥80% recall-probe score, visible archive pointer,
halved prefix-break count, and a repo-map-attributed navigation advantage over the current default — all four
numbers from one report, all mechanisms shipped. That is a measured improvement over today's harness, achieved
without touching autonomy semantics, and it validates the portfolio/cache economics bet before Phase 3 spends on it.

**Done when (roadmap, not this research run)**: the 06 program's ledger shows every Phase-1/2/3 gate measured,
V1 deleted, and a two-day unattended task mix (nightly recipes + park-resume + verifier) at ≥90% honest completion.

## Correction addendum (2026-10-14 plugin audit)

Phase 3's concrete starting point, item by item (source: 01 Part D, live-DB evidence in D6):
- **Port `/goal` → durable rows**: GoalState closure Map (`goal/src/index.ts:267`) → `objective` +
  `done_predicate` rows; the `/goal` command becomes a UI over them; `/goal status` stops lying after restart; the
  iteration cap moves to durable accounting and gains token/time budgets alongside continue counts.
- **Port the verify runner**: the perl-alarm exit-0 runner (:149-156) keeps its semantics but runs from a cleanroom
  copy and emits durable check events (goal parts today are transcript-only forensics).
- **Port continues → wakes**: the idle-continue path becomes the event kind of `session_wake`; the
  goal-less todo-continue mode becomes V2's own "eligible work" rule; the restart gap (continue 13 / start 10 /
  done 5 in the live DB, three goal starts silently recycled) is exactly what the startup sweep closes.
- **Port wake-on-process**: `engine/memory.ts` already wakes a mid-turn wait when a session-started shell job ends
  (`Effect.raceFirst(sleep, ShellJobs.anyEnded)`, :91-160) — generalize that primitive into the
  `wake(kind=process)` row instead of inventing one.
- **Port desktop schedules → session→recipe**: the recurrence scheduler (`desktop/src/main/schedules.ts:74-110`,
  hourly/daily/weekdays/weekly/once + run history + 100 cap) is the existing format; the phase adds
  agent-callable management, session-scoped wake (it currently starts *new* sessions), and session→schedule
  extraction.
- **Port guard/design gates to V2 boundaries**: guard's loop-window (ring of 12, repeats 3 — complements the
  consecutive doom-loop) and unfinished-reply continue move to runner policy; the design gate becomes the shipped
  first boundary kind under S4's routing.
- **Plugin API ceiling is why this is core work**: plugins get no durable state, no startup replay, no scheduler
  (01 D5) — the goal/guard/design behaviors cannot become durable autonomy without core storage and boundary
  checks. The plugins stay as the user-facing surface and third-party extension point.
