# Autonomy — subsystem study

DRAFT — P2 study, written 2026-10-14, awaiting cross-check

Scope: objectives and done-predicates, verification oracles, continue/wake scheduling, budgets, durable restart
behaviour, recipes and scheduled runs. Built on `PRIOR_WORK.md`, `01-ZERO_CODE_BASELINE.md` Part D +
`PROGRESS.md` verdicts (W1 WRONG, P2/P3/P8 PARTIAL, C33/C35 CONFIRMED), `09-IMPLEMENTATION_ROADMAP.md` Phase 3 +
correction addendum, `06-EXPERIMENT_DESIGN.md` E6, `rsi-harnesses.md` rung 3. Zero Code behaviour is normative in
`spec/plugins.md`, `spec/agent-loop.md`, `spec/server-api.md`, `spec/app-desktop.md`; source read only for gaps.
Clone reads are read-only at the pins in `docs/harness-research/SOURCES.md`; the owner's earlier prototype was
not read. Correction commits honoured: `516106c2a7` (goal restored from transcript), `eea6bdc514`/`254d3e6837`.

## 1. How each implementation works

**Zero Code today (spec + Part D).** `/goal` is a done-ness driver, not durable state; since `516106c2a7` the
closure state is rebuilt from the last start/stop/done transcript part after restart, iteration resuming at the
last continue (`goal/src/index.ts:254-270,296-302` — `goal.test.ts:459-501`, `spec/plugins.md:20`). Oracle =
exit-0 of `--verify` via perl-alarm timeout (120 s) else exact `GOAL COMPLETE` line; the marker does not end a
goal whose todos (touched since start) are open (`:94-96,149-156,324-327`). Continues fire only on
`session.idle`, capped at 20 continues — not tokens or time; goal-less todo-continue caps 8 per user message,
blocked by question/deny/dismiss/untouched-todos (`:158-222,444-450`). `engine/memory.ts:91-160` already races a
memory-refusal wait against `ShellJobs.anyEnded` — a real wake-on-process primitive, in-turn only (01 D3).
Desktop `schedules.ts:76-127`: 30 s tick, hourly→once, starts a NEW session via `prompt_async`, cap 100, missed
run fires once, failed runs advance (`spec/app-desktop.md:26-31`). `core/src/background-job.ts:113-119,202-335`:
process-local registry, `wait` snapshot+`timedOut`, `promote` flips metadata and releases waiters; restart loses
all status — the gap deliberately deferred to this study (`spec/server-api.md:49-58`). Loop budgets: `agent.steps`
with `MAX_STEPS_PROMPT` prefill (`prompt.ts:1306-1307`), length-continues max 3 (`prompt.ts:68-89,1123-1154`),
guard unfinished-reply continue ≤2 (`guard/src/index.ts:22,65-100,274-309`). Live-DB (01 D1): 13 continues / 10
starts / 5 done; three starts silently recycled; a premature `GOAL COMPLETE` passed only because the todo guard
was unengaged.

**Kilo Code `4ee8d1b` (goal loop; paths rel. `packages/opencode/src/kilocode/`).** The strongest TS park/wake.
Goal state is persisted *session metadata* `kilo.goal {text,status,active,reason,wait}` with status
`active|complete|blocked|paused|waiting` (`session/goal/state.ts:20,41-66`); a mapper projects a live-run goal to
`active` and a token-less one to `paused` (`state.ts:126-131`) — the restart rule: an *active* run degrades to
paused, a *waiting* goal hydrates with its wait (`runner.ts:852-858`, `goal/link.ts:121-128`). The model must
PARK: a time-based objective is curbed to a tools-whitelist of `schedule_wakeup | cron_create |
background_process` until a wait is armed (`policy.ts:12,31-42`, `instructions.ts:9,16`); a turn that armed a
timer is explicitly *not progress* and suspends the goal (`runner.ts:59-84,174-184,506-532`). Progress =
success/failure classification over every tool part; no successful action or report → paused; rejection → blocked;
errors → paused (`runner.ts:86-207,479-489,533-538`). `goal/link.ts:27-59` is the single classification table for
the six scheduling tools; `take()` clears a wait only on id match (`:85-90` — a fence); fires landing on an
in-flight turn queue and merge into the next continuation note instead of racing a second run
(`:141-161,169-193`; `runner.ts:490-531`); process waits poll to terminal status in the instance scope, raced
against supersession (`link.ts:222-238`, `runner.ts:406-411`). Wakeup scheduler: one-shot + cron records persisted
under `["wakeup"|"cron",session,id]` (`wakeup/index.ts:64-65`); caps 10+10 and a 7-day horizon *clamped, with the
clamp named in the tool result* (`wakeup/schema.ts:15-19,154-199`); count-and-write serialized by a semaphore so
two schedulers cannot both pass a cap (`index.ts:80-89,294-310,340-351`); one-shot persistence dropped *before*
resume, cron next occurrence persisted *before* fire, in-flight guards so `adopt()` cannot double-fire
(`:144-163,209-211`); missed windows skip, not replay, and a fire outliving its interval skips that occurrence but
keeps the schedule (`:175-205`); FNV-1a jitter clamped to expiry (`cron.ts:164`, `index.ts:190`); startup `adopt()`
re-arms due and future records, dropping expired (`:504-532`). Completion is still self-report — "Completion is
your report, not independent verification" (`instructions.ts:16`, `runner.ts:500-503`).

**Goose `e9e0ab6` (recipes + scheduler; paths rel. `crates/`).** The strongest declarative durable-run story.
Recipe = version/title/description, instructions-or-prompt, pinned extensions, `Settings{provider,model,
temperature,max_turns}`, typed parameters, `response.json_schema`, `retry`, `sub_recipes` with
`sequential_when_repeated` (`goose/src/recipe/mod.rs:43-129,198-207`); `summon` auto-injected for sub-recipes
(`:255-271`). Scheduling is safety-gated: bounded read (1 MiB, `scheduler/common.rs:11`), O_NOFOLLOW +
regular-file check before-and-after open (`common.rs:31-55`), validated bytes copied to `scheduled_recipes/` at
0600 with cleanup on failure (`common.rs:58-92`), *re-validated at every fire* (`full.rs:35-73`), response schema
checked at add (`recipe/validate_recipe.rs:379-409`). Jobs persist to `schedule.json`; startup
`load_jobs_from_storage` clears stale `currently_running`/`process_start_time` crash flags and skips jobs whose
recipe vanished (`full.rs:89-97,439-462`); each fire persists running flags before/after, holds a per-run
`CancellationToken` enabling `kill_running_job`, plus `run_now`/pause/unpause and `sessions()` provenance
(`full.rs:137-246,604-827`). `execute_job` runs the *same* Agent + PermissionManager as interactive chat in
`GooseMode::Auto`, creates a `SessionType::Scheduled` session and stamps it `schedule_id` + materialized recipe
(`full.rs:829-1000`). The agent schedules itself via `scheduler__manage_schedule`, which canonicalizes, bounds and
re-validates the recipe file the model wrote (`agents/schedule_tool.rs:23-59`).

**Qwen Code `6212189` (goals quartet + managed wake; paths rel. `packages/`).** The strongest *oracle* and
*budgets*. Goal state carries three independent ceilings — `tokenBudget`, `turnBudget`, `activeTimeBudgetMs` —
each spent and *re-armed only by explicit user resume* ("a budget stop is a spent authorization, not a fault:
resuming IS the re-arming", `core/src/goals/goal-reducer.ts:73-84,216-217,480-625`). Terminal `complete|blocked`
is a *proposal* gated by an independent Goal Verifier side-query: temperature 0, thinking off, exact 2-key schema,
120 s timeout, 256 KB byte-boxed evidence window (`goal-verifier.ts:18-48,212-255`). Its evidence ladder is the
best done-ness idea found anywhere: records carry `proofKind` (`user_input | external_fact | delivered_output |
execution_output`), a model-chosen wrapper cannot prove an external fact, a delegated summary alone ⇒ reject,
*insufficient evidence is a rejection, never an acceptance* (`goal-verifier.ts:36-48`). Durable restore = newest
parseable `goal_state` transcript record wins; unparseable newer records skip with a named trace
(`goal-persistence.ts:41-80`) — the versioned, honest generalization of our `516106c2a7`. Scheduler
(`services/cronScheduler.ts:42-181`): `MAX_JOBS` 50, recurring age-out 7 days, deterministic per-id jitter (≤15
min recurring; 90 s one-shots only at human-rounded :00/:30), a durable-file lock elects one firing owner, missed
one-shots batch into one confirm-first notification, overdue recur runs fire as "catch-up"; `loop_wakeup` is the
in-session self-wake: clamped 60 s–1 h, never persisted, 24 h chain cap, permission `'ask'` so the LLM classifier
still vets model-scheduled future input (`tools/loop-wakeup.ts:47-56`). The managed path derives the pending-input
set from journal replay alone — an input is owed work iff no `turn.settled` carries its `turnId`; no separate
ledger, safe across any number of restarts (`cli/src/serve/hosted-wake-intake.ts:12-59`). Kimi `b510fa9` supplies
the counter semantics: `maxSteps` fails a turn only when it settles with *no new messages*; promoting new input
resets the step counter (`packages/agent-core-v2/src/human/agent/turn.ts:289-291,840-858`).

## 2. What is genuinely best, and why (evidence)

1. **Kilo's parking protocol** — wait-as-suspend-state with a tools-curb until a wait is armed converts "waiting
for CI" from token-burning polling into zero-token suspension, caps live in a serialized count-and-write so they
cannot be raced, and the fire-during-turn queue (`link.ts:141-193`) is exactly the fence+coalesce a durable wake
table needs.
2. **Qwen's verifier oracle + three budgets** — the only shipped machine-checkable done-ness gate (evidence
provenance + reject-on-insufficient), and budgets in tokens/turns/active-time with explicit re-arm make autonomy
cost a durable per-goal fact, not a continue count.
3. **Goose's recipe durability hygiene** — validate at add *and* re-validate at fire, copy the validated bytes,
bounded symlink-safe reads, crash-flag reset at startup, run provenance on the session; scheduled autonomy becomes
an inspectable artifact, not a chat.
4. **Replay-derived pending work** (qwen `hosted-wake-intake.ts:31-59`) — the durable answer to C33 ("no durable
execution status"): derive owed work from the journal instead of a status column that can lie.
5. **Kilo's restart honesty** — active degrades to paused, waiting re-arms, expired drops, persisted waits
survive; compare D1's pre-`516106c2a7` restart (continues stopped, `/goal status` lied).

## 3. What to combine

- **Durable objective + done_predicate rows** — *port the goal plugin, not from scratch* (09 addendum): `/goal`
becomes UI over rows; verify keeps exit-0 + 30-line-tail semantics but runs from a cleanroom copy and emits
durable check events; marker-vs-open-todos becomes a second predicate kind. Transcript parts stay for display;
state moves to rows, qwen-style versioned parse (`goal-persistence.ts:41-80`).
- **`session_wake` table + one scheduler fiber** — kinds `timer|process|event|manual` (09:78). Process wake
generalizes the shipped `engine/memory.ts:91-160` `raceFirst(sleep, ShellJobs.anyEnded)` precedent and the
background-job `wait`/`promote` seam into durable rows. Fences = objective revision + wake-id match (kilo
`take`); coalesce = mid-turn fires merge into one promoted continuation note (kilo `resumeOrQueue`); per-session
caps enforced in the count+write critical section with the clamp surfaced in the tool result; missed windows skip,
not replay (kilo `rearm`; qwen catch-up only for recurring).
- **Startup re-arm** — one sweep: hydrate waiting objectives + waits, degrade orphaned active to paused (honest
generalization of `516106c2a7`), clear stale running flags (goose `full.rs:439-452`), adopt due/future wakes and
drop expired (kilo `adopt`), reconcile owed work from replay (qwen `pendingSessionInputs`).
- **Budgets in tokens and time, not continue-counts alone** — qwen's three ceilings with re-arm on resume;
kimi's promoted-input-resets-the-allowance rule (`turn.ts:854`); and the rsi rung-3 "grind" inversion — a
*floor* (minimum tokens/time before give-up) is as valid as a ceiling; a budget stop is authorization-spent, never
a fault.
- **Recipes (goose format subset) + session→recipe** — YAML `instructions|prompt`, `parameters`, model pin,
`response.json_schema`, `max_turns`, `retry`, `sub_recipes`; admitted by scheduler, API, or an agent-callable tool
that re-validates what the model wrote (goose `schedule_tool.rs`). Desktop schedules (`schedules.ts:76-127`)
become rows in the same wake/recipe tables — keeping tick, cap 100, missed-run-once — gaining session-scoped wake
(today they start *new* sessions only) and `zero recipe from-session` extraction. Honest scoping: no audited repo
ships transcript→recipe in source (§6); goose supplies the format, desktop the recurrence shape.

## 4. Proposed design for the new core

One autonomy core under V2; the shipped plugins stay the user-facing surface (01 D5). Tables:
`objective(session_id, text, status, revision, budgets max_continues/token_budget/active_time_ms + spent)`;
`done_predicate(objective_id, kind verify_command|marker|todos_open|contract, definition, durable runs:
exit/tail/started/finished)`; `session_wake(objective_id?, kind timer|process|event|manual, due_at, recurrence,
fence, coalesce_key, status pending|armed|fired|skipped|expired)`. One scheduler fiber per server: arm/adopt at
startup, sleep to earliest due, fire = durable promote (a synthetic `session_input` promoted at the next safe
provider-turn boundary — never a direct prompt call, so V1 join-running-loop and V2 promote-algebra both hold).
Goal-less todo-continue survives verbatim as the "eligible work" promote rule: open todos, cap 8 per user turn,
blocked by pending question/deny/dismiss/untouched todos (`spec/plugins.md:24-26`); guard's unfinished-reply
continue stays a runner policy capped 2 (`:30-34`). Oracle ladder, every rung keeping `spec/plugins.md`
testable: (1) `verify_command` exit-0 in cleanroom, tail recorded; (2) marker-vs-open-todos hold; (3) optional
evidence-provenance verifier side-query (qwen), off until E6 prices it. Wake storms are structurally bounded: a
fire never invents work beyond one promote; a waiting objective costs zero provider calls; caps + jitter +
skip-missed as §3. Executive stays advisory, logging `would:` at wake/oracle boundaries (09 Phase 3). Every row of
`spec/plugins.md` maps to a port test (`goal/guard/design.test.ts`); new tests: wake fence/coalesce (mid-turn fire
→ one merged promote), cap serialization, restart table-vs-transcript parity, expired-drop. The Phase 3
**long-park-resume fixture** (09:86): start goal → arm 30 s wake → durable suspend (zero turns) → kill server →
restart → exactly one fire → cleanroom oracle runs → continue promoted → second park. Assert: no provider request
between kill and fire; every objective/wake transition queryable from SQLite.

## 5. Measurable targets

1. **False-completion halved at <40 % time overhead** — E6 threshold verbatim
(`06-EXPERIMENT_DESIGN.md:113-119`); predicate rows make the rate countable from durable check events.
2. **Zero duplicated provider calls across kill+restart** in long-park-resume (09:86); replay-derived owed work
means a restart never re-asks what a settled turn answered (`hosted-wake-intake.ts:31-59`).
3. **Wake-storm bound** (09:87): under load, per-session wakes/hour capped — one-shots ≤10, recurring ≤10, wakes
clamped 60 s–1 h in-session / 7-day horizon, deterministic jitter on shared marks (kilo `schema.ts:15-19`; qwen
`cronScheduler.ts:42,65-78,152-181`); ledger kill-switch row per fire.
4. **Restart honesty**: `/goal status` and `/schedule` match durable state — zero silently recycled goal starts
(baseline had 3; `goal.test.ts:459-501` already pins the transcript-restore floor).
5. **Zero-token waiting**: a parked objective issues no requests between park and fire (ledger rows = 0 while
`status=waiting`).
6. **Budget truth**: continue-, token- and time-spent per objective; budget stop requires explicit resume.

## 6. Beyond / correcting the audits

- **kilo-code.md:32-37,83 mis-cites and under-reads restart behaviour.** `instructions.ts` is 18 lines: caps are
*stated* at `:13,16`, *enforced* at `wakeup/schema.ts:15-19` + `index.ts:294-344`; the classification table is
`goal/link.ts:27-59` (the audit's `:8-14` is the doc comment). "Paused on restart" is half true: active runs
degrade to paused (`state.ts:126-131`), but waiting goals re-hydrate and *persisted* wake/cron records are adopted
at startup, due ones fired (`runner.ts:852-858`, `wakeup/index.ts:504-532`). Kilo's waits survive restart; only
the run fiber dies.
- **goose.md:54 misdirects "session→recipe extraction".** `extract_from_cli.rs:10-52` runs the opposite way —
recipe → CLI session (`InputConfig`, plus sub-recipe splicing); `session-recipes.md` means running a recipe *in* a
session (desktop `Save & Run`), and no transcript→recipe generator exists anywhere in audited source. The
capability is real (Goose desktop, closed) but the roadmap item is ours to build; goose supplies the format side.
The audit also under-reads the scheduler: crash-flag reset (`full.rs:439-452`),
`kill_running_job`/pause/run_now/sessions (`:604-827`) and per-fire re-validation (`:35-73`) are the copyable
pieces.
- **qwen-code.md:200 cites a wrong path**: `hosted-wake-intake.ts` lives at `packages/cli/src/serve/`, not in
`managed-runtime`. Missed additions: wakeup bounds 60 s–1 h + 24 h chain cap + never-persisted
(`cronScheduler.ts:74-117`), the `'ask'`-permission rationale (`loop-wakeup.ts:47-56`), and the evidence-
provenance verifier (`goal-verifier.ts:36-48`) — the most transferable autonomy idea in the survey.
- **Beyond all audits**: nobody names kilo's fire-queue (mid-turn fire merges, never races — `goal/link.ts:141-161`
+ `runner.ts:490-531`) or qwen's budget *floor* framing (rsi rung 3; `goal-reducer.ts:216`) as primitives; both
enter the design. Kimi's step-reset-on-promotion (`turn.ts:854`) is the missing semantics for how `agent.steps`
should interact with wakes.
- **Our own corrections stand**: 01 D1's "restart silently downgrades" is fixed by `516106c2a7` (transcript
restore); the durable row makes the same guarantee explicit and versioned. D3's `engine/memory.ts` wake remains
the strongest internal seed; D5's plugin-ceiling argument (no durable state, no startup replay, no scheduler) is
why the wake scheduler must be core.

## 7. Sources

- Clones (read-only, pins per `docs/harness-research/SOURCES.md`) under
`~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`: kilo-code `4ee8d1b` (2026-10-13, paths rel.
`packages/opencode/src/kilocode/`), goose `e9e0ab6` (2026-10-13, rel. `crates/`), qwen-code `6212189` (2026-10-13,
rel. `packages/`), kimi-code `b510fa9` (2026-10-13, `packages/agent-core-v2/src/human/agent/`).
- Audits: `02-COMPETITOR_AUDITS/{kilo-code,goose,qwen-code,kimi-code}.md`; `tier3-scan.md` (continuation
enforcement family — done-ness as durable predicate checked at idle); `claude-code.md` (`Cron*`/`ScheduleWakeup`
surface). Codex audit carries no goal/wake subsystem (checked: budget/guardian material only).
- Zero Code: `docs/harness-engineering/spec/{plugins,agent-loop,server-api,app-desktop}.md`;
`01-ZERO_CODE_BASELINE.md` Part D + §C8; `docs/harness-research/PROGRESS.md`;
`09-IMPLEMENTATION_ROADMAP.md` Phase 3 + addendum; `06-EXPERIMENT_DESIGN.md` E6;
`docs/research/2026-09-15-harness-and-engine/rsi-harnesses.md`; commits `516106c2a7`, `eea6bdc514`, `254d3e6837`.
