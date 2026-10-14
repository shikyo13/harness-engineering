# Observability — subsystem study (P2)

DRAFT — P2 study, written 2026-10-14, awaiting cross-check.
Scope: structured logs, per-request/per-tool ledgers, cache economics, token accounting, dashboards, and the
measurement→behaviour loop. Method: `PRIOR_WORK.md`; audits of codex/aider/opencode/kilo/roo, then deeper into
three clones READ-ONLY at `SOURCES.md` pins (§7); Zero Code from the `spec/` files and `01` Part C §6, source read
only for ledger schema and log lines. Kilo/roo: nothing local — hosted adoption plumbing (`kilo-code.md:26,90-92`),
stale branch (`roo-code.md:5`).

## 1. How each implementation works

**Codex** (`6786dccbe726`, `codex-rs/`). *Durable ledger*: the rollout JSONL *is* the metrics substrate — typed
`RolloutItem` kinds include `TokenUsageRecord` (`history/src/lib.rs:224`) and persisted `EventMsg::TokenCount`
(`rollout/src/policy.rs:133`), gated by a named persist policy, transcript vs memory separately (`policy.rs:78-120`).
*Live metrics*: dedicated `otel` crate with a metric-name registry (`otel/src/metrics/names.rs:3-51`) — turn
e2e/TTFT/TTFM (`:35-37`), `codex.turn.token_usage`/`cost_microusd` (`:41-42`), and a provider-side decomposition
of Responses-API *overhead* vs *engine inference* vs IAPI/service TTFT/TBT (`:24-33`); turn cost recorded in
micro-USD with effort/speed tags (`events/session_telemetry.rs:395-420`). *Phase accounting*: `TurnProfileState` accumulates
before-first-sampling/sampling/compaction/between-sampling-overhead/tool-blocking durations plus sampling, retry
and tools-change counts (`core/src/turn_timing.rs:57-70`), shipped as analytics `TurnProfile`/`TurnTokenUsageFact`
(`analytics/src/facts.rs:241`) through a facts+reducer pipeline (`analytics/src/reducer.rs`, `accepted_lines.rs`).
Per request a `feedback_tags!`
tracing line captures model/approval/sandbox/effort/auth as key=value (`core/src/util.rs:23-30`, `turn.rs:2579`);
one `notify` argv hook with JSON stdin for external consumers (`config/mod.rs:782-792`).

**Aider** (`5dc9490bb35f`). *Cost user-visible every turn*: `calculate_and_show_tokens_and_cost`
(`aider/coders/base_coder.py:1994-2068`) prints `Cost: $X message, $Y session` after every turn;
`compute_costs_from_tokens` (`:2070-2099`) encodes cache economics per provider family — DeepSeek cache-hit
pricing from model info, Anthropic cache-write ×1.25 / cache-read ×0.10. `/cost` re-derives the whole chat's cost
from price tables (`aider/commands.py:514-530`). *Telemetry*: PostHog with deterministic 10 % hex-threshold sampling
(`aider/analytics.py:16,30`); `message_send` events carry tokens and message+session cost
(`base_coder.py:2108-2122`). *Measurement drives behaviour*: repo-map build time is policy input —
`use_cache = self.map_processing_time > 1.0` (`repomap.py:609`) — plus a published Exercism `benchmark/` harness:
aider is the harness that argues with numbers.

**OpenCode** (`3ec856b6c265`, our logging lineage). *History is the ledger*: every mutation is a durable event on
a per-session aggregate with reserved seq (`core/src/bus.ts:21-45`); `SessionProjector` folds per-message
cost/tokens incl cache read/write into session totals (`session/projector.ts:40-67`) — accounting is a projection
of history and cannot disagree with it. Each step carries an estimated prompt-token count forward for budget math
(`session/runner/llm.ts:246`, chars/4 in `util/token.ts`). *Logs*: `packages/util/src/observability/logging.ts` —
key=value formatter flattening spans under a per-process `run` id (`:17-49`), shared append-only log compacted in
place 50 MB→25 MB with a stale-ownable trim lock (`:7-11`). *Dashboards are hosted*: `packages/stats` is a
SolidStart/Lambda cloud site (`packages/stats/README.md`); nothing local.

**Pi** (`3daa20280f6a`): the only cache-*economics* decision record found — warming fires only when
`P(continuation) × missCost − warmCost ≥ $0.05`, and the decision is persisted alongside
(`coding-agent/src/core/cache-warmer.ts:19-26,91-106`).

**Zero Code today** (specs + confirmed source). Structured key=value logs with per-process `run` id
(`core/src/observability/logging.ts:6-56`, opencode-lineage): `session context assembled` with
`context_assembly_ms`, `context_cache_hit`, `system_chars`, `prefix_breaks` (`prompt.ts:1203-1208,1490-1497`,
breaks added `2c38c09c37`). Per-message usage normalization incl cache split, reasoning, tier pricing
(`session/session.ts:353-420`, `processor.ts:474-528`) plus `timings{...,engine{...}}` (`processor.ts:512-528`);
engine prefill/decode ride raw chunks to `engine.timings` events (`llm/engine-timings.ts`; MLX only). The
telemetry plugin's SQLite WAL ledger (`telemetry/src/ledger.ts:77-121`): `requests` (tokens incl `cache_write`,
ttft, observed-window throughput, `prefill_ms` vs `engine_decode_ms` = harness-overhead delta,
`prefix_hit = cacheRead/prompt`, `prefix_broken` on cache-read regression — `ledger.ts:7-33`,
`index.ts:150-165`) and `tool_calls` (args hash, duplicate/noop/read-before-edit flags — `ledger.ts:35-50`,
`index.ts:103-123`), upsert per `message_id`/`call_id`, additive-column migration (`ledger.ts:127-139`),
dashboard on 127.0.0.1:11235 (`index.ts:169-211`). `zero-eval` aggregates these rows read-only into every P-gate
report (`zero-eval/src/ledger.ts:19-45`; gate columns `report.ts:55-128`; guard refusals detected by substring
`zero-guard` — string coupling, not a flag). The P5 gate is defined over these columns (pass rate, steps,
tokens, compactions, prefix breaks, `GOAL.md:49-50`) — but compactions come from transcript counting, no table
records a compaction's outcome, and `prefix_broken` is read by no policy (`01` Part C §6).

## 2. What is genuinely best, and why (evidence)

- **Codex separates overhead from inference better than anyone**: named metrics for overhead vs engine-inference
  vs IAPI TTFT/TBT (`names.rs:24-33`) plus per-turn phase accounting (`turn_timing.rs:57-70`) — exactly the
  attribution Zero lacks: only MLX separates harness from engine time, nothing splits sampling/compaction/
  tool-blocking inside a turn. Transcript-as-ledger (`history/src/lib.rs:224`, `policy.rs:78-133`) makes resume,
  replay and metrics read one artifact; Zero splits that truth across rows, ledger and logs with no shared key.
- **Aider's culture is the import**: measurement that lands every turn in front of the user, cache-aware pricing
  as data (`base_coder.py:2070-2099`), and measurement feeding policy (map-build >1 s rule `repomap.py:609`).
  Zero's 11235 dashboard is off the loop; aider proves per-turn lines change operator behaviour, and its
  hex-threshold sampling (`analytics.py:16,30`) keeps optional remote telemetry honest offline.
- **OpenCode's projector pattern** — usage folded from durable events (`projector.ts:40-67`, `bus.ts:21-45`) — is
  how accounting stops disagreeing with history (crashed turns leave Zero's ledger stale partial rows).
- **Zero's own columns are the best local-engine metrics found anywhere**: codex/pi/aider count dollars, which are
  0 locally (`01` §6); `prefix_hit`, `prefix_broken` and the prefill gap measure what decides local speed — the
  411× warm/cold prefill gap (192.5 s vs 0.5 s at 100k, `2026-09-15-harness-and-engine/measurements.md:163`)
  makes prefix hit the P5 primary KPI, not cost. No other harness detects cache regression per request.
- **Pi's decision records** (persist the *why* next to the action) are the template for compaction and
  cache-mutation outcomes, where Zero records nothing.

## 3. What to combine

1. **One append-only local ledger as gate substrate**: tables `requests`, `tool_calls`, `compactions`, `wakes`
   (goal continues, background-job finishes, subagent completions — the autonomy study's wake kinds), every row
   keyed by `session_id` + request/message id. Close `01` Part C §6 gaps as columns: compaction outcome
   (`tokens_before/after`, `cache_retained`, `duration_ms`, `trigger ∈ {overflow,threshold,manual,ladder}`);
   tool token cost (attributable tokens, not just `output_bytes`); route-normalized cache-write — Anthropic from
   metadata (`ledger.ts:92`), else *derived* (uncached prompt = `prompt − engine cachedTokens`) — with
   aider-style per-family cache pricing for cost.
2. **Per-component prompt-token breakdown as first-class** — the lost owner idea (`drops.md:43`
   contextQuality/contextIntelligence "partially replaced"): revive as ledger columns, not a subsystem. At
   assembly, where `system_chars`/`context_cache_hit` are already computed (`prompt.ts:1490-1497`), record per
   block into `prompt_components(request_id, component, tokens)` — `{system, instructions, tool_schemas, repomap,
   memory, reminders, history, tool_outputs}`, length÷4 estimates over text in hand. `06` §5 names this the
   prerequisite for reading E1/E3; it makes the prefix-hit floor *attributable* to a component's growth.
3. **Structured loop logs keyed to the ledger** (codex `feedback_tags!`): every loop/compaction/tool line carries
   the ledger request id, so a pass-doc footnote joins `SELECT` to `rg` on one key. Logs stay human-greppable,
   the ledger numeric-queryable — do not fold one into the other (codex keeps tracing and rollout separate too).
4. **Codex's persist policy as one module** (which event kinds land where, what is truncated — the 500-char error
   cap, what is never recorded) plus **aider's per-turn surfacing**: status line shows `hit % / tok/s` from the
   row the turn just wrote.

## 4. Proposed design for the new core

- **Ledger is a core service, not a plugin.** Core-owned append-only store in the sessions DB (WAL + busy timeout
  as today, `ledger.ts:161-162`), written by the runner and tool loop directly; the plugin layer keeps readers
  only (dashboard, summary API, export). Zero-cost hot path: one upsert per message and per settled tool call,
  `INSERT ... ON CONFLICT DO UPDATE` off the await chain (queue + flush at session end; today's
  upsert-per-message-update already survives this at <1 % harness overhead, `01` §7). No writer besides the core.
- **Schema = today's two tables carried verbatim** (zero-eval's queries must keep working — "the new core must
  write the same ledger rows", `eval-harness.md:32-35`) **plus** `compactions`, `wakes`, `prompt_components` and
  a per-run `run_meta` row (branch/commit/machine/engine/power — provenance the report assembles by other means
  today, `report.ts:55-128`). Growth via the existing additive migration (`ledger.ts:127-139`); never breaking.
- **Every P-gate column by SELECT from this one DB**, compactions included (transcript-count fallback deleted at
  cutover) and guard refusals as a typed flag replacing the `zero-guard` substring scan (`ledger.ts:38-39`).
- **Phase timing as columns, not OTLP** (codex `TurnProfile` idea minus the transport): `compaction_ms`,
  `tool_blocking_ms`, `sampling_ms` on `requests` when known, so "harness overhead <1 %" becomes a standing
  column, not a per-report experiment. A local research project needs SQL, not a collector; keep codex's
  metric-name discipline (one documented list), drop the exporter.
- **Retention**: bound the *log file* like opencode (`logging.ts:7-11`); keep the ledger unbounded — it is the
  research record (<1 MB per long run).
- **The loop closed**: the pressure/prune/portfolio gates read *measured* occupancy (context-engine study §4);
  the ledger persists every gate decision with its expected-and-actual cache delta (pi decision-record pattern),
  so gates are auditable after the fact and defaults change only with evidence.

## 5. Measurable targets (baselines cited)

Baselines: P0 `long-qa-spec@64k` — pass, 2 turns, 1 compaction, 10/10 recall, 91 % hit, 2 breaks, 28.7 tok/s
(`docs/evals/2026-10-14-p0-qa-spec.md:26-28`); summary-as-turn 98 % vs 0 % (`01:521`); hit range 0.78-0.91
(`SYNTHESIS-2026-09-29.md`); overhead <1 %, decode 80-86 % of wall (`01` §7); 411× prefill (`measurements.md:163`).

| metric | baseline | target |
|---|---|---|
| P5 gate columns from one DB | mixed: ledger + transcript scan + report-time probes | 100 % ledger SELECT; report = query+render |
| prefix hit at 64k | 91.1 % (P0 qa-spec); 98 % on cache-continuation compactions | floor ≥91 % (today's 91-98 % baseline band), breaks ≤2 and attributable |
| ledger write overhead | unmeasured (plugin today) | <0.1 % of wall, A/B with ledger off |
| compaction outcomes in ledger | 0 % (gap, `01:672-673`) | 100 %: tokens reclaimed, retained-prefix %, duration, trigger |
| per-component prompt breakdown | totals + `system_chars` only | every request answers component share (contextQuality revived, `drops.md:43`) |
| cache-write accounting | Anthropic routes only | every route: provider value or derived uncached equivalent |
| log↔ledger join | impossible | every loop log line carries the ledger request key |
| remote-provider overhead split | impossible (no engine timings, `01:670-671`) | sampling/tool/compaction split from harness-side phase columns |

## 6. Beyond / correcting the audits

- The codex audit cites `feedback_tags!`/OTel spans (`turn.rs:2579-2590`) but never inventories the taxonomy:
  the overhead-vs-inference decomposition (`names.rs:24-33`) and `TurnProfile` phase ledger
  (`turn_timing.rs:57-70`, `facts.rs:241`) are the liftable parts; the backend-coupled OTLP machinery is not
  (audit §10.3).
- The aider audit documents mechanics but misses the *culture*: per-turn cost lines (`base_coder.py:2058-2061`),
  `/cost` history re-derivation (`commands.py:514-530`), sampled deterministic analytics (`analytics.py:16,30`),
  build-time-as-policy (`repomap.py:609`), published benchmarks (`benchmark/README.md`). Add under "adopt" in its §9.
- The opencode audit is right that STATS is documented-only but incomplete: the lineage file is
  `packages/util/src/observability/logging.ts` — Zero copied the formatter, not the 50 MB→25 MB in-place log
  compaction (`:7-11`, liftable); stats is a hosted site, so Zero's local 11235 dashboard is ahead offline.
- `01` Part C §6 needs one correction: "no cache-write accounting for non-Anthropic routes" overstates —
  Anthropic routes record `cache_write` from provider metadata (`ledger.ts:92`, `session.ts:353-420`). The real
  gaps: no route-normalized cache-write *definition* (DeepSeek-style cache-hit pricing unmodeled — aider's
  `compute_costs_from_tokens` `base_coder.py:2070-2099` is the reference) and no derived equivalent for
  implicit-cache providers or the local engine.
- Lost owner metrics (`drops.md:43`): revive as columns (§3.2/§4), not the old subsystem — verdict: "replaced by
  first-class ledger columns", which also satisfies the context-engine study's attribution prerequisite
  (`context-engine.md` §3.4, `06` §5).

## 7. Sources and commit pins

Clones READ-ONLY under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/` (`SOURCES.md`):
codex `6786dccbe726`, aider `5dc9490bb35f`, opencode `3ec856b6c265`, pi `3daa20280f6a` — file:line re-read this
session; kilo-code `4ee8d1b3b1f8`, roo-code `b867ec914575` audit-read only
(`02-COMPETITOR_AUDITS/*`). Zero Code: `spec/plugins.md` (ledger,
`prefix_broken`, `2c38c09c37` loop line), `spec/eval-harness.md` (ledger reading, gate columns),
`spec/agent-loop.md` (usage accounting), `spec/context-engine.md` (engine timings, prune cache math),
`spec/drops.md:40,43`, `01-ZERO_CODE_BASELINE.md` Part C §6-7 (`:639-698`, gaps `:668-675`),
`packages/zero-plugins/telemetry/src/{ledger.ts,index.ts}`, `packages/zero-eval/src/ledger.ts`,
`packages/zero/src/session/prompt.ts:1490-1497`. Measurements: `docs/evals/2026-10-14-p0-qa-spec.md/.json`,
`docs/research/2026-09-15-harness-and-engine/measurements.md:163`, `docs/evals/SYNTHESIS-2026-09-29.md`,
`06-EXPERIMENT_DESIGN.md` §5, `GOAL.md` P5. Sibling studies: `context-engine.md` (attribution prerequisite),
`autonomy.md` (wake kinds).
