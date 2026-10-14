# 01 — Zero Code baseline: technical audit of the current harness

Audited 2026-10-13 on branch `long-horizon` at HEAD 123cb9262f. Method: three source-reading passes over
`packages/zero`, `packages/core`, `packages/llm`, `packages/codemode`, `packages/zero-eval` (agent loop, tools/repo
intelligence, context/persistence/observability — **plus a 2026-10-14 correction pass over
`packages/zero-plugins/` (goal/guard/engine/design) and post-audit commits, see Part D; corrections are listed in
D6 and inline**). Every claim carries a `file:line` reference relative to `packages/`.
Facts from reading implementation, not docs. This run is research-only; nothing was changed.

## Executive summary

Zero Code today is **two harnesses in one repo, with the older one shipping**.

- The live path is the V1 loop: TUI → `POST /session/:id/message` → `SessionPrompt.runLoop`, a ~1,900-line
  `while(true)` loop over durably-reloaded history (`zero/src/session/prompt.ts:1176`), with `SessionProcessor`
  running one AI-SDK stream per iteration.
- The V2 core (`core/src/session/`: durable `session_input` admission, `SessionRunner`, one `llm.stream` per
  provider turn, steer/queue delivery vocabulary, context epochs, Location-scoped placement) is coherent and
  better-designed, but is feature-flagged off (`ZERO_EXPERIMENTAL_SESSION_V2`,
  `zero/src/effect/runtime-flags.ts:62`), lacks retry/doom-loop/subagents/MCP tools, and its only HTTP entry
  blocks the request for the whole drain (`handlers/session.ts:335-355`).

Around the loop sits an unusually strong **tool reliability layer**: a 9-stage edit matcher cascade with
near-miss failure guidance, a single truncation service with spill files and capability-aware "do X next" hints,
tool-alias/JSON-repair/argument-coercion for weak models, doom-loop redirection, tree-sitter-parsed bash
permissions, shadow-git per-step checkpoints, and background shell jobs with push-based completion notices.
Context engineering shows real measured thinking: frozen system/tool prefixes persisted to disk for prefix-cache
stability across restarts, cache-continuation compaction summaries (98% vs 0% cache hit), verbatim-tail selection,
stable-index pressure notes.

The foundation is suitable for the intended system. **Option B (restructure subsystems, keep interfaces) over
Option D (rewrite)**: V2 already embodies the right durable design; the gap is completion and migration of V1's
hard-won reliability machinery (retry discipline, doom-loop, output-length handling, MCP/plugin tools), not a
missing foundation. The evidence for this is gathered in the three sections below; the architecture decision is
finished in 08-TARGET_ARCHITECTURE.md / 09-IMPLEMENTATION_ROADMAP.md.

Highest-severity findings, expanded below:

1. Shipping loop is the legacy monolith; every reliability fix exists twice and only the older copy runs by default.
2. V1 steering is implicit (history reload at loop iteration); the documented steer/queue delivery vocabulary is
   only live in the disabled V2 path.
3. No durable execution status anywhere; a crash mid-drain is repaired only lazily on the next wake (`runner/llm.ts:127-147`).
4. Edit semantic-corruption risk concentrates in the fuzzy matcher tiers (65% similarity replaces whole regions;
   read-before-edit is unenforced).
5. The best repository-intelligence pieces (LSP tool, repomap, code-mode) are all experimental-off.
6. Four overlapping "reach the right tool" layers (registry visibility → repair aliases → tool_search/tool_exec
   → code-mode) make the effective tool surface vary by flag, model id and MCP census.
7. Observability has the right primitives (telemetry ledger with `prefix_broken`, engine-decode vs observed
   decode delta) but nothing in the loop consumes them; no per-component prompt-token breakdown; local models
   cost $0 so cost comparisons are meaningless locally. *(Part D refinement: the engine plugin does consume live
   probe data into measured policy — thinking suppression for title/compaction agents; the ledger DB itself
   remains unread by the loop.)*

---

## Part A — Agent execution loop (source audit)

Scope: `packages/zero/src/session/*` (V1/legacy), `packages/core/src/session/*` (V2), server handlers, background jobs, subagents.
Method: read implementation top-down; every claim carries a `file:line`.

## 0. Headline finding: the live loop is still V1

The V2 durable session core is real and coherent, but it is **feature-flagged off by default**:

- `packages/zero/src/effect/runtime-flags.ts:62` — `experimentalSessionV2: bool("ZERO_EXPERIMENTAL_SESSION_V2")`.
- `packages/zero/src/server/routes/instance/httpapi/handlers/session.ts:332-334` — `promptV2` returns 404 with "The v2 session runner is disabled" unless the flag is set.
- The TUI submits through `sdk.client.session.prompt` (`packages/tui/src/component/prompt/index.tsx:1095`), which hits `POST /session/:id/message` (`.../httpapi/groups/session.ts:100`) → `promptSvc.prompt` (`.../handlers/session.ts:312-326`) → `SessionPrompt.Service` → the 1920-line legacy monolith `packages/zero/src/session/prompt.ts`.

So the shipping execution model is `SessionPrompt.runLoop` + `SessionProcessor` + AI SDK. The V2 path (`SessionV2.prompt` → durable `session_input` → `SessionRunner`) is exercised by `promptV2` and tests only. `packages/core/src/session/runner/llm.ts:45-93` is an explicit in-code TODO list admitting V2 is unfinished (durable ownership, retry bounding, plugins/MCP tools, snapshots, final status).

V2's own HTTP entry is also not wired for real use: it admits with `resume: false`, then calls `sessionV2.resume` and **blocks the request** until the session settles, returning the whole projected context (`handlers/session.ts:335-355`). No steering, no streaming, no async variant.

## 1. Prompt entry (V1, live)

`SessionPrompt.prompt` (`prompt.ts:1094-1113`):
1. `sessions.get` + `revert.cleanup` (discards a staged revert).
2. `createUserMessage` (`prompt.ts:677-1092`) resolves agent/model (`input.model ?? agent.model ?? currentModel(sessionID)`, `prompt.ts:688`), persists a user message and its parts (text, files, MCP resources, `subtask` parts), and writes agent+model onto the session row when changed (`prompt.ts:714-731`).
3. `input.noReply === true` returns early — this is how background-subagent result injection posts a message without starting a turn elsewhere.
4. Otherwise `loop({sessionID})` → `state.ensureRunning(sessionID, lastAssistant, runLoop(sessionID))` (`prompt.ts:1629-1633`).

`SessionRunState` (`run-state.ts:52-94`) keeps one `Runner` per session inside `InstanceState` (per-directory). `Runner.ensureRunning` (`packages/zero/src/effect/runner.ts:110-133`) is a `SynchronizedRef` state machine: `Idle` → fork the loop; `Running` → just await the existing `done` deferred. **A second prompt while busy does not enqueue work; it joins the in-flight loop, which picks the new message up because `runLoop` reloads history from scratch every iteration** (`prompt.ts:1200-1202`). That is V1 steering: no explicit steer/queue vocabulary, no durable admission, purely "re-read history at the next safe boundary".

`Shell` / `ShellThenRun` states serialize user shell commands against the model loop (`runner.ts:135-166`, `run-state.ts:96-105`, mapped to `Session.BusyError`).

## 2. Prompt entry (V2)

`SessionV2.prompt` (`packages/core/src/session.ts:360-386`), wrapped in `Effect.uninterruptible`:
1. `resolvePrompt` normalizes file parts, sniffing mime (`session.ts:460-472`).
2. `SessionInput.admit` (`packages/core/src/session/input.ts:41-81`) publishes a durable `PromptAdmitted` event and returns `Admitted{admittedSeq,...}`; a concurrent/lost race re-reads the row instead of failing.
3. Reuse of the same `messageID` returns the existing row (`input.ts:51-52`); if the returned record isn't equivalent by session+prompt+delivery (`SessionInput.equivalent`, `input.ts:191-202`) the call fails `PromptConflictError` (`session.ts:380-381`). Exact retry of an identical prompt is therefore idempotent; a conflicting reuse is not.
4. `delivery` defaults to `"steer"` (`session.ts:366`); `resume !== false` calls `execution.wake(sessionID)` (`session.ts:382`).

The projector writes admission into the durable inbox: `PromptAdmitted` → `SessionInput.projectAdmitted` inserts a `session_input` row with `admitted_seq` and `promoted_seq NULL` (`projector.ts:362-372`, `input.ts:83-116`, table `packages/core/src/session/sql.ts:140-164`). Promotion (`Prompted`) fills `promoted_seq` and also materializes the visible user message (`projector.ts:348-361`, `input.ts:118-168`); `projectPrompted` can lazily synthesize the record for historical prompts admitted before this code existed (`input.ts:155-167`), which is what makes the "reconcile an exact retry of a historical prompt" rule work.

`SessionExecution` (`packages/core/src/session/execution.ts:9-18`) is a Session-ID-keyed, process-global façade with `active/resume/wake/interrupt`. The only non-noop implementation is `execution/local.ts:11-38`, which builds a `SessionRunCoordinator` whose `drain` looks up the session row, then runs `SessionRunner` **provided by `LocationServiceMap.get(session.location)`** — that is how a Location-scoped runner (model registry, tools, permissions, filesystem) is reached from a global façade.

`SessionRunCoordinator` (`run-coordinator.ts:24-104`) serializes per key with per-key `Deferred` + `FiberSet` fork:
- `run` while an entry exists → await its `done`; if the entry is `stopping`, await then re-run (`run-coordinator.ts:67-79`).
- `wake` sets `pendingWake` on a live entry, else starts a drain; repeated wakes coalesce into one successor drain (`run-coordinator.ts:51-65, 81-92`).
- `interrupt` marks `stopping`, clears `pendingWake`, interrupts the owner fiber (`run-coordinator.ts:94-101`). Idle interrupt is a no-op.
- Drains have **no durable identity** and nothing survives process death except the inbox rows.

`SessionV2.resume`/`interrupt` delegate straight to it (`session.ts:426-432`); `shell`, `skill`, `compact`, `wait` all return `OperationUnavailableError` (`session.ts:387-392, 417-424`) — V2 cannot do those yet.

## 3. The turn loop

### V2: one `llm.stream` per provider turn

`SessionRunner.run` (`runner/llm.ts:428-451`) is a two-level loop:

```
outer: while (shouldRun)                 // queue drain
  inner: while (needsContinuation)       // provider turns
    runTurn(sessionID, promotion, step)
```

Eligibility: `hasPending("steer")`, else `hasPending("queue")`; if neither and not `force`, return without any provider call (`runner/llm.ts:432-434`) — this is the "advisory wake drains eligible rows only" rule. `force` comes from `coordinator.run` (`run-coordinator.ts:77`), so an explicit `resume` always performs exactly one attempt even on an empty inbox.

`runTurnAttempt` (`runner/llm.ts:181-393`) per provider turn:
1. Location guard: session location must equal the runner's `Location`, else `Effect.interrupt` (`runner/llm.ts:188-189`).
2. `SessionContextEpoch.initialize/prepare` builds or reconciles the system baseline (`runner/llm.ts:191, 205-206`; `packages/core/src/session/context-epoch.ts:23-38, 40-78`) using `SystemContextRegistry.load() + SkillGuidance + ReferenceGuidance` in parallel (`runner/llm.ts:176-179`).
3. **Promotion at the safe boundary** (`runner/llm.ts:195-204`): cutoff = current latest durable seq; `steer` promotes all pending steers admitted at or before the cutoff (`SessionInput.promoteSteers`, `input.ts:245-266`); `queue` promotes exactly one queued row first, then any steers (`input.ts:268-288`, `runner/llm.ts:199-202`). Any promotion resets `currentStep = 1`, which is precisely "promoting new user input resets the agent's provider-turn allowance; a batch of steers resets it once".
4. History reload: `SessionHistory.entriesForRunner(db, sessionID, baselineSeq)` (`runner/llm.ts:208`; `packages/core/src/session/history.ts:90-99`) — rows after the latest `compaction` message, plus post-baseline system messages. Context is rebuilt from the projection every turn, never held in memory, with an incremental append cache keyed on session|agent|model|variant|baselineSeq and validated against the stable message prefix (`runner/llm.ts:211-235`).
5. Step limit: `isLastStep = agent.info.steps !== undefined && currentStep >= steps` disables tool materialization, pushes `MAX_STEPS_PROMPT` as an assistant message and sets `toolChoice: "none"` (`runner/llm.ts:209-210, 256-258`; text at `runner/max-steps.ts:1-16`). A late tool call at the last step is failed with "Tools are disabled after the maximum agent steps" (`runner/llm.ts:289-292`). Default: no limit (`steps` optional).
6. Compaction gate before the stream (`runner/llm.ts:260-261`) — see §5.
7. `llm.stream(request)` (`runner/llm.ts:277`) is consumed by `Stream.runForEach`; every event is durably published through `createLLMEventPublisher` under a **single-permit semaphore** so publishers' writes serialize (`runner/llm.ts:263, 273-276`). Tool calls become durable events first, then the local settlement is forked into a `FiberSet` (`runner/llm.ts:288-316`): calls run **eagerly and concurrently**, results re-enter history as `Tool.Success/Failed` events via `publish` (`publish-llm-event.ts:337-375`).
8. `Effect.uninterruptibleMask` tail (`runner/llm.ts:322-392`): awaits the stream exit, then `awaitToolFibers` = `raceFirst(FiberSet.join, FiberSet.awaitEmpty)` (`runner/llm.ts:149-150`) — i.e. wait until all tools settle or the set empties, but surface the first failure. Interrupted tool fibers are cleared and unsettled calls failed ("Tool execution interrupted"), a user permission decline halts the whole loop rather than feeding the model (`isUserDeclined`, `runner/llm.ts:152-158, 342-346`, comment matches V1 parity). `Step.Ended` records finish reason, tokens, an end snapshot and the file diff (`runner/llm.ts:361-382`).
9. Return `{ needsContinuation: !hasProviderError && needsContinuation, step }`, where `needsContinuation` was set by any non-provider-executed `tool-call` (`runner/llm.ts:288, 293`).

**Continuation decision** (`runner/llm.ts:438-450`): after the inner loop ends for lack of tool calls, it re-checks pending steers (`line 446`) — a steer admitted mid-turn forces another provider turn; then the outer loop checks pending `queue` (`line 448`) and promotes one queued input per boundary, re-evaluating before the next (`AGENTS.md` "V2 Session Core" preserved in code).

**Completion** therefore has exactly three causes: model finished without local tool calls and no new steers/queue; a provider error settled the turn; or the drain was interrupted. There is no separate "task complete" judgment. The advisory Executive that would make that judgment is **not wired into V2 at all** (§6).

Recovery transitions are signalled as a `TurnTransitionError` **defect** carrying a tagged union, caught one level up so the rebuilt request cannot re-enter the same recovery (`runner/llm.ts:160-174, 400-426`): `ContinueAfterCompaction` re-runs with the same step and no promotion; `ContinueAfterOverflowCompaction` re-runs through a path with `recoverOverflow` removed, and a second overflow there is a hard defect (`runner/llm.ts:405-406`).

### V1: `runLoop` + `SessionProcessor`

`runLoop` (`prompt.ts:1176-1627`) is `while(true)` over the durable history:
- Status `busy` each iteration (`prompt.ts:1196`); history via `MessageV2.filterCompactedEffect` (`prompt.ts:1200-1202`); `MessageV2.latest` extracts last user/assistant/finished/tasks (`prompt.ts:1210`).
- **Exit** when the last assistant has a terminal `finish` not in `["tool-calls","unknown"]`, no unresolved local tool parts, and it is a reply to the current last user (`prompt.ts:1220-1244`). `hasToolCalls` deliberately ignores `providerExecuted` parts and "orphaned interrupted" parts (`prompt.ts:135-139, 1221-1223`) — the latter is a `cleanup()`-marked abandoned tool_use block, so post-retry orphans don't keep the loop alive.
- Step 1 forks title generation (`prompt.ts:1247-1253`) and a summary (`prompt.ts:1407-1408`); steps > 1 fork advisory Executive observation (§6).
- `subtask` parts are handled by `handleSubtask` (`prompt.ts:297-491`), which is **not** a normal tool execution: it fabricates the assistant/tool parts itself, calls `TaskTool.execute` directly with `bypassAgentCheck`, and on `!task.command` posts a synthetic "Summarize the task tool output above and continue" user message (`prompt.ts:472-491`). `compaction` task parts run `compaction.process` (`prompt.ts:1274-1287`).
- Pre-turn overflow check on the last finished message's tokens (`prompt.ts:1289-1296`).
- `SessionReminders.apply` injects synthetic reminder parts (`prompt.ts:1308-1312`); step 1 appends memory reminders as a part on the user message (`prompt.ts:1314-1328`).
- Tools: `SessionTools.resolve` (`prompt.ts:1379-1395`) then `sys.frozenTools(prefixKey, live)` when `experimental.stable_prefix !== false` (`prompt.ts:1374, 1396`); system prompt computed under the same key (`prompt.ts:1464-1465`, `compute` at `prompt.ts:1412-1461`), plus `StructuredOutput` tool when `format.type === "json_schema"` (`prompt.ts:1398-1405`, tool at `prompt.ts:1851-1877`).
- `handle.process(...)` (`prompt.ts:1528-1542`) runs one stream; the outcome mapping decides the loop:
  - `structured` captured → break (`prompt.ts:1544-1549`).
  - finish terminal + no error: `content-filter` → error + break (`prompt.ts:1558-1565`); `finish === "length"` → `continueAfterLength` (`prompt.ts:1566-1568`); `json_schema` without output → `StructuredOutputError` + break (`prompt.ts:1569-1576`).
  - `result === "stop"` → break; `"compact"` → `compaction.create(auto:true, overflow:refused)` with the **double-refusal guard** `compactedAfterRefusal` (`prompt.ts:1181-1185, 1579-1614`) so a local engine that stays out of memory can't compact forever; a finished non-length step that made no tool call breaks instead of posting the continue nudge (`prompt.ts:1580-1592`).
- `compaction.prune` forked on exit (`prompt.ts:1624`), return `lastAssistant`.

`SessionProcessor.process` (`processor.ts:710-825`) is one provider attempt:
- Request-size estimate before streaming (`processor.ts:719-721`) as the overflow fallback when a provider reports no usage.
- `llm.stream(streamInput)` → `Stream.tap(handleEvent)` → `Stream.takeUntil(() => ctx.needsCompaction)` (`processor.ts:750-768`): an overflow detected mid-stream **stops consumption early**.
- Retry wrapper (`processor.ts:782-816`): before each retry, partial text/reasoning/tool parts of the failed attempt are **removed** and the tool map pruned (`processor.ts:730-744`), and `canRetry: () => !ctx.toolCallEmitted` (`processor.ts:786`, `processor.ts:365`) means a retry only ever happens while the attempt had no side effects — once a tool call was emitted the attempt is not repeated.
- `halt` (`processor.ts:678-708`) classifies: `ContextOverflowError` sets `needsCompaction` and deliberately does **not** publish an error event (recovery isn't a failure, `processor.ts:695-699`) unless `compaction.auto === false`; anything else sets `assistantMessage.error`, publishes `Session.Event.Error` and marks idle.
- `cleanup` (`processor.ts:618-676`) always runs: patches snapshots into `patch` parts, closes text/reasoning parts, gives pending tool calls 250 ms to settle (`processor.ts:650-654`), then marks leftovers `error: "Tool execution aborted"` with `metadata.interrupted = true` — the exact shape `isOrphanedInterruptedTool` looks for.
- Result: `"compact" | "stop" (blocked||error) | "continue"` (`processor.ts:821-823`).

Provider-error mapping to those classes happens in the LLM layer: `processor.ts:425-431` turns `classification === "context-overflow"` into `ContextOverflowError`, everything else into `APIError{isRetryable}`; the comment there is explicit that a plain `Error` would have ended sessions that compaction could fix. `finish === "length"` errors out any still-pending tool parts ("Not executed: the response was cut off...", `processor.ts:482-501`), matching the `lengthGuard` middleware in `llm/length-guard.ts:35-85` that **buffers `tool-input-end`/`tool-call` until the finish reason is known and drops them entirely on `length`** (provider-executed calls pass through). The `compat.finish_reason` remap is `length-guard.ts:25-33`, applied at `llm.ts:358`.

## 4. Interruption / steering / cancellation

**V1**: `POST /abort` (`handlers/session.ts:249-252`) → `SessionPrompt.cancel` → `SessionRunState.cancel` (`run-state.ts:77-86`), which first cancels background jobs transitively — the fixpoint loop over `metadata.sessionId`/`parentSessionId` (`run-state.ts:111-143`) kills a parent **and its subagents and their jobs** — then `Runner.cancel` interrupts the loop fiber and fails waiters with `Cancelled` (`runner.ts:169-186`). `Effect.onInterrupt` paths finalize the assistant message as `AbortError` (`prompt.ts:1346-1362, 1616-1619`) and, for subtasks, mark the part errored (`prompt.ts:402-421`). Interruption is immediate, mid-stream; there is no "finish the current step" notion.

**V2**: `SessionV2.interrupt` is `Effect.uninterruptible(execution.interrupt(...))` (`session.ts:430-432`) → coordinator interrupt of the process-local fiber. The turn tail then settles durably inside `uninterruptibleMask`: clears tool fibers, fails unsettled tools and the active assistant (`runner/llm.ts:340-355`). `failInterruptedTools` (`runner/llm.ts:127-147`) runs at the start of every drain and fails any leftover `pending`/`running` tool in the projection — this covers a crash between `Tool.Called` and settlement (the AGENTS.md note "crash recovery of eligible durable inbox rows"; also note that after a crash the **next drain** is the recovery point; there is no eager restart of in-flight provider work, matching "post-crash continuation recovery requires a separate explicit design").

Steering in V2 is the promotion machinery of §3 (durable, cutoff-bounded, step-allowance-resetting). Steering in V1 is implicit history reload plus one durable-adjacent trick: the runner state machine means a mid-turn prompt only takes effect at the loop's next iteration, and `MessageV2.latest`'s `parentID` comparison is what makes the loop notice a new last-user message.

## 5. Failure and recovery inventory

| Concern | V1 (live) | V2 |
| --- | --- | --- |
| Provider retry | `SessionRetry.policy` — exponential 2 s base ×2, 25 % jitter, 30 s cap without headers (`retry.ts:30-38, 95-98`), honours `retry-after-ms`/`retry-after`/HTTP-date (`retry.ts:62-93`), ≤5 attempts, message/regex + status classification (`retry.ts:45-56, 100-178`) | none (`llm.stream` failure is terminal; TODO at `runner/llm.ts:57`) |
| Retry safety | no retry once a tool call was emitted; partial attempt parts deleted (`processor.ts:730-744, 786`) | n/a |
| Rate-limit UX | Go free/account upsell actions (`retry.ts:121-166`) | none |
| Local engine memory refusal | one pre-compact retry, memory waiter, engine outage wait (`retry.ts:36-43, 216-226`; `processor.ts:787-805`) | n/a |
| Context overflow | provider error → `needsCompaction` → compaction task; silent recovery; double-refusal guard (`processor.ts:425-431, 686-699`; `prompt.ts:1593-1605`) | `isContextOverflowFailure` before assistant start → `compactAfterOverflow` once, else hard fail (`runner/llm.ts:281-285, 327-333, 400-426`) |
| Auto compaction | post-step token check `isOverflow` incl. request estimate fallback (`overflow.ts:15-31`, `processor.ts:549-560`); pre-turn check (`prompt.ts:1289-1296`); continuation handoff summary (`compaction.ts` V1 `Continuation`, keyed by prefix, `prompt.ts:1156-1174, 1274-1287`) | pre-turn size check vs `context - max(output, 20k buffer)`; keep last 8k tokens; summary+recent split (`core/session/compaction.ts:12-15, 141-178, 257-268`) |
| Output truncation | `lengthGuard` drops tool calls, ≤3 synthetic continue turns then `OutputLengthError` (`llm/length-guard.ts:41-83`; `prompt.ts:68-89, 1123-1154`) | none observed in V2 runner |
| Doom loop | identical consecutive call redirect at N=3, then `doom_loop` permission ask (`doom-loop.ts:9, 23-44, 61-74`; `session/tools.ts:157-175, 514-525, 701`) | none (TODO `runner/llm.ts:57`) |
| Prefix stability | frozen system prompt + tool snapshot per session|agent|model|history-root, `reconcile` keeps definitions and swaps executors, missing tools get an "unavailable" executor (`tool-freeze.ts:28-88`; `system.ts:115-123`) | `contextCaches` prefix cache only (`runner/llm.ts:116, 220-235`) |
| Content filter | `ContentFilterError` + break (`prompt.ts:1558-1565`) | none |
| Interrupted tool orphans | `metadata.interrupted` marker excluded from `hasToolCalls` (`prompt.ts:135-139`) | `failInterruptedTools` on next drain (`runner/llm.ts:127-147, 435`) |
| Revert | V1 `SessionRevert` service; `revert.cleanup` on every prompt/shell (`prompt.ts:1098, 500-502`) | core `SessionRevert.stage/clear/commit` snapshot-based, restores files after the boundary message and stores a `Revert.State` on the session (`core/session/revert.ts:58-120`; exposed at `core/src/session.ts:433-453`) |
| Message errors | `message-error.ts` is a 14-line stub: only `OutputLengthError`, `AuthError`, `Unknown` (`message-error.ts:4-12`); the real taxonomy lives in `SessionV1` | `SessionEvent.Step.Failed{error:{type:"unknown"}}` everywhere (`publish-llm-event.ts:199-211`) |

**Prompt/message-ID reuse**: V2 is explicitly idempotent — same message ID + identical session/prompt/delivery returns the original `Admitted`; any difference fails `PromptConflictError` (`session.ts:374-381`, `input.ts:191-214`). V1 has no such notion; `messageID` reuse just overwrites the row via `updateMessage`.

## 6. Executive: an idea with three bodies

- V1: `SessionExecutive.observe` is **advisory only**, forked at step > 1, never blocks or steers; logs `would:` decisions for later scoring (`executive.ts:10-14, 133-146`; call site `prompt.ts:1254-1264`). Disabled unless `config.experimental.executive.enabled` (`executive.ts:79-84`).
- Core: `@zero/v2/Executive` service with `decide` and an authority-bearing `Decision` (`core/executive/executive.ts:15-41`) is **never referenced outside its own tests** — `Executive` in `packages/core/src/session/runner/llm.ts` is only a dead import at line 16. `SessionRunnerModel.resolve(session, override?)` (`runner/model.ts:74-76`) exists to let "the executive pass a routing override", but the sole production call passes no override (`runner/llm.ts:207`).
- Net: model routing by outcome is designed, plumbed at the seam, and unimplemented.

## 7. Long-running work, background, placement

- `BackgroundJob` (`packages/zero/src/background/job.ts:17-35`) is a thin instance-scoped wrapper over `@zero-code/core/background-job` (registry with `start/extend/wait/waitForPromotion/promote/cancel`) — used by the task tool and shell jobs, and as the cancellation tree root (§4).
- V2 placement: `SessionRunner` is Location-scoped (`makeLocationNode`, `runner/llm.ts:459-477`) and only reached via `LocationServiceMap.get(session.location)` per drain (`execution/local.ts:20-21`); `SessionExecution` is global and Session-ID-keyed (`execution.ts:20-23`), exactly as AGENTS.md prescribes. Drains are process-local — `execution/local.ts:10` ("Future remote placement belongs here") and `SessionV2.resume`'s error type has no transport error. `session.location` mismatch → `Effect.interrupt`, not an error (`runner/llm.ts:188-189`).
- V1 has no placement concept: everything is `InstanceState`-keyed by directory.

## 8. Subagents

Spawned by the `task` tool (`packages/zero/src/tool/task.ts`), and by `subtask` parts in the loop (`prompt.ts:297-491`):
- Depth bound walks the `parentID` chain, `cfg.subagent_depth ?? 1` (`task.ts:107-120`).
- Child session inherits derived permission rules and additionally **denies `todowrite` and `task`** unless the subagent config grants them (`task.ts:150-183`) — so nesting is opt-in per agent.
- Context isolation: a fresh child `SessionID` (or `task_id` resume, `task.ts:139-149` — an invented id is an explicit error, `task.ts:142-149`); the child runs `ops.prompt`, i.e. the *same* loop code recursively (`task.ts:211-224`, wired at `prompt.ts:186-192`), returning only its last text part (`task.ts:236-239`, `EMPTY_REPLY` guard at `task.ts:64-65`). Attachments come back through the tool result (`prompt.ts:424-429`).
- Parallelism: foreground task blocks the parent's tool settlement; multiple `task` calls in one turn settle concurrently (AI SDK executes tools per-call). `background: true` requires `ZERO_EXPERIMENTAL_BACKGROUND_SUBAGENTS` (`task.ts:100-105`), starts a `BackgroundJob` keyed by child session id, returns immediately, and on completion **injects a synthetic user prompt into the parent** (`inject`, `task.ts:242-269`) — the loop picks it up as a steer. `extend` reuses a running job for a follow-up to the same `task_id` (`task.ts:282-297`).
- V2 has no `task` tool yet (`core/src/tool/builtins.ts:26-29` lists "task, LSP, repo_clone, repo_overview, plan_exit" as unported).

## 9. One turn, end to end (V2)

1. `POST /session/:id/v2/prompt {text}` → flag check → `SessionV2.prompt(resume:false)` → durable `PromptAdmitted` → `session_input` row (`admitted_seq=N, promoted_seq=NULL`) → `coordinator.run(id)` → `drain(force=true)`.
2. `hasPending("steer")` true → `failInterruptedTools` → `runTurn(id,"steer",1)`.
3. Location matches → `SessionContextEpoch.initialize/prepare` → cutoff=`latestSequence` → `promoteSteers` publishes `Prompted` → projector fills `promoted_seq=N` and writes the user message → `promoted=1` → `currentStep=1`.
4. Model resolved from the Location catalog with credential; system baseline + skill/reference guidance composed; history loaded post-compaction; request built with affinity headers and `promptCacheKey`.
5. Size check passes; start snapshot captured; `llm.stream(request)` starts. `text-delta` events published as `Text.Delta`; a `tool-call` is published as `Tool.Called` **before** `toolMaterialization.settle` is forked; the tool authorizes, executes, and its result is published as `Tool.Success` (with managed `outputPaths`) under the publisher's mutex.
6. `Step.Ended` with finish/tokens/end-snapshot/file diff. `needsContinuation=true` → next turn promotes steers again → reloads projection (cache appends only new messages) → second `llm.stream`.
7. Model answers with text only, `finish: stop`, no pending steer: inner loop ends; outer checks queue, finds none; drain returns; `settle` finds `pendingWake=false` and deletes the entry; the blocked HTTP request finally returns `{messageID, elapsedMs, messages}`.

## 10. Strengths

- **V2's durable admission is genuinely well-designed**: monotonic `admitted_seq`/`promoted_seq` with unique indexes (`sql.ts:140-164`), idempotent admit with equivalence-checked reuse, a `LifecycleConflict` defect → typed `PromptConflictError` translation at the boundary, and lazy promotion of historical rows.
- **Explicit step/turn boundaries**: one `llm.stream` per provider turn, history always reloaded from the projection before the next (`runner/llm.ts:78-79, 208`), promotion only at a provider-turn boundary.
- **Interruption is settleable, not just killed**: `uninterruptibleMask` + `failUnsettledTools` + `failInterruptedTools` means an interrupted or crashed turn cannot leave the projection with a permanently `running` tool.
- **V1's retry discipline is unusually careful**: side-effect-aware `canRetry`, attempt-part deletion, silent overflow recovery, no-compaction-forever guard, header-aware backoff, and an engine memory/outage rung on the ladder.
- **Prefix-cache engineering** (frozen system + tool snapshot, incremental context caches in both paths) is real, documented thinking, not accidental.
- Coordinator semantics match the stated contract exactly (join on resume, coalesce wakes, no-op idle interrupt, per-session serialization with cross-session concurrency).

## 11. Weaknesses / risks

1. **The live path is the 1920-line monolith.** Every fix (steering semantics, interruption settlement, retry bounds) exists twice, and only the older copy ships by default. `prompt.ts` mixes HTTP-facing schemas, MCP resource normalization, template/`$ARGUMENTS` parsing, title generation, and loop control in one closure.
2. **V1 steering is implicit and lossy-by-design**: a second prompt returns the *first* loop's final assistant (`ensureRunning` join), the new message is picked up by `lastUser` comparison, and there is no durable notion of delivery mode — no queue, no "promote one at a time", no step-allowance reset semantics. The documented delivery vocabulary only exists in the disabled path.
3. **V2 has no provider retry, no doom-loop guard, no output-length continuation, no subagent/task tool, no MCP/plugin tools, no structured output, and `shell`/`skill`/`compact`/`wait` return `OperationUnavailableError`.** The in-file TODO list (`runner/llm.ts:45-93`) is honest but long.
4. **V2 turn-level control flow rides on defects** (`TurnTransitionError`, `runner/llm.ts:166-174`), and a second overflow after overflow-compaction is `Effect.die` (`runner/llm.ts:405-406`) — a defect with no typed error channel, so a legitimate "still too big" condition surfaces as a crash-level failure to the caller rather than a typed `RunError`. V1 made the same case typed (`ContextOverflowError` on the message, `prompt.ts:1593-1605`).
5. **V1 loop exit heuristic is fragile**: it hinges on `finish` ∉ {tool-calls, unknown} **and** no local tool parts **and** `parentID === lastUser.id` (`prompt.ts:1225-1230`), with a logged "orphaned interrupted tool" escape hatch (`prompt.ts:1231-1241`) — a provider that reports `stop` with a tool call is papered over by the `hasToolCalls` re-check. V2 replaced this with a positive signal (`needsContinuation` from observed tool calls), which is the better design.
6. **No durable execution status.** V1 status is in-memory per-instance (`SessionStatus`), V2 has no status at all ("Replace local ownership...", `runner/llm.ts:53-55`). A crash mid-drain leaves the projection repaired only lazily, on the next wake/resume for that Session; nothing restarts eligible work by itself.
7. **Dead / vestigial code found**: `core/src/executive/executive.ts` service (unwired outside tests) + the unused `Executive` import at `core/src/session/runner/llm.ts:16`; the never-passed `override` parameter of `SessionRunnerModel.resolve`; `SessionV2.compact/wait/shell/skill` stubs; `packages/zero/src/session/message-error.ts` (14 lines, superseded by `SessionV1` errors); `prompt.ts`'s `input.tools` (`@deprecated`, `prompt.ts:1791-1794`); the catch-defect re-read in `admit` (`input.ts:77-79`) partly duplicates the caller's conflict check (`session.ts:374-381`).
8. **`promptV2` blocks an HTTP request for the entire session drain** and returns the full projected context (`handlers/session.ts:343-355`) — unusable for long sessions; there is no async/streaming V2 surface yet, so V2 cannot replace the live path as-is.
9. Two parallel context-conversion stacks (`MessageV2.toModelMessagesEffect` in V1, `to-llm-message.ts` in V2) with different attachment/materialization gaps (V2 leaves TODOs for remote/managed URIs at `to-llm-message.ts:38-41`).

---

## Part B — Tools, editing quality, repository intelligence (source audit)

Research-only audit, 2026-10-13. All claims from reading implementation; `file:line` refs are `packages/...` relative.

## 1. Tool registry, schemas, dispatch, permissions

**Registry** (`zero/src/tool/registry.ts`)
- `ToolRegistry.Service` builds `builtin` + `custom` tool lists per-instance via `InstanceState` (registry.ts:137-280). Builtins assembled at registry.ts:250-278: invalid, memory_correct, question (client-gated, :223), bash, shell_output, shell_kill, read, glob, grep, edit, write, task, webfetch, todowrite, websearch, skill, apply_patch, browser (only when `BrowserBridge.available()`, :130-131), computer (macOS non-CLI only, :132), `execute` (code-mode, flag-gated :134-135,:247), lsp (flag :273), plan (flag+cli :274).
- Model-conditional edit strategy: `apply_patch` is shown **only** when `modelID.includes("gpt-") && !oss && !gpt-4` and then `edit`+`write` are hidden (registry.ts:331-334). All other models get edit/write instead.
- `coding-minimal` tool profile exposes only `bash,read,edit,write` (registry.ts:82-83); truncate hints and the bash prompt rewrite themselves for it (truncate.ts:130-137; tool/shell/prompt.ts:306-312).
- Custom tools: files `{tool,tools}/*.{js,ts}` in config dirs are dynamically imported and validated structurally (registry.ts:199-213); plugin tools go through `fromPlugin` which keeps Zod as a boxed compatibility layer and converts to JSON Schema once (:141-197, :408-455).
- Per-tool description/schema is mutable by plugins via `plugin.trigger("tool.definition", …)` before sending (registry.ts:352-356). Task's description is appended with the live subagent catalog filtered by `task` permission (registry.ts:291-305); `execute` gets the code-mode tool catalog appended (:307-316,:339-342).

**Tool definition contract** (`zero/src/tool/tool.ts`)
- Params are Effect `Schema` decoders; decode closure compiled once per tool init (tool.ts:108-111). Schema failure → `InvalidArgumentsError` whose `message` getter is model-facing: "…called with invalid arguments… Please rewrite the input so it satisfies the expected schema" (tool.ts:24-34).
- Every result without an explicit `metadata.truncated` is auto-truncated via the Truncate service (tool.ts:131-144). All tool failures become defects (`Effect.orDie`, :145) and surface as error tool-results at the AI SDK boundary — error vs empty stays distinct.

**JSON Schema for the model** (`zero/src/tool/json-schema.ts`)
- Effect Schema → 2020-12 doc, then normalized for LLMs: strip `type:"null"` branches for optional props (:51-54), collapse single-member `anyOf` (:72-75), flatten non-conflicting `allOf` (:78-81), bound `integer` with MIN/MAX_SAFE_INTEGER (:83-85), inline `#/$defs` refs then drop defs (:121-150), WeakMap cache (:6).
- Per-provider sanitizing afterwards: `ProviderTransform.schema` (zero/src/provider/transform.ts:1580+) — OpenAI/Azure `sanitizeOpenAISchema`, Moonshot/Kimi `$ref`-sibling stripping and tuple-items collapse.
- Session wraps registry defs as AI SDK tools (zero/src/session/tools.ts:139-213): each call runs `LLMRepair.coerceArguments` (schema-driven type coercion, session/llm/repair.ts), `DoomLoop.detect` (below), plugin `tool.execute.before/after` hooks (tools.ts:174-196), and appends `ShellJobs.notices` for finished background jobs (tools.ts:187).
- Native route path: `packages/llm/src/tool-runtime.ts` is a narrow one-call typed dispatcher — decode params → execute → encode success; `ToolFailure` → `tool-error` + error `tool-result` so the model self-corrects (tool-runtime.ts:23-76). Used by `session/llm/native-runtime.ts:123`. It does not own the loop.

**Permissions** (`zero/src/permission/`)
- Wildcard ruleset, `findLast`-wins (permission/index.ts:28-38). `ask()` evaluates each pattern against ruleset + session-approved set; deny → error; ask → Deferred published as event; reject cascades to all pending requests of the session (:67-167); "always" appends allow rules (:145-151).
- Tools ask with named permissions `read/edit/grep/glob/bash/lsp/skill/doom_loop/external_directory/memory_correct` and `always: ["*"]` patterns (read.ts:256-261, edit.ts:146-154, grep.ts:39-48, shell.ts:264-292…).
- Bash approval is command-aware: commands are parsed with **tree-sitter bash/powershell WASM** (shell.ts:312-337, :736-745), out-of-project file arguments raise `external_directory` asks (:265-281), and the "always allow" suggestion is a semantic prefix via an LLM-generated command-arity table `BashArity.prefix` (shell.ts:431; permission/arity.ts header comment + `ARITY` map — e.g. `git checkout` = 2 tokens).
- `Permission.disabled` maps edit/write/apply_patch → `edit` and MCP resource tools → `read` for whole-tool deny (permission/index.ts:204-219).
- External paths: `external-directory.ts` asks `external_directory` for anything outside `containsPath(directory|worktree)`, with a realpath-checked carve-out for the memory root when `memoryRead` is set (external-directory.ts:28-55). Non-git projects set `worktree="/"` and the worktree check is skipped to keep the boundary honest (project/instance-context.ts:17-24).

**Guardrails around dispatch**
- Doom-loop: the 3rd identical consecutive call (same tool+args *and same result*, so polling isn't punished) is answered in place with a redirect message; later repeats raise a `doom_loop` permission (session/doom-loop.ts:7-76, applied tools.ts:154-173). Adapted from oh-my-pi.
- Hallucination repair: `session/llm/repair.ts` carries an alias table (`read_file→read`, `rg→grep`, `str_replace_editor→edit`, `todo→todowrite`, … repair.ts:18-53), provider prefixes (`functions.`, `default_api.`, …:15), `repairJson`, `alignKeys`, and `coerceArguments`; undeclared MCP/deferred names are re-routed onto the `tool_exec` dispatcher.
- Tool-search indirection: when the MCP catalog is crowded (≥ `toolSearchMinTools` and declarations ≥ `TOOL_SEARCH_AUTO_SHARE` of context budget), MCP tools leave the declared list and are reached via `tool_search`/`tool_exec` meta-tools; six rare built-ins (`question, shell_output, shell_kill, memory_correct, skill, engine_status`) are deferred with them (`DEFERRED_BUILTINS`, tools.ts:41-46, :460-491; comment at :41-45 measures built-in declarations ≈ 2,100 of ~6,000 tokens). Flags: `ZERO_EXPERIMENTAL_TOOL_SEARCH` auto, `ZERO_DEFER_RARE_TOOLS` default true, `ZERO_TOOL_SEARCH_MIN` (effect/runtime-flags.ts:73-90).

## 2. File tools

**read** (tool/read.ts, read.txt)
- Caps: 2000 lines default (read.ts:13), 2000-char line cap with a suffix telling the model to `cut -c2001-` on that line (:14-15), 50 KB total byte cap with a streaming `ReadStop` (:16, :137-180). Truncation notices carry the exact next `offset=` (:345-351). Line-numbered `<content>` envelope with "End of file" close-out (:339-352).
- Directories: sorted listing, `/` suffixes, offset paging notice (:101-115, :265-299). Missing file: "Did you mean one of these?" from substring dir scans (:76-99). Binary detection: extension list + NUL + >30% non-printable in a 4 KB sample (:182-227). Images (jpeg/png/gif/webp) and PDFs return a one-line message plus base64 attachment (:302-326).
- Reads lazily attach nearest `AGENTS.md`-style files as a trailing `<system-reminder>` (via `Instruction.resolve`, :301,:356-358; session/instruction.ts:179-221 attaches once per message) and warm LSP in the background (:117-120).

**edit** (tool/edit.ts, edit.txt, edit/near-miss.ts) — search/replace, not diff/patch
- Params: `filePath/oldString/newString/replaceAll` (edit.ts:48-57). CRLF detection + conversion both ways (:23-34,:130-132); BOM preserved (:98-99,:127-135); per-path `Semaphore` write lock (:36-46 — module-global `Map`, never GC'd).
- Matching cascade of 9 `Replacer` generators tried in strict→loose order (:695-726): exact, line-trimmed, block-anchor (first/last line anchors + Levenshtein mid-line similarity ≥0.65, :289-426), whitespace-normalized, indentation-flexible, escape-normalized, trimmed-boundary, context-aware (≥50% trimmed-line match), multi-occurrence. Once a strict matcher found something, looser matchers are refused (:723-725). Duplicate exact matches → "Found multiple matches… provide more context" (:740, edit.txt:9).
- Safety gate: if the fuzzy matched span is much larger than `oldString` (≥ oldLines+3 or ×2 lines, or ≥4× chars) the edit is *refused*, not applied (:710-714, :743-749).
- Failure guidance: on total miss, `findNearMiss` scans bounded windows (all windows if ≤5000, else anchor+stride sampling; min similarity 0.6; result ≤40 lines/4000 chars) and the error reports "closest match is N% similar, at lines X-Y: …" so the model corrects in-turn rather than guessing again (edit/near-miss.ts:45-53,189-217; edit.ts:728-738; edit.txt:8).
- Post-write loop: formatters run then re-read (`Bom.syncFile`, :156-158), diff computed for UI/permission, then `lsp.touchFile` + diagnostics appended as "LSP errors detected in this file, please fix" (:197-202). Empty `oldString` only creates new files; on existing files it's rejected with a pointer to `write` (:91-97, :687-691).
- edit.txt (10 lines): mandates read-before-edit (**not enforced in code**), line-prefix explanation, failure semantics, replaceAll.

**apply_patch** (tool/apply_patch.ts + zero/src/patch/index.ts) — Codex-style
- Format: `*** Begin Patch` envelope with `*** Add File:` / `*** Delete File:` / `*** Update File:` (+ optional `*** Move to:`) sections and `+/-` lines (patch/index.ts:76-87,:192-226; apply_patch.txt teaches exactly this with one example). Multi-file hunks in one call, one combined `edit` permission ask carrying per-file diffs (apply_patch.ts:199-221). Empty patch rejected (:47-53); duplicate Update sections for one file build on the prior section instead of clobbering (:107-121); same format+LSP diagnostics loop as edit (:258-299). Consumers: only the registry, gated to gpt-5-family models (registry.ts:331-334).

**write** (tool/write.ts, write.txt)
- Whole-file overwrite with unified diff shown at permission time (:53-62); after write, diagnostics for the file plus up to 5 other files are surfaced (MAX_PROJECT_DIAGNOSTICS_FILES=5, :18, :78-90). Read-before-overwrite is description-only policy (write.txt:3).

**glob / grep** (tool/glob.ts, tool/grep.ts) — backed by a **bundled ripgrep binary**
- Core spawns the bundled `rg` and streams NDJSON with `Stream.take(limit+1)` for exact truncation detection (core/src/ripgrep.ts:97-131).
- grep: limit 100 matches (grep.ts:71-77,:89), each matched line hard-capped at 500 chars — explicitly "exempt from the generic output cap" (:105-106), "(more matches available)"/"(Results truncated…)" hints (:96,:110-113); mistyped path returns "Path not found" instead of silently searching the parent (:63-70); empty result = `"No files found"`, not an error (:30-34). No pagination — narrower pattern is the only recourse (bash `rg` is steered there by grep.txt).
- glob: limit 100 (glob.ts:50), hidden dirs included on purpose (:51-52), `path` must be a directory (:41-43).

## 3. Shell / command execution

(tool/shell.ts + shell/prompt.ts + shell/jobs.ts + shell-jobs.ts; exposed id stays `bash` for compat — shell/id.ts:14-16)
- **No persistent shell.** Each call spawns a fresh process via `ChildProcess.make(command, {shell, cwd, env, stdin:"ignore", detached:!win32})` (shell.ts:294-311); the description states cd/export/venv do not carry over (shell/prompt.ts:270). `workdir` param replaces `cd` (shell.ts:728-730; prompt.ts:123-129).
- Timeouts: default 2 min (`bashDefaultTimeoutMs ?? 2*60*1000`, shell.ts:349; `ZERO_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS`), per-call `timeout` param (shell/prompt.ts:27). Race of exit/abort/timeout with `kill({forceKillAfter:"3 seconds"})` (shell.ts:566-589); signal-kills are parsed out of the exit failure and reported (shell.ts:568-575,609).
- Process lifetime: each command in its own process group; groups that leave detached children behind are recorded and SIGTERMed at server shutdown, pruned when dead (shell.ts:350-369,591-597); a `<shell_metadata>` note tells the model background `&` output is uncaptured and to use `run_in_background` (:611-614).
- Output capture: merged stdout+stderr streamed; ring buffer keeps the tail 2×maxBytes, overflow spills to a Truncate file with an append sink, live tail previews UI via `ctx.metadata` (preview ≤30 000 chars, shell.ts:28,221-224,460-555). Final output = tail(lines≤maxLines, bytes≤maxBytes) with "...output truncated… Full output saved to: file" prefix (:615-627); empty output is literally `"(no output)"` (:623); timeout/abort/nonzero-exit all appended in `<shell_metadata>` (:602-631).
- Background jobs: `run_in_background` forks the process into a job scope outliving the call (shell.ts:644-705): caps `MAX_RUNNING=8`, 1 MB rolling buffer/job, `MAX_FINISHED=16` (shell/jobs.ts:5-11); `shell_output` returns only bytes since last read + status with optional `wait_ms≤30000`; `shell_kill` kills the tree (shell-jobs.ts:26-77); completion is *announced* into the next tool result rather than polled (`ShellJobs.notices`, tools.ts:187, jobs.ts `announced` field). Busy-cap message tells the model to stop one first (shell.ts:651-657).
- The description is a large generated prompt per shell family (bash/pwsh/powershell/cmd × platform): quoting examples, verification steps, "prefer dedicated tools", chaining rules, PR/gh etiquette, and embedded live limits/timeout (shell/prompt.ts:89-230; template shell/shell.txt). Heavy but genuinely small-model-friendly (good/bad examples).

## 4. Repository intelligence

**repomap** (zero/src/repomap/repomap.ts) — aider-style, tree-sitter + PageRank
- WASM tree-sitter grammars for py/ts/tsx/js/jsx with hand-written `@def/@range/@ref` queries (:18-66); extract defs + per-file reference sets (:112-149); file-level PageRank (damping 0.85, 12 iterations) over cross-file name-reference edges, with an 8× seed boost for files defining identifiers that appear verbatim in the last user message (:151-187, :219-224). Output: ranked `file` + up to 8 defs with line ranges and `# Nx` refs, token-budgeted, "line numbers may drift" header (:226-245). Limits: 2000 files, 512 KB/file, skip dist/node_modules etc. (:68-81), bails under 12 candidates (:204).
- Wired behind `ZERO_EXPERIMENTAL_REPO_MAP` (effect/runtime-flags.ts:79), built once per session-prefix with 45 s timeout and graceful skip (session/prompt.ts:1420-1448), injected into the system prefix alongside instructions/memory/skills (:1453-1460).

**LSP** (zero/src/lsp/) + tool (tool/lsp.ts)
- Service spawns configured servers (bundled catalog + zero.json overrides, lsp/lsp.ts:149-195), lazily per root with spawn dedup (:217+); `touchFile`/`diagnostics` feed the *automatic* post-edit error reporting in edit/write/apply_patch — this is the strongest quality loop in the harness and does not depend on the model invoking anything.
- The `lsp` tool (off by default; `ZERO_EXPERIMENTAL_LSP_TOOL`) exposes 9 operations: goToDefinition, findReferences, hover, documentSymbol, workspaceSymbol, goToImplementation, prepareCallHierarchy, incomingCalls, outgoingCalls (tool/lsp.ts:11-21); output is raw `JSON.stringify(result, null, 2)` (:108) — big for call-hierarchy JSONs, saved only by generic truncation.

**project + instructions** (zero/src/project/, session/instruction.ts)
- `Project.fromDirectory` resolves directory→worktree(git root)/vcs and persists identity (project/project.ts:90-91, :217-310); `containsPath` defines the permission boundary (instance-context.ts:17-24).
- Instruction discovery: first project-level `AGENTS.md`/`CLAUDE.md`/`CONTEXT.md` (deprecated) wins — no ancestor stacking (instruction.ts:64-68, :122-133); global fallback `~/.config/…/AGENTS.md` else `~/.claude/CLAUDE.md` (:60-63, first match only); config `instructions` entries support globs and http(s) URLs fetched with 5 s timeout (:135-150, :158-168, :95-103). Nested instruction files attach lazily on `read`, once per assistant message, walked upward from the file being read (:179-221).

**git + snapshot** (zero/src/git/index.ts, zero/src/snapshot/index.ts)
- `git/` is a typed wrapper (status/diff/numstat/patch/show/applyPatch) with byte-capped output (`Patch.truncated`, git/index.ts:48-90).
- `Snapshot` = **checkpoint/revert engine**: a shadow git dir at `data/snapshot/<projectID>/<hash(worktree)>` with `--git-dir/--work-tree` against the real worktree (snapshot/index.ts:71-75). Object DB seeded from the repo via `objects/info/alternates` + copied index so large repos don't rehash (seed(): :198-233). `track()` = add(candidates only, ignores honored, untracked >2 MB excluded) + `write-tree` (:235-347); the session processor tracks *before* the stream starts and at every step boundary, storing the hash on `step-start` parts (session/processor.ts:119-121, :433-445). `revert(patches)` restores per-file from stored hashes, batched in path-clash-safe runs of ≤100 with `ls-tree` existence checks and delete-if-absent fallback (:408-524); `restore()` is full read-tree+checkout-index (:382-406); consumed by session/revert.ts for rewind. `diffFull` builds whole-file before/after patches with `git cat-file --batch` fast path and per-file `git show` fallback (:546-758). `gc --prune=7.days` hourly (:23, :300-316, :761-766).

**memory** (zero/src/memory/, tool/memory-correct.ts)
- Per-project store (status.md / lessons.json / map.json / state.json + global lessons) at `~/.local/share/zero/memory/<projectKey>` (memory/index.ts:62-87; store.ts). Background file scan via ripgrep glob capped 4000 files, 10-min rescan throttle with touched-file invalidation (:179-199; health/errors recorded in state.json :95-107).
- Injection: `block()` renders the `<project_memory>` system section with hard token budgets **status 200 / lessons 150 / map 150 / total 500 tokens** (render.ts:4), drops lessons whose text is already in retained context (render.ts:22-30), top-5 by hit count, and appends "Full records (read on demand): <root>" (render.ts:41). Every tool call is observed for lesson learning and may have a `note` appended to its output (session/tools.ts:70-97 `observed()`).
- `memory_correct` (tool/memory-correct.ts): one-shot, permission-gated, replacement ≤240 chars; failure text says "do not retry memory maintenance" (:23). Compaction/handoff format constants (Now/Next/Blocked) at memory/index.ts:25-28.

## 5. MCP (zero/src/mcp/)
- Transports: remote tries **StreamableHTTP then SSE** with OAuth pending-transport flow (mcp/index.ts:269-337, auth.ts/oauth-*.ts); local stdio (:350-363); stdio pid surfaced (:544).
- Namespacing: `sanitize(server)_sanitize(tool)` (catalog.ts:117-119); keys are `%`-escaped `server%3Aname` for resources to stay unambiguous (:104-108). Tool defs get `additionalProperties:false` forced (catalog.ts:43-48); `isError` → thrown with concatenated text content (:68-74); `structuredContent` → JSON text (:75-80); 30 s default timeout with `resetTimeoutOnProgress` + no-op onprogress to earn progress tokens (:11, :61-66).
- Listing is cursor-paginated with duplicate-cursor and 1000-page guards (catalog.ts:18-36); tolerant `tools/list` retry when a server's `outputSchema` breaks SDK validation (:14-16, :151-173).
- Per-call permission ask uses the full sanitized key as the permission name (tools.ts:540, executeMcp); MCP text output goes through the same Truncate service (tools.ts:598); images/resources become attachments, blobs ≤10 MB and only pdf/gif/jpeg/png/webp, otherwise an explanatory placeholder line (tools.ts:36-43, :562-590).
- Resources are first-class via `list_mcp_resources` / `list_mcp_resource_templates` / `read_mcp_resource` only when some connected server advertises resources, gated by `mcp:<server>:*` read permission (tools.ts:34-38, :215-455).

## 6. Skills and plugins
- Skills: `SKILL.md` scan from `~/.claude/skills`, `~/.agents/skills`, project-up `.claude`/`.agents`, config dirs `{skill,skills}/**`, configured `skills.paths`, and pulled `skills.urls` (skill/index.ts:21-25, :173-233; discovery.ts fetches). Built-in `customize-zero` registered first so disk can override (:276-283). System prompt carries name+description+location only (`Skill.fmt` verbose XML, :321-346; injected session/system.ts:194). The `skill` tool loads the full body + a *sampled* ≤10-file listing with base-dir guidance (tool/skill.ts:36-61; skill.txt). Missing name error lists available skills (skill/index.ts:73-80).
- Plugins (`zero/src/plugin/`, `@zero-code/plugin`): expose `tool` definitions (zod args), `auth`/provider modules, and hooks `tool.definition`, `tool.execute.before`, `tool.execute.after`, `shell.env` (registry.ts:352; tools.ts:174-196; shell.ts:438-448). Plugin `ask` is bridged Promise-side via EffectBridge with host context preserved (registry.ts:160-169). Drop-in files in config `tool/*.ts` are the filesystem extension point (registry.ts:199-213).

## 7. Output shaping (cross-cutting)
- One truncation service: defaults **2000 lines / 50 KB** (truncate.ts:15-16), config-overridable `tool_output.max_lines/max_bytes` (:76-84); overflow writes the full text to `data/tool-output/<ToolID>` and returns preview + `...N lines|bytes truncated...` + a hint (:86-147). Hint branches by capability: task-capable agents are told to delegate to an explore subagent, others to Grep/Read offset, coding-minimal to `rg` (:129-137). 7-day retention, hourly cleanup loop (:13, :49-67, :149-154).
- Distinctions preserved: shell `"(no output)"`, grep/glob "No files found", read errors with "Did you mean", edit failures with near-miss excerpts, invalid-args with schema-rewrite request, `invalid` tool receiving provider-side parse failures (tool/invalid.ts:12-19). No tool reports an error as empty output; empty is never silent.
- Small-model support, candidly: long example-rich descriptions (shell prompt, task.txt usage notes 1-7, todowrite when/not-use examples), alias/repair/coercion layer, doom-loop redirect text ("Do not repeat this call. Use the previous result, change the arguments…", doom-loop.ts:66-72), truncate hints that name the *next* action. Costs: ~6k tokens of builtin declarations (measured in tools.ts:41-45), websearch/task keep large nested schemas, PowerShell/cmd branches re-render the whole bash prompt per shell family.

## 8. codemode (packages/codemode + tool/code-mode.ts)
- Standalone Effect library: JS-subset **acorn** parse (interpreter/runtime.ts:1) after TypeScript `transpileModule` (:3), a 3465-line hand-written tree-walking interpreter, hand-rolled stdlib sandbox (src/stdlib/*: string/json/promise/…), copy-in/copy-out JSON-only data boundary, `SafeObject` prototype-blocked members, and a typed diagnostic taxonomy `ParseError…ExecutionFailure` (codemode.ts:62-75). Optional per-run limits `timeoutMs/maxToolCalls/maxOutputBytes` (codemode.ts:10-17; enforced interpreter/runtime.ts:3351, :3378-3386, :3405-3438; catalog budget for instructions tool-runtime.ts:495-532). Package AGENTS.md keeps it host-agnostic with no internal permission policy.
- Zero's `execute` tool (tool/code-mode.ts): **live only behind `ZERO_EXPERIMENTAL_CODE_MODE`** (runtime-flags.ts:58; registry.ts:134-135). It composes *only MCP tools*, regrouped `server.tool` from sanitized keys (code-mode.ts:39-56, :120-132); the catalog renders into the tool description (registry.ts:307-316); each sandbox call re-asks permission under the MCP key (code-mode.ts:147) and projects MCP results incl. attachments (:75-116). When code-mode is on, the MCP tool-search path is skipped entirely (tools.ts:462). Notably `CodeMode.make(...)` here passes **no limits** (code-mode.ts:239) — no wall-clock or tool-call cap in Zero's wiring; only the outer Truncate 50 KB and the abort signal bound it.
- `codemode/src/openapi/` generates tools from OpenAPI specs, used by other hosts, not by the session path.

## 9. Candid assessment

**Strongest areas**
1. *Edit pipeline*: exact match first, 8 graded fallbacks that can only widen once a strict matcher proved uniqueness, disproportionate-match refusal, multi-match refusal, near-miss excerpt on failure, CRLF/BOM preservation, per-file locking, formatter + automatic LSP diagnostics feedback. Error strings are written to be acted on, not just logged.
2. *Output budget*: one coherent Truncate service with spillover files, capability-aware next-step advice; shell ring-buffer + file spill; grep's own 100×500-char caps; memory block under 500 tokens; repomap token-budgeted.
3. *Recovery machinery*: doom-loop redirect, tool-name aliases + JSON repair + argument coercion, InvalidArgumentsError rewrite loop, invalid tool. This is unusually good for weak models.
4. *Snapshots*: real per-step shadow-git checkpoints with alternates seeding and per-file revert; genuinely durable undo.
5. *Background execution*: shell jobs with incremental reads, caps, and push-style completion notices instead of model polling.

**Weakest / risky areas**
1. *Edit-corruption risk concentrates in the fuzzy tier*: BlockAnchor/ContextAware can replace a 65%-similar region when `oldString` is a hallucinated variant; the disproportionate-span guard bounds size, not semantics, and LSP diagnostics won't catch behavior-preserving-looking misfires. Fuzzy tiers also silently re-indent by design (LineTrimmed/IndentationFlexible yield the *file's* span).
2. *Read-before-edit is unenforced convention* (edit.txt:4, write.txt:3), and read's 2000-char line cap means an edit built from truncated lines can miss — the advice to use `cut` also conflicts with "do not use bash for file ops".
3. *Single-shot edit for non-GPT models*: one search/replace per call; multi-file/multi-hunk is reserved for the apply_patch path (registry.ts:331-334), so most models pay a round-trip per edit.
4. *Four overlapping "reach the right tool" layers* (registry visibility filtering → LLMRepair aliases/prefix-stripping → tool_search/tool_exec deferral → code-mode `execute`) plus per-provider schema sanitizers. Each is defensible alone; together they are hard to reason about and the effective tool surface varies by flag, model id, and MCP census.
5. *Schema cost & gating*: ~20 builtins with long descriptions; LSP intelligence (the best repo signal after grep) is experimental-off; repomap experimental-off; codemode experimental-off and, when on, uncapped internally and MCP-only.
6. Smaller: grep has no pagination; `locks` Map in edit.ts grows forever; edit.ts throws plain Errors that ride `Effect.orDie` (message survives to the model but typed-error matching upstream is lost); write can dump up to 6 files' diagnostics at once; `websearch` schema is provider-flag soup for weak models.
7. *Unnecessarily complicated*: the 9-generator replacer cascade with early-exit ordering rules (:706-726) is clever but underdocumented at the behavior level; snapshot `revert` batching with clash detection (:445-521) and `cat-file --batch` byte-precise parsing (snapshot/index.ts:604-677) carry a lot of failure-mode surface for a UI-undo feature; the code-mode interpreter (3.5k lines) duplicates a runtime before there is a non-experimental consumer.

---

## Part C — Context engineering, persistence, providers, observability (source audit)

Research-only audit of the Zero Code coding-agent harness. All claims verified against source
(2026-10-13). Two session runtimes coexist: the V1 loop in `packages/zero/src/session/` (the one
shipping, driven by `SessionPrompt.runLoop`) and the V2 core in `packages/core/src/session/`
(durable `session_input`, `SessionContextEpoch`). Both are covered; V1 is the default path.

## 1. System / context construction

**Model-specific base prompt.** `packages/zero/src/session/system.ts:35-59` (`provider()`) picks one
`.txt` per model family by substring match on `model.api.id`: `claude→anthropic.txt`,
`gemini-→gemini.txt`, `gpt-6→gpt-astra.txt`, `codex→codex.txt`, `gpt→gpt.txt`,
`gpt-4|o1|o3→beast.txt`, `muse→meta.txt` (with `{{MODEL_NAME}}` substitution),
`trinity→trinity.txt`, `kimi→kimi.txt`, else `default.txt`. `compact=true` returns only
`coding-minimal.txt` (12 lines, system.ts:36). Prompts vary sharply: `default.txt` (96 lines) leads
with verbosity rules ("fewer than 4 lines", `<example>` blocks); `gpt.txt` (107) is a
"deeply pragmatic engineer" persona with dirty-worktree and apply_patch rules; `gemini.txt` (155)
is an "Understand→Plan→Implement→Verify" workflow with heavy CLI-output constraints;
`anthropic.txt` (105) adds "Professional objectivity" and aggressive TodoWrite mandates;
`coding-minimal.txt` is a stripped eval prompt. Plan mode is injected per-turn, not in the base
prompt (below).

**Per-session prefix.** `SystemPrompt.environment` (system.ts:132-182) emits the `<env>` block
(cwd, worktree, git?, platform, date) plus, when the model runs on a local engine, machine-memory
lines from `EngineMemory.read` (system.ts:135,149-161) and a warning about memory contention
(citing a 2026-10-01 stall). `skills()` (184-196) lists skills verbose; `mcp()` (198-214) wraps
MCP server instructions in `<mcp_instructions>`, filtered by permission.

**Instructions (AGENTS.md).** `packages/zero/src/session/instruction.ts`: global file first
(`~/.config/zero/AGENTS.md` else `~/.claude/CLAUDE.md`, lines 60-63, first match wins in
`systemPaths` 110-153); project files `AGENTS.md`/`CLAUDE.md`/`CONTEXT.md` found *upward* from cwd
to worktree — first filename with matches wins so ancestors don't stack (122-133); plus config
`instructions` globs and `http(s)` URLs fetched with 5 s timeout (95-103, 155-169). Each rendered
as `Instructions from: <path>\n<content>`. `resolve()` (179-221) additionally attaches nested
AGENTS.md files lazily when the model `read`s a file under a new directory, once per assistant
message (`claims` map, deduped against already-loaded via `extract` 17-32 which scans read-part
metadata).

**Project memory.** `packages/zero/src/memory/render.ts:11-46` builds a `<project_memory>` block
(status Now/Next/Blocked, top-5 adopted lessons sorted by hits, directory map) with token budgets
(status 200, lessons 150, map 150, total 500, render.ts:4) and a redundancy filter that drops
entries whose text already appears in retained history (20-28). It is injected *inside the frozen
prefix* (prompt.ts:1448-1456). Memory reminders can also be appended to the first user message of a
loop (prompt.ts:1314-1328).

**Assembly order.** prompt.ts `compute` (1412-1461): `[...env, ...instructions, memoryBlock,
mcpInstructions, skills, repoMap]`, joined with base prompt at request build (`llm/request.ts:59-69`:
`agent.prompt || provider(model)`, then session system, then per-user `user.system`). Optional repo
map (`experimentalRepoMap`, 2000-file rg glob, token budget, 45 s timeout, prompt.ts:1421-1446).

**Prefix freezing (cache stability).** The whole computed prefix is frozen per
`session|agent|model|first-message-id` key (`prefixKeyOf`, prompt.ts:130-133) and persisted to disk
under `frozen-system/<session>/<sha16>` so a restart reuses the exact bytes — a rebuild would change
free-memory/date/edited instructions and cost 7-12 min of re-prefill on a 120k session
(system.ts:95-113 with dated comment). Tool schemas freeze under the same key (`frozenTools`
system.ts:116-131; `tool-freeze.ts` reconciles drift, dropping new/direct-changed tools, with an
info log when the live list diverges). Opt-out: `experimental.stable_prefix=false` (prompt.ts:1374).
Compaction changes the first message id → resets the freeze (comment prompt.ts:1370-1373).
Context-pressure notes are placed at *stable indices* so the cached prefix isn't broken mid-history
(reminders.ts:109-137): a note stays where first shown; new notes only at each tenth of pressure
(≥0.7, `PRESSURE_THRESHOLD` reminders.ts:94); notes whose history prefix changed (post-compaction)
are dropped.

## 2. Per-turn context assembly

Loop: `runLoop` (prompt.ts:1176-1626) reloads full projected history each iteration via
`MessageV2.filterCompactedEffect` (1199-1208; DB read, timed as `history_load_ms`).
`filterCompacted` (message-v2.ts:530-585) walks backwards from the newest completed summary, keeps
the retained tail (`tail_start_id`), and *reorders* for the model:
`[compaction-user, summary, ...tail..., continue-user]` (573-584), keeping only the newest
compaction-context message.

Parts→provider messages: `toModelMessagesEffect` (message-v2.ts:133-424). User messages keep
text/file parts (media replaced with `[Attached mime: name]` when `stripMedia`); compaction parts
render as the literal "What did we do so far?" (235-240). Assistant turns: pruned tool results
(`state.time.compacted`) become `[Old tool result content cleared]` (301-304); media in tool
results is hoisted into synthetic user messages for providers that reject media there
(`supportsMediaInToolResult` per-SDK table, 149-161, 306-313, 389-408); reasoning is kept with
provider metadata only for the *same* model, else downgraded to plain text (370-385);
`stripReasoning` drops it entirely (371, openai-compatible behind
`experimentalStripReasoningHistory`, prompt.ts:1376-1377); pending/running tools replay as
`[Tool execution was interrupted]` so Anthropic's tool_use/tool_result pairing never dangles
(357-368); interrupted turns inject a human-readable notice into the next user message
(`interruptionNotice` 631-654). Signed Anthropic thinking separators preserved as a single space
(270-287). Then AI SDK `convertToModelMessages` with per-tool `toModelOutput` (413-423).

System messages become `role:"system"` messages prepended, except OpenAI-OAuth where they ride in
`options.instructions` (llm/request.ts:105,111-122). Request params merge order:
provider-transform base → model.options → agent.options → variant (94-97); plugin hooks
`chat.params`/`chat.headers`/`experimental.chat.*.transform` (72-81,124-158,1410). Tools sorted
alphabetically for byte stability (196). Session affinity headers `x-session-affinity`/
`x-zero-session` for upstream cache routing (199-216).

**Incremental history conversion.** For openai-compatible models with
`experimentalInMemoryContext`, converted `ModelMessage[]` is cached per loop and only the new tail
is converted when prior message ids+part-counts match (prompt.ts:1466-1489; logged as
`context_cache_hit`, 1490-1497).

**Tool output retention.** Tool outputs enter history full; each tool output is truncated *at
source* to 2000 lines / 50 KB with the full text spilled to a file plus a search-don't-reread hint
(`truncate.ts:15-16,87-137`). Older completed outputs are pruned to `[Old tool result content
cleared]` markers once >40k tokens of outputs sit beyond the last 2 turns and the saving exceeds
20k (`PRUNE_PROTECT`/`PRUNE_MINIMUM`, compaction.ts:38-39,499-543; `skill` outputs protected, 96);
prune ran after each loop (prompt.ts:1624) and is config-gated (`compaction.prune`) — **corrected by
`3d338bc44f` (2026-10-14): pruning now runs only when the last turn filled 75% of the usable window, because
per-loop pruning rewrote old results mid-prompt and cost ~6-minute re-prefills (Part D §D6.10).**
`experimental.continuationThinkingBudget` lowers `reasoning_budget_tokens` on routine tool
continuations (request.ts:106-109).

## 3. Compaction, summarization, cache policy

**Trigger.** Single shared definition of "how full" in `packages/schema/src/context-budget.ts`:
`budget = limit.input − reserved(default min(20k COMPACTION_BUFFER, maxOutput))` or
`context − maxOutput` (36-41; `OUTPUT_TOKEN_MAX=32k` line 7); `occupied()` prefers provider
`total` else input+output+cache read+write (47-49). Auto-compact fires *before* the turn when the
last finished message overflows (prompt.ts:1289-1296 → `compaction.create({auto:true})` enqueues a
`compaction` part), and *after* a step that returned `compact` (refused/length), with a
double-refusal guard `compactedAfterRefusal` that stops "compact the summary" loops with a
`ContextOverflowError` (prompt.ts:1181-1185,1580-1614; local-engine memory refusal path).
Manual: CLI only (`run.ts:1070 auto:false`); `compaction.auto:false` disables (overflow.ts:27).
`estimate` fallback covers providers that omit usage (overflow.ts:21-25,29).

**Selection.** `select` keeps a verbatim tail sized 25% of usable window clamped [2k, max(15k,
15% of window)] (compaction.ts:96-98,306-316 — comment records why a fixed 15k starved a 262k
model), whole user-turns newest-first, splitting one turn if needed (360-395,449-495); config
`tail_turns` caps how many turns are considered. Head = everything older → serialized for the
summary. Prior compactions' summaries are carried as `previousSummary` and their raw messages
hidden (288-304).

**What the summary is.** Anthropic-structured template (Objective / Important Details / Work State
Completed-Active-Blocked / Next Move / Relevant Files) in
`packages/core/src/session/compaction.ts:16-46` (`SUMMARY_TEMPLATE`), with merge rules that
prior-summary wins for carry-forward, conversation wins on conflict (47-55,180-199). Output capped
4,096 tokens, asked to stay ≤1,500 (zero/compaction.ts:50,150). Serialized conversation clips each
tool output/input/reasoning to 2,000 chars (46 with the measured comment: whole files+thinking were
82% of a 102k summary request, 339 s prefill); refusal ladder reduces to 500 then 200 chars and
drops outputs, then halves history up to 3× (81-85,110-111,828-877). Summarizer uses the model's
*lightest variant* — reasoning is most of compaction latency locally (613-617).

**Continuation summary (cache-aware).** If the previous turn's exact request is remembered
(`continuations` map prompt.ts:1156-1165), the summary is asked as the *next turn of that request*
(same system+tools+history, tools without executors, `compaction-turn` agent), reusing the engine's
cached prefix — serialized summaries hit 0% cache vs 98% (compaction.ts:698-770), used only when it
prefills ≥8k fewer tokens than the serialized copy (71,797-800).

**What is preserved across the cut.** (1) the summary; (2) verbatim tail; (3) a synthetic context
message with the user's *own* earlier messages verbatim (newest-first, ≤20k tokens or 10% window,
oversized ones cut with a file pointer — `retainedUserMessages` 101-103,237-276,960-1002, Codex
parity comment) plus the current non-cancelled todo list; (4) an "archive" full-lossless dump
(`data/compaction/<session>/<msg>.txt`, whole messages at infinite detail + todos, 645-667) — but
the archive path is attached to context **only when overflow/recovery forced it** (668,783,849,902):
in a routine auto-compaction of a healthy session nothing else keeps the discarded text. (5) The
handoff text is fed to project memory (`memory.recordHandoff`, 1131-1139). Media attachments are
dropped from context on overflow with an explanatory continue-note (1100-1105). Auto-compaction
replays an *unanswered* prompt (≤4k tokens verbatim, else spilled to file with a pointer, 101,
571-606,1025-1068) or posts a "Continue / stop and ask" nudge (1070-1128).

**Recovery checkpoint.** If summaries keep getting refused (memory-bound engine) or the model burns
its whole output budget on thinking with no text, the harness stores a *factual* checkpoint (last 6
messages clipped + archive path + prior summary clipped) instead of an LLM summary (879-938).
`prune` (see §2) is the separate cheap context-freeing mechanism; `fitConversation` drops oldest
entries when the *summarizer's own* window is smaller (318-338).

**V2 compaction** (`core/src/session/compaction.ts`): buffer 20k, keep 8k defaults (12-13), fires
pre-request when estimate exceeds `context − max(output, buffer)` (257-268), emits durable
`Compaction.Started/Ended` events carrying summary + recent text; simpler (no continuation/retained-
user machinery yet). V2 history load (`core/src/session/history.ts:66-101`) selects messages after
the latest compaction seq plus post-baseline system updates, per `SessionContextEpoch`
(`session_context_epoch` table, core/session/sql.ts:168-176, context-epoch.ts:30-50).

**Prompt-cache policy** (`packages/llm/src/cache-policy.ts`): default `auto` places ephemeral
breakpoints at last tool def, last system part, latest user message (18-22,99-111); applied only
for `anthropic-messages`/`bedrock-converse` (42) — OpenAI/Gemini implicit caching is trusted to
prefix stability (which is exactly what freezing + stable pressure notes buy on the local path).
`getUsage` reads Anthropic/Vertex/Bedrock/Venice cache-write fields explicitly
(zero/session/session.ts:360-374).

## 4. Session persistence

**File store (V1 legacy/aux):** `packages/zero/src/storage/storage.ts` — JSON-per-key under the
data dir (`file()` 59-61), versioned migrations (76+), used for e.g. frozen system prompts and
`session_diff`.

**SQLite (drizzle, core):** `core/src/session/sql.ts` — `session` (cost/tokens columns 43-48,
`revert` JSON 49, summary diffs 41), `message`/`part` (V1 rows, JSON `data` payloads 68-98), `todo`
(100-117), and the V2 tables: `session_message` (append `seq`, unique per session 119-138),
`session_input` (durable prompt admission: `admitted_seq`, `promoted_seq`, delivery steer/queue,
140-166), `session_context_epoch` (system-context snapshot + baseline seq 168-176). Events are a
real event store: `event` + `event_sequence` (aggregate, monotonic seq, unique (aggregate,seq)) in
`core/src/event/sql.ts:4-25`.

**Eventing:** `EventV2Bridge` (zero/src/event-v2-bridge.ts:19-62) stamps instance location on
publish, rebroadcasts everything on the in-process `GlobalBus`, and re-emits *durable* events as
`sync` payloads for cross-process sync. Event inventory/manifest: `schema/src/event-manifest.ts`
splits durable vs live definitions (35-80), re-exported by `zero/src/event-manifest.ts`. Compaction
publishes `SessionCompactionEvent.Compacted` (compaction.ts:1140-1142) and V2 publishes
Compaction.Started/Ended.

**Restart/resume.** Execution is process-local: `SessionRunState` runners are cancelled by scope
finalizer (run-state.ts:36-52); there is no durable run lease. Resume works through *durable
state*: the loop recomputes pending tasks each pass — an unattempted `compaction` part re-runs
compaction, but one whose summary message already exists (crash mid-summary) is not re-attempted
(message-v2.ts:606-617 with comment); interrupted turns replay as error tool results + an
interruption notice (631-654); frozen prefix and tool list reload from disk so a restarted session
still hits the engine prefix cache (system.ts:95-113). V2 goes further: `SessionV2.prompt` admits
one durable `session_input` row then advisory-wakes; steering promotion happens at safe provider-
turn boundaries (`core/src/session/input.ts` admit 41+, promoteSteers 245+, promoteNextQueued 268+;
exact-retry reconciliation in AGENTS and `input.ts:131-165`); `SessionExecution.resume(sessionID)`
(core/session/execution.ts:13) drains eligible inputs after restart. V1 has no per-turn durable
transcript boundary.

**Revert/checkpoints:** `Snapshot` is a *shadow git repo* per project under
`data/snapshot/<project>/<hash(worktree)>` with its own git-dir and info/exclude handling
(zero/src/snapshot/index.ts:71-196); `step-start`/`step-finish` parts carry snapshot hashes
(processor.ts:439-443,504-508), `SessionRevert.revert` tracks→restores→reverts file patches and
stores a forward diff + `revert{messageID,partID,snapshot}` on the session (revert.ts:40-92),
`unrevert` restores (94-102), `cleanup` truncates the message/part rows (104-127). Message diffs
for the UI come from `SessionSummary.computeDiff` between first step-start and last step-finish
snapshots (summary.ts:82-100).

## 5. Provider abstraction

Two stacks. **V1/AI SDK (default):** `zero/src/session/llm.ts` runs `streamText` against AI SDK
provider packages, converts `fullStream` to shared `LLMEvent`s (`llm/ai-sdk.ts`), and applies
compat transforms in `zero/src/provider/transform.ts` (1,927 lines: reasoning field normalization
per DeepSeek/OpenRouter/etc. 303-345, `cacheControl` handling 465-469, encrypted-reasoning
include for store:false OpenAI 20-23). Model catalog from models.dev: limits/cost/`context_over_200k`
tier pricing mapped in `zero/src/provider/provider.ts:756-757,1290-1302,1338-1360`; fuzzy model-id
suggestions on typos (1423).

**V2/native (`packages/llm`):** Effect-schema core. Protocols: openai-chat, openai-responses,
openai-compatible-chat, anthropic-messages, gemini, bedrock-converse (+event-stream framing);
routes compose Protocol × Endpoint × Auth × Framing (`packages/llm/src/protocols/`, `route/`,
`llm/AGENTS.md` architecture section). Providers: openai, azure, anthropic, google/vertex,
bedrock, cloudflare, copilot, xai, openrouter, plus openai-compatible profiles (DeepSeek,
Together, …) reusing `OpenAIChat.protocol` verbatim. `LLM.stream/generate` = exactly one provider
turn; `ToolRuntime.dispatch` = one typed tool call; continuation is the caller's job (llm AGENTS
tool-dispatch section). Native runtime gate (`zero/src/session/llm/native-runtime.ts`, opt-in via
`ZERO_EXPERIMENTAL_NATIVE_LLM`) supports OpenAI, any API-key OpenAI-compatible (incl. local MLX),
and Anthropic-family entries; falls back per-request otherwise.

**Tool-call repair:** `llm.ts:320-321` wires `experimental_repairToolCall` → `llm/repair.ts`
(pi-derived, MIT): strips `functions.`/`default_api.`-style prefixes (line 58), alias map
(`cat→read`, `str_replace→edit`, ~40 entries, 18-56), JSON-arg repair + schema coercion, and routes
*undeclared* MCP/deferred names through `tool_search`/`tool_exec` meta-tools so MCP servers joining
late never mutate the frozen prefix (11-14, tools.ts:484,800).

**Streaming robustness:** length-guard (pi-derived) forbids executing tool calls of a
`finish:length` turn (llm/length-guard.ts:5-8; processor.ts:481-501 marks them error);
finish-reason remap via config compat (length-guard.ts:13-31); retry with backoff honoring
`retry-after[-ms]` (session/retry.ts:54-75); content-filter finish surfaced as error
(prompt.ts:1553-1565); `finish:length` triggers up-to-N silent continuations with a synthetic user
nudge (prompt.ts:1123-1154).

**Model switching mid-session:** history replay is model-aware — cross-model turns drop provider
metadata/reasoning signatures and re-pair tool calls (message-v2.ts:252,291,330,370-385); a model
change alters `prefixKey`, forcing a fresh frozen prefix and a cold cache. Context-limit metadata
flows: model.limit{context,input,output} → ContextBudget → compaction + pressure ratio; `--context`
style overrides exist in eval (run.ts:271-274) and `outputTokenMax` flag.

## 6. Observability / metrics

- **Structured logs**: key=value structured lines with a per-process `run` id to
  `<data>/log/zero.log` + stderr, `ZERO_LOG_LEVEL` (core/src/observability/logging.ts:6-56;
  global.ts:24). The loop logs `loop`, `session history loaded` (+`history_load_ms`),
  `session context assembled` (+`context_assembly_ms`, `context_cache_hit`, `system_chars`,
  message count — prompt.ts:1203-1208,1490-1497), pruning, compaction refusals/retries with token
  counts, tool-freeze drift, orphan tools.
- **Per-message accounting**: every assistant row carries `cost` + `tokens{input,output,reasoning,
  cache{read,write}}` (session.ts getUsage 353-407: tiered per-context pricing, Anthropic
  cacheCreation metadata, Copilot nanoAIU; processor.ts:480-481) and `timings{start,firstToken,end,
  engine{promptTokens,cachedTokens,prefillMs,generatedTokens,decodeMs}}` (processor.ts:512-528).
  Session row aggregates cost/tokens (sql.ts:43-48).
- **Engine timings**: local mlx-serve/llama.cpp reports measured prefill/decode/cached per request
  on the final raw chunk; the harness has no other way to separate prefill from queue/transport, so
  it is lifted through provider metadata (`llm/engine-timings.ts:1-99`, raw-chunks passthrough) and
  published as durable `engine.timings` events (processor.ts:464-474; schema engine-event.ts:72).
  Native runtime does not carry them yet (session/llm/AGENTS.md).
- **Telemetry plugin** (`packages/zero-plugins/telemetry/`): SQLite ledger at
  `~/.local/share/zero/telemetry.db` (ledger.ts:71-120) with `requests` (ttft_ms, tokens_per_second,
  prefill_ms vs engine_decode_ms — the recorded harness-overhead delta, index.ts:132-136;
  prefix_hit = cacheRead/prompt, prefix_broken when cache reads regress, index.ts:155-163) and
  `tool_calls` (args_hash, duplicate read / noop edit / read-before-edit flags, index.ts:103-123),
  plus a local dashboard on port 11235 (index.ts:19,169-211). Decode rate is measured over the
  first→last streamed *part* window, deliberately not the message window (index.ts:21-52).
- **Evaluation** reads the ledger (`zero-eval/src/ledger.ts:18-40`) + hidden-test grades + recall
  probes; see §7.
- Session sharing (`zero/src/share/`) uploads sessions; not a metrics channel.

**Missing for harness-vs-model work** (candid): no per-request decomposition of prompt bytes by
component (system vs instructions vs tool schemas vs history vs tool outputs) — only total
`system_chars` and tokens; no cache-write accounting for non-Anthropic routes; engine timings exist
only for the MLX family (remote providers report none, so overhead attribution is impossible
there); no per-compaction outcome metric in the ledger (tokens reclaimed vs summary quality —
recall probes are eval-only); no tool token-cost column (only output_bytes); costs are 0 for local
models so cost-based comparisons are meaningless locally; `executive` decisions are logged but
advisory and unscored so far (executive.ts:10-13, prompt.ts:1254-1264).

## 7. Evaluation (packages/zero-eval)

Task format `fixtures/<id>/task.json` (`types.ts:3-19`): kinds `coding|long|subagent|browser|
ladder`, `prompt` or `command:"goal"` + `arguments`, `timeout_ms`, optional `expect_tool_calls`,
`config`, `serve`, `require_tools`, `followups` (multi-turn), and a `retention` answer-key for an
end-of-session recall probe. Long-horizon fixtures exist: `long-kv-cli`, `long-csv-sum`,
`long-inventory-api`, `long-live-bugs`, `long-qa-spec`, ladder tiers 3/4 (ladder-t3-split,
ladder-t4-meals/rooms). `--context`/`--context-output` override the model's declared limits by
injecting a config override for the selected model (run.ts:29,55,271-274) — i.e. force the harness
to treat a 262k engine as a small window to exercise compaction. `--config` supports A/B prompt
injection (eval-only robustness instructions). Grading: hidden tests run on a copy
(`verifyCopy`, status via `grade(score.ts:33)`, timeouts diagnosed with
`verified_after_timeout`), blind LLM review for quality (research README §How runs were judged),
and `recall()` scoring of requirement retention (score.ts:73-85). Reports record model/engine/
machine/git-commit/power-source (battery halves decode, run.ts:66,119,131); speed rows come from
direct engine probes with the `timings` chunk (speed.ts:81-98) vs harness rows (prefill null).
Results under `docs/research/2026-10-09-qwen-vs-sonnet-ladder/README.md`: hidden tests tied at
every tier, blind review preferred Sonnet and the gap widened with task size (tier4 38 vs 20,
Mealbook 40 vs 27); Qwen perfect on explicit-spec tasks, failure mode = unstated pitfalls; 80-86%
of run time is decode, harness overhead <1%, rewrites 0-6%; low reasoning effort was slower *and*
worse; harness fixes measured (design-review subagent +blind gap and ~2× time; robustness
instructions +tests at ~2.5× time; pitfall-hint after JS/TS edits, unmeasured) — table lines 31-39.

## 8. Objective persistence / steering primitives

- **Todo**: `todowrite` tool (tool/todo.ts) → `Todo.Service` → durable `todo` table
  (session/sql.ts:100-117); the non-cancelled list is carried verbatim across every compaction
  (compaction.ts:983-984,1000) and pressure notes tell the model to park future-needed facts in it
  (reminders.ts:104-105).
- **Question**: blocking ask with options/custom answers bound to the tool call
  (tool/question.ts:22-41; plan_exit confirms via question, tool/plan.ts:29-46); answers return as
  tool output text.
- **Plan mode**: plan file at `<worktree>/.zero/plans/<created>-<slug>.md` (session.ts:346-350);
  `plan-mode.txt` reminder injected on plan-agent turns with plan-file pointer, `build-switch.txt`
  on the plan→build transition, experimental flag switches between legacy agent-based and
  reminder-based modes (reminders.ts:16-91); plan is durable *as a file*, surviving compaction by
  path + todo carry-over.
- **Status**: transient per-session busy/idle in an in-memory map, published as events, not
  persisted (session/status.ts:26-46).
- **Steering**: V1 `run-state` joins same-session runs (`ensureRunning`) — new prompts queue into
  the running loop's next iteration; V2 formalizes steer/queue delivery with durable admission
  (§4). **Doom-loop guard**: 3rd identical consecutive call answered with a redirect that quotes
  the previous result instead of executing; further repeats need a `doom_loop` permission
  (doom-loop.ts:9,23-44,61-74). **Executive**: SystemOne-style advisor observing step boundaries,
  decision logged, authority pending (executive.ts:10-13).

## Candid risk assessment

**Context-loss at compaction.** Biggest hole: the lossless archive is written on *every*
compaction (saveArchive is `Effect.cached` and `archiveFile` is evaluated unconditionally,
compaction.ts:645-667) but the archive *path* is only surfaced to the model when overflow/recovery
set `archive` (668,783) — routine compactions discard head text with no pointer to the file that
contains it. Second: reasoning is summarized once and its detail is gone; the retained-user-message
budget (≤20k/10% window) cuts oldest requirements first if they're large (237-276 keeps newest
first). Third: `prune` cleared old tool outputs after each loop once 40k+ of outputs accumulate —
a model that planned edits from a file it read 12 turns ago could lose it *before* compaction ever fired.
**Corrected by `3d338bc44f`: prune now waits until the last turn filled 75% of the usable window, where it
holds off a compaction — the surprise-eraser risk is largely gone.** The summary template preserves
paths/commands but the pipeline never verifies claim survival except in eval's recall probe.

**Token efficiency.** Strong prefix-cache discipline (freeze + disk-persist + stable note indices +
sorted tools + session-affinity headers + tool_search to keep MCP out of the prefix + continuation
summaries + in-memory incremental conversion). Costly parts: full history reload + full JSON
re-conversion per loop iteration unless `experimentalInMemoryContext` (flag-gated, openai-compat
only); `estimate()` re-stringifies slices during tail selection; per-step recompute of pressure
ratio is cheap. 50 KB/2000-line tool output ceiling is generous relative to small windows.

**Persistence.** V1 sessions are durable row-by-row but loop execution state (frozen prefix cache
excepted) is process-local; recovery relies on replaying durable parts, which works for
compaction/interruption but has no durable turn boundary. V2 fixes admission (session_input) and
context epochs; clustered execution is still future work (AGENTS).

**Instrumentation gaps for harness-vs-model comparison.** Component-level prompt-token breakdown;
cache accounting for implicit-cache providers; compaction outcome ledger (tokens freed, summary
faithfulness); generalizable engine timings (only MLX today, absent on native runtime); executive
decision-vs-outcome scoring; tool-attributed token cost. The telemetry ledger already stores the
two best harness signals — engine-vs-observed decode delta and prefix_broken — but nothing in the
loop *consumes* them.

---

## Overall assessment: does the foundation fit the goal?

**What Zero Code already does particularly well** (evidence in Parts A–C):
- Edit-application reliability engineering: exact-first matcher cascade with uniqueness gating,
  disproportionate-span refusal, actionable near-miss errors, automatic post-edit LSP diagnostics
  (`tool/edit.ts:695-749,197-202`).
- Output-budget coherence: one truncation service, spill files, next-action hints, explicit
  empty-vs-error distinction everywhere (`tool/truncate.ts:86-147`, Part B §7).
- Weak-model recovery: repair aliases, argument coercion, doom-loop redirects, invalid-args rewrite loop
  (`session/llm/repair.ts`, `session/doom-loop.ts`).
- Prefix-cache discipline with measured justification, including disk-persisted frozen prefixes and
  cache-continuation compaction summaries (`session/system.ts:95-131`, `session/compaction.ts:698-770`).
- Durable checkpoint/revert through shadow-git with alternates seeding (`snapshot/index.ts:71-233`).
- V2's durable admission design: idempotent, conflict-checked, monotonically sequenced
  (`core/src/session/input.ts:41-81`).
- (Part D) A default-on autonomy stack no other open harness ships: `/goal` done-ness driver with a `--verify`
  oracle, guard's read-table/pitfall-lints/loop-window/unfinished-reply continues, the design plugin's measured
  auto review gate, and engine's measured adaptive thinking policy (`zero-plugins/*/src/index.ts`, Part D).

**What is unnecessarily complicated or duplicated:** the two full loop stacks; two message→provider conversion
stacks; the 9-generator replacer cascade's early-exit ordering rules; snapshot revert batching for a UI-undo
feature; a 3.5k-line code-mode interpreter with no non-experimental consumer; per-model prompt `.txt` forks that
duplicate most content.

**Known reliability problems recorded during this session:** see ZERO_CODE_ISSUES.md (nothing blocking was hit
so far; the V1/V2 split itself is logged there as the master limitation).

**Correction note (owner review, 2026-10-14):** Parts A–C missed `packages/zero-plugins/`, the `app`/`desktop`
packages, and server-side subsystems (engine lifecycle, message-text search, `prompt_async`). This under-credited
Zero Code's autonomy story — the goal/guard/design plugins ship default-on and implement a non-durable version of
the done-ness/verification/steer-continuation mechanisms that 04–10 called absent. Part D is the correction; D6
lists every downstream claim fixed, and PROGRESS.md carries the full 44-claim verdict table. No *architectural*
verdict below changes: what exists is plugin-level and per-process, and the durable-core argument stands.

**Foundation verdict:** keep the Effect v4 + SQLite event-store + Location-scoping foundation; finish and promote
V2; port V1's reliability machinery into it; keep the tool layer (it is the crown jewel) provider-agnostic across
the AI-SDK and native runtimes. A rewrite (Option D) would discard the edit/truncate/recovery layer that Parts B
and C show is the most differentiated code in the repository.

---

## Part D — Plugins correction pass (goal, guard, engine, design; 2026-10-14)

Correction for the pass's largest blind spot: Parts A–C read `packages/zero-plugins/` only for telemetry. The other
four plugins ship **in the default bundle** — `bundlePlugins()` loads engine, guard, telemetry, goal, design
unconditionally (`zero/src/plugin/index.ts:102-104`; only `ZERO_DISABLE_BUNDLE=1` disables, :97-100) — so they are
part of the shipping harness, on the live V1 path, and several conclusions in 01–10 under-credited Zero Code.
Method: source audit (2026-10-14, head `480df4d680`) plus read-only sqlite forensics on this research's own
`~/.local/share/zero/zero-main.db` (this session ran under the goal plugin).

### D1. goal — a real done-ness driver, but non-durable

- `/goal <text> [--verify "<cmd>"] [--max N] | stop | status` via the `config` hook (`goal/src/index.ts:130-135`);
  state `{goal, verify, max, iteration, done}` lives **only in a closure `Map`** keyed by session (`goals`, :267).
  The `/goal` command text is replaced in-place by the brief before storage (`command.execute.before`, :336-357);
  every plugin prompt is a synthetic text part tagged `metadata.zero_goal` (`part()`, :253-255) — durable in the
  transcript, useless as state.
- Driver = one event: `session.idle` (:359; emitted at `zero/src/session/status.ts:43`). Per idle: busy latch →
  1.5 s debounce → require last message a clean completed assistant message → refusal check → todo fetch over the
  SDK → `check()` → `next()` → `send()` — an **unawaited** `client.session.prompt` that inherits the last real
  message's agent/model (:384-399). Brief template ("At most ${max} continue prompts…", :102-113); continue
  template "Goal check ${iteration} of ${max}" with three bodies (verify tail / marker-vs-open-todos reproach /
  marker-missing) (:320-327); goal reference re-sent in full only when ≤240 chars else head+file-pointers (:118-128).
- **Oracle**: with `--verify`, exit-0 of the command (perl-alarm-timeout runner, 120 s default, :149-156,286-288);
  otherwise exact-line `GOAL COMPLETE` match (`completed()`, :93-96), which does **not** finish a goal whose todo
  list has open items *only if the model called `todowrite` after goal start* (:290-293). Cap 20 default, enforced
  only inside `next()` (:311-319) — in-memory, so a restart resets the count.
- Default-on **goal-less todo-continue** mode (cap 8, `todo_continue` default true, :166-172,409-415) keeps
  todo'd sessions running without a goal.
- Compaction restatement has **no compaction hook**: at the next idle it infers a compaction happened by scanning
  for `info.summary` parts (:239-241) and then sends the full goal text (:402-406) — restates the goal, not the
  verify contract or iteration position. `experimental.session.compacting`/`compaction.autocontinue` hooks exist
  and are unused here.
- **Live evidence** (zero-main.db, read-only): continue 13 / start 10 / done 5 goal parts. This very session:
  17:21:13 start → 00:15:35 "Goal check 1 of 20: … ended with GOAL COMPLETE, but these todos are still open"
  (the open-todo hold firing as designed) → 00:15:52 done. Restart resilience: a fresh server **adopted** the
  engine at 00:57:03 and the user re-issued `/goal` 12 s later — transcript survived, driver state did not.
  Overnight ladder session ran continues 1→10 before verifying; three later goals recorded start parts with zero
  continues (replaced/recycled). Post-compaction full restatements: **0 parts ever fired in this DB**; goal-less
  todo-continues: **0**. A 2026-09-16 session is titled "Partly — let me be precise, because my GOAL COMPLETE was
  sco…" — a premature marker that passed because the todo guard wasn't engaged.
- **Weaknesses:** no durable state (restart silently downgrades to todo-continue and `/goal status` says "No goal
  is active", :257-260); no wake-on-time/process (everything is idle-gated); marker oracle with only the
  todowrite-conditional guard; budget counts continues, not tokens/time; refusal detection is a regex on harness
  error wording (`/^The user (rejected permission|dismissed this question)/`, :216-222); all continue failures
  swallowed (`catch { return }`, :416-417); user prompts admitted during a minutes-long verify can lose ordering
  (identity check only, :405).

### D2. guard — advisory tool hygiene, default-on

Four rules at `tool.execute.before/after` + idle continue, all advisory (mutate args / append notes; no vetoes):
arg coercion (deletes literal `"undefined"` args, strips path quotes, :31-44); **read-before-edit nudge** — per-
session `reads` set, editing an unread file appends `[zero-guard] … edited without being read …` (:225-257; the
comment :178-183 records it *used to refuse* and was demoted because the edit tool's near-miss text already
teaches); fingerprint loop detector (ring of last 12 calls, 3 identical → warning; any edit resets, :46-63,236-252
— complements doom-loop's consecutive-only rule); four **static pitfall lints distilled from the 2026-10-09/10
ladder** (exit-after-write, local-utc-dates, in-operator, parse-fallback) on JS/TS edits, comments stripped, once
per file+id (:102-144). Plus **unfinished-reply continue** at idle: unbalanced code fence or INTENT-regex ending
("let me / I'll / now I…") → synthetic continue, capped 2 per user turn (:65-100,274-309). State in-memory, lost on
restart. Live: 11 read-before-edit notices, 5 loop warnings, 1 guard-continue, 0 pitfall emissions in this DB.

### D3. engine — thin plugin, fat server-side lifecycle (the wake-precedent subsystem nobody cited)

The plugin itself only syncs loopback provider config from `/v1/models` (clamps authored context windows to served
values, :71-112), sets 30-min stream timeouts on local providers (:63-69), fails requests pre-flight on
unreachable engine (`chat.params`, :231-259), and **forces thinking off for `title`/`compaction` agents with
measured numbers** (summary decode 80 s vs 31-38 s at equal recall, :34,250-251; effort-aware prefix-free variant
:44,252-258) — a loop-consuming adaptive policy the docs claimed didn't exist. All lifecycle is server-side:
`zero/src/engine/manager.ts` adopts/boot/health-polls/restarts (≤3, backoff) the mlx engine under a file lock with
ownership handoff; warm-up completion at startup (:365-376). `engine/memory.ts` (from the 2026-10-01 starvation
incident) waits **inside the provider turn** on a memory refusal, polling `/props` but waking instantly when any
session-started shell job ends (`Effect.raceFirst(sleep, ShellJobs.anyEnded)`, :91-160) — the strongest existing
precedent for roadmap wake-on-process. Nothing in engine ever wakes a session after idle+exit.

### D4. design — shipped auto-routed reviewer gate

At idle, if the current turn cleanly edited UI-extension files and wasn't yet reviewed (`UI_FILE` regex +
`turnStart`/`reviewed()` checks, :21,79-126), inject one synthetic prompt carrying a `subtask` part for the hidden
`design-review` subagent (:160-187) — browser review (desktop+phone+dark), ranked report, same model as builder
(fresh context, not fresh weights), one review per turn, reviewer cannot edit or spawn. The hidden-not-offered
choice is priced: when the model could call it, Qwen added a third ~17-min review per task (:56-58 comment);
ladder README:34 measured blind 29 vs 22 at ~2× time. Live: 32 review parts across 7 sessions.

### D5. Plugin API ceiling (what the roadmap must change, not assume)

Hooks (`packages/plugin/src/index.ts:222-335`): config, chat.*, tool.execute.before/after, `permission.ask` (the
only veto), command-before, message/system transforms, compaction transforms, tool definitions; input carries a
full SDK client to its own server (:56-74). Host: triggered hooks awaited sequentially in bundle order (:315-328);
every bus event delivered fire-and-forget with per-hook error isolation (:279-293). **A plugin cannot hold durable
state, replay events on startup, or schedule across process death** — the `event` table is durable (97,625 rows
here) but plugins get a live subscription only. That, not concept absence, is exactly why the goal driver's oracle
is non-durable — and why Phase 3's job is making these organs first-class, *porting* proven plugin behavior
(idle-check → boundary-check; continue-prompt → durable wake) rather than inventing it.

### D6. Corrections to Parts A–C and to 04–10 caused by this gap

1. "no wake primitive, no objective/done-ness anywhere" (09 P3; echoed 04 §D/§E, 05 §4, 07 S2, 10 #1/Closing) →
   wrong as stated: `/goal` ships a done-ness driver (D1) and desktop ships a recurrence scheduler that starts
   saved prompts as new sessions (`desktop/src/main/schedules.ts:74-110`); still true: no **durable, session-level**
   objective/wake in core, nothing wakes an existing session after process exit.
2. "read-before-edit prompt-convention only" (01 exec #4, 05 F2) → partial: guard keeps the read table and nudges
   (D2); still missing: crush-style refusal + mtime freshness; R4's 1/4 reading is measured *with* the nudge on.
3. "no conversation search / add FTS5" (07 S6, 09 W5, 10 #11) → partial: LIKE+json_extract message-text search
   ships and is wired to UIs (`zero/src/session/session.ts:203-213`, `handlers/session.ts:79`); still missing: FTS5
   index/ranking/jump-to-message.
4. "frontier review priced but unrouted" (07 S4, 09 P4) → partial: the *verify/review* boundary kind is already
   auto-routed by the design plugin (D4); model-tier routing (reviewer runs session's model today) is what's open.
5. "pitfall checklist unshipped" (06 R1, 07 S5) → partial: guard's four ladder-derived lints ship default-on (D2).
6. "`--verify` is eval-only, promote to primitive" (07 S2, 09 P3) → partial: `/goal --verify` is the product
   contract already (D1); what's missing is durability of the row + cleanroom execution + V2 boundary checks.
7. exec #7 "nothing consumes telemetry" → refine: the engine plugin consumes *live probes* with measured policy
   (D3); the telemetry **ledger** is still read only by zero-eval reporting.
8. Part C prune risk → superseded by `3d338bc44f` (see C4 note above).
9. Also true and previously under-stated: V1 has `prompt_async` (used by desktop schedules), so "async headless
   surface" exists on the shipping path; only V2 lacks one (C17 unchanged for V2).
10. Prune gate (`3d338bc44f`, 2026-10-14): end-of-loop pruning now runs **only when the last turn filled 75% of the
    usable window** — pruning rewrote old tool results mid-prompt and cost ~126k-token re-prefills (~6 min) on
    idle-resume; Part C's "prune clears old outputs after each loop… before compaction ever fires" risk is
    materially reduced: prune is now a pre-compaction lever, not a surprise between-compaction eraser. `dc49a6f04a`
    fixed eval boot from inside a Zero Code session (inherited server password → 401).
