# Agent kernel and durability — subsystem study

DRAFT — P2 study, written 2026-10-14, awaiting cross-check

Scope: turn loop, message/tool-call settlement, durable admission and interruption recovery, retries, history
projection, per-session serialization. Prerequisites: `docs/harness-engineering/PRIOR_WORK.md` and
`docs/design/2026-09-22-session-runtime-review.md`, whose retired-V2 ideas are requirements (tools settled outside
the provider stream, restart settlement of interrupted tools/turns, predictive overflow, durable admission with
steer/queue — decisions 2 and 7). Current behaviour is normative in `docs/harness-engineering/spec/agent-loop.md`
and `spec/server-api.md`. Clone reads are read-only at the pins in `docs/harness-research/SOURCES.md`; the old
the owner's earlier prototype was not read.

## 1. How each implementation works

**OpenCode `3ec856b` (our inherited V2 lineage).** Admission is durable and returns: `Session.prompt`
reconciles/commits an inbox row and only *advisories* execution (`session.ts:145-178`, wake `:174`). The inbox holds
typed pending items `user|synthetic|compaction|move` with delivery `steer|queue` (`inbox.ts:43-48`); `admit` is
idempotent first-writer-wins on message ID, conflicts raise a typed `LifecycleConflict` (`:155-202`). Promotion
happens only at boundaries, in one transaction: steers at a step boundary, exactly one queued item at the idle
boundary (`inbox.ts:43-48`), enforced by a drain eligibility check that returns without a provider call when only
parked queued items remain (`runner/llm.ts:64-69`). Execution is a per-session doorbell coordinator: one fiber per
busy period, wakes coalesce on the scope lattice `input⊇steer` (`run-coordinator.ts:136-145`), a mid-drain doorbell
trampolines a re-drain (`:74-85`), a late doorbell starts a successor (`:114-120`). Interruption is a
fire-and-forget ack that claims already-recorded wakes so cleanup cannot spawn a dead-intent successor; wakes
arriving during cleanup restart normally (`:147-167`). A write-ahead *claim* commits with the started event;
terminals release it except shutdown interruption, which keeps it so the next start resumes the turn — "recovery is
a property of the database, never of a shutdown hook" (`execution.ts:76-86,128-135`). Before any provider call a
drain settles orphaned `running` compactions and `streaming|running` tool calls as interrupted, carrying the
`executed` flag so a side effect that really ran is not replayed (`runner/llm.ts:71-72,332-385`). Tools settle as
fibers forked mid-stream at each `tool-call` event, joined after stream close under uninterruptible final settlement
(`runner/step.ts:73-76,122-134,141,151-153`), with exit classification separating declines, interrupts and defects
(`:298-321`). Continuation is a tagged outcome: transport fail *before* output → full retry; fail *after* output →
durable synthetic "previous response was interrupted, continue" (`llm.ts:35-39,281-289`, `step.ts:183-188,263-271`);
`length` → anti-apology continuation, hard stop after 3 consecutive length finishes read from durable history
(`llm.ts:290-319`). Retry: 2 s→10 s ×10 jittered, `retry-after` honoured capped at 15 min, timeout-class failures
capped at 3 per step (`retry.ts:31-60,76,84`). Residual gaps: ownership is process-local; the prompt call streams
nothing back — clients replay the event log.

**Pi `3daa202` (turn model).** The loop is a pure function over an `AgentMessage[]` context with injected callbacks,
no storage or session knowledge (`agent/src/agent-loop.ts`). `agentLoopContinue` is the retry/resume primitive and
refuses to continue from a trailing assistant message (`:71-100,:81-83`). Outer loop drains follow-ups, inner loop
runs tool/steering turns (`:163-321`); steering is polled only after the current turn's tool results settle (`:295`)
and re-polled after a long `prepareNextTurn` such as compaction (`:201-206`) — never mid-tool-batch.
`error`/`aborted` are hard exits (`:245-256`); a `length` finish fails *every* tool call in the message because
streamed arguments may be truncated (`:264-270`); `finishTurn` may force exactly one more provider turn
(`:286-298,:310-314`). Durability is being rebuilt in `packages/durable`: core rule "only committed state is
observable", volatile publication and external effects inside the commit transaction forbidden
(`docs/spec.md:41-44,63-78`). Every task is a durable state machine `pending|running|waiting|completing|terminal`
with an `orphaned` outcome and records carrying `abandonOnRestart`/`abortReason:"restart"` (`spec.md:1650-1703`).
The tool task is the exactly-once mechanism: it commits intent — `{phase:"execute", arguments, replay:
"safe"|"unsafe"}` — *before* running; on restart the `execute` phase re-executes only if both stored and current
replay policies are safe, else settles `interrupted` from the slot's durable partial output
(`spec.md:4075-4077,4093-4107`). Abort is a durable mark and a run invocation may not commit after it appears
(`:1985-1993`); clean `close()` writes no outcome so work resumes on reopen (`:2016`); cascade marks are re-derived
"again at open, so a crash in between loses nothing" (`:2041`). Submissions carry `requestId` dedup (`:2332`) and a
`placed/queued/done/unanswered` lifecycle with deterministic boundary selection (`:2259-2350`).

**oh-my-pi `bcd11cb` (loop + durability extras).** Same pi loop hardened three ways. (1) Cross-turn loop guard
canonicalizes tool args (sorted keys, intent field stripped, `packages/ai/src/utils/tool-call-loop-guard.ts:40`),
counts identical consecutive assistant tool turns with an exempt list, and answers with a shared-wording redirect
record used by primary-session guards and the advisor's alike (`tool-call-loop-guard.ts:67-98`;
`packages/coding-agent/src/session/tool-call-loop-redirect.ts:13-21`). (2) Startup recovery: resuming after process
exit pairs every pending tool call in the trailing assistant message with a synthetic "outcome is unknown" result so
the transcript never dangles (`exit-diagnostics.ts:251-291`); a fork from a *still running* parent instead preserves
the unknown outcome rather than asserting the tool never ran (`session-manager.ts:4097-4131`). (3) Compaction
entries carry `firstKeptEntryId`, `tokensAfter` and `providerReplayThroughEntryId`, tying the summary to the last
entry covered by a native provider replay snapshot, so summary and provider context cannot drift on resume
(`docs/compaction.md:31-42`).

**Codex `6786dcc`.** Turn = task run of many provider steps. `TurnStarted` is emitted inline so lifecycle never
waits on prewarm; the task re-runs `run_turn` while steering input is queued, and on a terminal error deliberately
stops and *preserves pending input* instead of restarting the failed turn for it
(`core/src/tasks/regular.rs:48-50,115-120`). Each step drains pending input into history durably
(`PersistContext::SteeredUserInput`) and only then sets the `RecordedTurnInput` flag
(`core/src/session/turn.rs:437-459`) — a turn is capturable for daemon recovery only after its input is recorded
(`daemon_recovery.rs:9,19-56`), and only for one local environment. `Op::RecoverTurn` re-drives a captured turn;
`Op::SuspendTurnAndShutdown` exits only after history is durable and the rollout writer has closed
(`handlers.rs:574-602`). Tools settle through a `FuturesOrdered` drained before the step ends while the stream is
still consumed (`turn.rs:2634,3237`). The rollout is the single durable artifact: typed items behind an explicit
persist policy (`rollout/src/policy.rs:80-99`), single-writer JSONL whose `flush()` is a oneshot ack with
reopen-and-retry, so a transient write failure never drops buffered items
(`rollout/src/recorder.rs:1084-1100,1192,1789-1851`). Forks drop the unfinished turn suffix at the active turn's
opening boundary (`core/src/thread_manager.rs:192-208`).

## 2. What is genuinely best, and why

- **OpenCode's admission/coordination core**: admit-and-return + doorbell coalescing + claim-held-on-shutdown gives
crash continuity in ~750 readable lines, each piece testable alone (`run-coordinator.ts` is storage-agnostic); tools
outside the stream (`step.ts:122-134`) make interruption and retry distinguishable — our review's core requirement,
implemented.
- **pi-durable's replay-policy checkpoint is the best exactly-once story anywhere**
(`spec.md:4075-4107`): it names the three legal answers to "we died mid-execution and cannot know if the side effect
landed" — re-run iff safe, settle interrupted from durable partials, never guess. oh-my-pi's
`exit-diagnostics.ts:251-291` is that idea one level cheaper, in production. Zero V1's duplicate-subagent retry
(F11, `6ce4f7b644`) was exactly this hole: results persisted, request didn't, retry re-issued the calls.
- **Codex's ordering discipline**: input recorded → flag set → only then recoverable (`daemon_recovery.rs:32`, `turn.rs:459`);
shutdown ordered after the flush ack (`handlers.rs:592-602`, `recorder.rs:1084`). Durable admission needs this so
recovery never resumes from a half-recorded turn.
- **Pi's loop skeleton**
(`agent-loop.ts:245-298`): hard-exit on error/aborted, length-fails-all-calls, steering only at settle boundaries —
the vocabulary our length-guard and doom-loop features already speak.
- **`requestId` submission dedup**
(`spec.md:2332`): the cheapest correct answer to retried POSTs, matching our AGENTS.md exact-retry rule.

## 3. What the new core should combine, cross-checked against measurements

- Durable inbox (opencode) + `requestId` dedup (pi). Structurally fixes the measured F3 bug — a message sent to a
full session folded into the summary and lost (`docs/evals/2026-09-23-long-horizon-results.md` Harness-bugs;
SYNTHESIS F3, patched by `632e803b5d`): a promoted prompt is its own row and compaction is a control item that jumps
the queue *without consuming* user rows (`llm.ts:91-95`). F4 (finished answer followed by unrequested work,
`7a25304f26`) becomes structural the same way: eligibility is checked before the provider call (`llm.ts:64-69`) —
also the predictive-overflow requirement.
- Tools settled outside the stream (opencode fibers; review decision 2)
with per-call intent + replay checkpoints (pi-durable). F11's retry-safety fence (`spec/agent-loop.md`,
`processor.ts:86-89,729-746`) papers over tools-in-stream; outside settlement makes it unnecessary — a retry can
never re-enter an executed call.
- Claim-held-on-shutdown (opencode) + recorded-input gating and flush-ack
shutdown order (codex) + restart settlement of stale tools/compactions (opencode `llm.ts:332-385`; oh-my-pi
unknown-outcome pairing). Our only recovery measurement is one run (`2026-09-17-zero-recovery.json`, 8.1 s/12.0 s
phases, SYNTHESIS win #13) and no other harness has long-horizon recovery data — new measurement decides, not
comparison.
- Outcome-enum continuation (opencode `llm.ts:273-328`) replaces ad-hoc branches, but the two *Local*
retry behaviours survive verbatim: memory-refusal retry with engine wait, and engine-outage health-wait without
burning retry budget (`spec/agent-loop.md`, 2026-10-07 incidents).
- One provider call at a time per session
matches the engine reality — one heavy request, prefix-cache hits decide speed (`PRIOR_WORK.md` operational facts);
80-95% of long-horizon wall is model time and compaction alone was 16-24% (SYNTHESIS §3). The stable-prefix A/B
measured no gain (SYNTHESIS tried-and-not-helped), so the target is *not breaking* cache state — e.g. keep tool
definitions on the capped step (`llm.ts:243-244`) — not new machinery.

## 4. Proposed design

1. **Durable admission.** `prompt(id?, steer|queue)` writes one inbox row (existing V1 SQLite tables) idempotent on
   message ID — exact retry reconciles, conflict fails — returns immediately after commit; `resume:false` admits
   only; advisory wake after. Mid-run prompts persist immediately and are picked up at the next history reload,
   mirroring V1 steering semantics (`spec/agent-loop.md` "Turn orchestration") so existing prompt.test steering
   tests keep equivalents.
2. **Per-session doorbell coordinator** ported from `run-coordinator.ts` shapes: busy-period fiber, scope lattice,
   interrupt claims, successor-on-late-doorbell. One session never has two provider turns; different sessions run
   concurrently.
3. **Exactly-once provider calls across restart.** Write-ahead execution claim committed with turn-started, released
   by every terminal except shutdown. Per attempt: record attempt-start before `llm.stream`; per tool call: commit
   `{call, arguments, replay: safe|unsafe, startedAt}` before executing, `{result|interrupted, executed}` after. On
   start, settle `running|streaming` tools and `running` compactions as interrupted (executed-but-unknown →
   unknown-outcome result) before any provider call. Clean shutdown = codex order: stop intake, settle/keep claim,
   flush history, exit on ack.
4. **Steer/queue vocabulary.** Steers promote at the next provider-turn boundary (one batch resets the step
   allowance once); queue promotes exactly one item only when the session would otherwise idle; compaction/synthetic
   controls jump steers but never cross a `move`-class item — the AGENTS.md rules, implemented as
   `inbox.ts:43-48`+`llm.ts:64-69`.
5. **Interruption semantics**, so every `spec/agent-loop.md` behaviour stays
   testable: ack on acceptance, cleanup settles async (`awaitIdle`/`awaitSettlement`); running tool parts get `error
   + interrupted:true` with the real `executed` flag (replaces the 250 ms settle hack, `processor.ts:650-676`);
   user-interrupt releases the claim, shutdown keeps it; queued items stay parked across interrupt-with-resume;
   background jobs keep the V1 parent→child cancel tree. The prompt.test cancel group re-expresses as: bounded ack,
   no provider call after ack, orphans settled with truthful executed flags.
6. **Retry/continuation as an outcome enum**: pre-output transport fail → retry (existing `retry.ts` ladder + the
   two engine-local waits); post-output stream fail → durable partial + synthetic "continue without repeating";
   `length` → anti-apology continuation, cap 3, `MessageOutputLengthError` stays; provider overflow → predictive
   check, then compaction control item, then one re-attempt. Keep asserting the `6ce4f7b644`
   no-retry-after-tool-call fence as a test until outside-settlement ships.
7. **History projection stays event-folded**: admissions, settlements and compaction boundaries are events folded
   into the V1 message tables the app and `/event` SSE already read (`spec/server-api.md` V1 surface); the
   background-jobs durability gap (`server-api.md:52-58`) closes as a consequence.

## 5. Measurable performance targets

- Interrupt ack p95 < 100 ms (acceptance, not settlement); `prompt` POST p95 < 50 ms admitted-and-durable (today the
V1 call blocks for the drain).
- Crash-resume on a fault-injection harness (`kill -9` once at every tool
boundary): a 1k-entry session reopens and settles in < 1 s with zero provider calls before settlement; executed side
effects replay 0 extra times, safe-policy tools ≤ 1, unsafe-policy tools 0 with `interrupted` results; zero
duplicate subagent spawns (permanent F11 fixture).
- Durability adds zero extra provider calls per turn and zero
extra cold prefills: long-horizon prefix hit ≥ 0.78 (SYNTHESIS §3.1 range floor) and prompt-token totals within 1%
of a no-durability arm.
- A steer admitted at t is in the next provider request at the first boundary after t
(step-boundary timestamps), never mid-tool-batch; concurrent prompts on one session never overlap provider turns
(engine request log); concurrent sessions pipeline.
- Recovery re-measured against `2026-09-17-zero-recovery.json`
(8.1 s/12.0 s) under the R1-R5 gates of `docs/harness-research/06-EXPERIMENT_DESIGN.md`; no wall-clock turn/tool
timeouts introduced (`PRIOR_WORK.md`).

## 6. Beyond / correcting the audits

- **Verified** opencode: admit+wake `session.ts:145-178` (wake :174); claims `execution.ts:76-86,128-135`; orphan
settlement `runner/llm.ts:71-72,332-385`; mid-stream tool fibers + uninterruptible settlement
`runner/step.ts:122-134,141,151-153`; outcome mapping `llm.ts:273-328`; retry caps `retry.ts:31-60`. All audit
file:lines hold at the pin.
- **Verified** pi: `agent-loop.ts:163-321` loop shape, steering polls :176/:205/:295,
length-fail :264-270. **Verified** oh-my-pi loop-guard claims (`tool-call-loop-guard.ts:40,67-98`,
`tool-call-loop-redirect.ts:13-21`).
- **Corrected** codex audit §5/§12.2: forks do *not* truncate at a "sampling
boundary" — `thread_manager.rs:192-208` makes that an explicit TODO ("once core can represent sampling boundaries
directly instead of relying on whichever items happened to be persisted mid-turn"); the shipped cut is at the active
turn's opening (user-message) boundary. The conclusion (no fork inherits a half-finished turn) survives; the
mechanism label does not.
- **Beyond** pi audit §7: `pi-durable` is far more than "same problem space as V2
admission" — 5,340 normative spec lines define task-level durable state machines, the replay-policy checkpoint and
cascade-derived abort marks that neither our V2 nor opencode has. Richest unported design here.
- **Beyond**
tier3-scan oh-my-pi: it covers guard/compaction but misses startup recovery — `exit-diagnostics.ts:251-291` and
`session-manager.ts:4097-4131` are our restart-settlement requirement already in production.
- **Correction to our
review's framing**: the review wants "mark pending tools interrupted on restart" (V2 `failInterruptedTools`); all
four sources do the stronger thing — *pair* each pending call with a durable result carrying an
`executed`/unknown-outcome marker (opencode `llm.ts:369-381`, oh-my-pi, pi-durable `interrupted` settlement) —
history projection, not a status flag, is the recovery record.

## 7. Sources

- Clones (read-only, pins from `docs/harness-research/SOURCES.md`) under
`~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`: codex `6786dcc` (2026-10-13, paths relative to
`codex-rs/`), pi `3daa202` (2026-10-13, `packages/`), oh-my-pi `bcd11cb` (2026-10-13, repo root:
`packages/coding-agent/src/session/…`), opencode `3ec856b` (2026-10-14, `packages/core/src/session/`).
- Audits:
`02-COMPETITOR_AUDITS/{codex,pi,opencode}.md`; oh-my-pi via `tier3-scan.md` §2 (no standalone audit exists).
- Zero Code: `docs/harness-engineering/spec/agent-loop.md`, `spec/server-api.md`,
`docs/design/2026-09-22-session-runtime-review.md`, `docs/harness-engineering/PRIOR_WORK.md`,
`docs/evals/2026-09-23-long-horizon-results.md`, `docs/evals/SYNTHESIS-2026-09-29.md`.
