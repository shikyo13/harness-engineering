# 08 — Target architecture: the Zero Code harness, designed

Decision context: baseline verdict (01) and synthesis (07) select **Option B — restructure subsystems on the
existing foundation**: finish and promote the V2 durable core (already the right design), keep the tool layer (the
most differentiated code in the repo), port V1's reliability machinery into V2, and add the missing subsystems from
the synthesis. Target = the smallest architecture delivering the surveyed capabilities; every component below cites
the evidence that justifies its shape, and nothing exists for elegance.

```
                        ┌────────────────────────────────────────────────────────┐
                        │ Frontends: TUI · desktop · zero-eval · SDK · headless   │
                        │ (one closed machine contract: step_type NDJSON both     │
                        │  ways, structured errors, exit codes — 03 §8.2)         │
                        └───────────────┬────────────────────────────────────────┘
                                        │ HttpApi / events
        ┌───────────────────────────────▼────────────────────────────────────────┐
        │ Durable session core (V2, promoted)                                    │
        │  session_input admit ──► RunCoordinator ──► drain(force|wake)           │
        │  objective + done_predicate rows ──► Executive (advisory→armed)         │
        │  session_wake (timer|process|event|manual + fence) ──► scheduler fiber  │
        │  status = f(event replay)  (never a stored flag)                         │
        └───────────────┬────────────────────────────────┬───────────────────────┘
                        │ one llm.stream per provider turn│ typed events (durable)
        ┌───────────────▼──────────────┐   ┌─────────────▼───────────────────────┐
        │ Turn kernel (small)          │   │ Journal + ledger                     │
        │ promote → assemble → check → │   │ event store · engine timings ·       │
        │ stream → settle tools →      │   │ token-component breakdown ·          │
        │ continue? (tool-call signal) │   │ prefix_broken · tool-reuse flags     │
        └───────┬──────────────┬───────┘   └─────────────▲───────────────────────┘
                │              │                         │ consumed by policy, not dashboards
   ┌────────────▼───────┐  ┌───▼─────────────────────────┴───────┐
   │ Context portfolio  │  │ Tool kernel                          │
   │ budget service     │  │ registry → ONE deferral seam →       │
   │ floors·fit·ground- │  │ native dispatch  OR  PTC run_code    │
   │ truth·masking·map  │  │ shared: permissions·truncate·repair· │
   └────────────┬───────┘  │ journal·doom-loop·checkpoints        │
                │          └───┬──────────────────┬───────────────┘
   ┌────────────▼───────────┐  │ declared accesses│ worktree modes
   │ Repo intelligence      │  ├──────────────────┤
   │ repomap·LSP·read-table │  │ ┌────────────────▼─────────────┐
   └────────────────────────┘  │ │ Subagents: durable child      │
                               │ │ sessions, FTS-searchable,     │
   ┌────────────────────────┐  │ │ conflict-scheduled writers    │
   │ Provider integration   │  │ └──────────────────────────────┘
   │ model table (data) ·   │  │
   │ repair · traits · PTC  │  │ frontier boundary calls ──► (optional strong model:
   │ routing seam (exec)    │──┤   approve · verify · triage — priced in the ledger)
   └────────────────────────┘  │
                               └─ OS sandbox lane (phase 3): seatbelt/bubblewrap +
                                  classified denials beside the permission lattice
```

## 1. Agent execution engine

- **One loop, V2-shaped.** One `llm.stream` per provider turn; history reloaded from projection each turn;
  continuation = durable observation of a local tool-call (never a finish-reason heuristic); promotion of
  steer/queue inputs only at turn boundaries with one allowance reset per batch (already V2 semantics, proven by
  OpenCode shipping the same shape).
- **V1 machinery ports in, the monolith retires.** Port, in this order: retry policy (side-effect-aware `canRetry`,
  header-honoring backoff, memory-refusal ladder), doom-loop redirect, output-length handling (length-guard +
  capped synthetic continues), overflow→compaction with double-refusal guard, **now as typed errors** — V2's
  `TurnTransitionError` defects and `Effect.die` on second overflow become a real error channel (01 finding,
  baseline Part A §11.4).
- **Loop-detect with recovery, not just redirect.** Add Gemini's second tier (rare LLM double-check at
  confidence-scaled intervals) behind the doom-loop, and inject a "take a step back" feedback turn instead of
  permission-asking (F5).
- **Turn boundaries are the only mutation points**: checkpoints, promotion, wake-arming, oracle checks. No hidden
  mid-turn state anywhere (Codex StepContext principle, kept small).
- The `promptV2` blocking-entry bug dies with a streaming/async headless surface matching the closed machine
  contract (§ Frontends).

## 2. Tool architecture

- Keep the crown jewels **exactly as they are**: edit cascade (now upgrading the goal/guard bundle's shipped
  read-table nudge into enforced read-freshness
  table with mtimes; and a measured fuzzy-tier policy per E2), single truncate service (spill files, empty≠error,
  next-action hints), repair/alias/coercion layer, background shell jobs with push notices, tree-sitter bash
  permission parsing.
- **One deferral mechanism** (pi `ToolExposure` pattern): registry exposes pinned core tools + a search/load path;
  delete the four-layer stack (visibility filters → repair aliases stay only as *error recovery* → tool_search
  → per-flag codemode gating). Soft-deprecate tool names Antigravity-style (retire from default, keep opt-in).
- **PTC as a presentation mode** (07-S3): executor-enforced single `run_code` per agent-turn when selected,
  sandboxed run with hard caps, sub-calls through the *same* dispatch/permission/journal pipeline, curated output
  only. Generated from the live registry, not a hand-maintained spec.
- **Declared file accesses** on file tools → conflict-aware scheduling inside a turn (kimi) and worktree policy
  for child sessions (§7).
- Scheduler gains **explicit per-call status rows** (Validating/AwaitingApproval/Executing/…) — the durable-execution
  status substrate (Gemini's state machine, made durable).

## 3. Context and memory systems

- **Context portfolio service** (07-S1): owns every budget as data — floors (instructions 20k-style, skills,
  subagent roster, MCP), compaction reserve, map fitting, masking window. Triggered by *reported* provider tokens
  with estimate fallback (exists) — estimates demote to fallbacks everywhere.
- Compaction keeps V1's measured stack (verbatim tail sizing, retained user messages, todo carry-over,
  cache-continuation summaries, factual recovery checkpoint) and fixes its one big hole: **every** compaction
  surfaces the archive path in context, and pruning never clears a file another tool call planned against without
  a pointer (01 Part C risk list). Condensation becomes a replayable durable event (OpenHands) so resume sees it.
- **Memory stays tiny and append-only-at-tail**: `<project_memory>` keeps its 500-token budget but moves to a
  cache-safe placement (§9); lessons adoption gets feedback from the ledger (which lessons actually preceded
  successful tool calls) instead of hit-count folklore.
- **Cache-as-invariant accounting** (07-S7): per-provider cache model (explicit breakpoints where supported;
  prefix-hash hit proxy elsewhere), `prefix_broken`/hit-rate become first-class eval outcomes and executive inputs.

## 4. Repository intelligence

- Repomap ships **on**: aider's weight constants and binary-search fitting re-borrowed into the existing port,
  rebuilt change-triggered (not per-turn — local prefill economics), inserted cache-ordered ahead of history.
- LSP tool ships **on** with crush-shaped ergonomics: symbol-range edits (`lsp_replace_symbol`) and diagnostics
  as first-class outputs, keeping the automatic post-edit diagnostics loop (unchanged — it's already the strongest
  quality loop). Raw JSON.stringify output gets a projection layer (01 finding).
- Optional *verification* tier: rename/impact via call hierarchy feeds the acceptance contract (§6), not the map.

## 5. Persistent execution and recovery

- Admission/idempotency stay as designed (they're the good part). Add: **startup sweep** that re-arms eligible wakes
  and fails interrupted tools for sessions with open turns (the "post-crash continuation design" V2 explicitly
  deferred — now it has a budget); **rollout meta-line** freezing base-instruction version + model + tool-surface
  fingerprint into every session (Codex lesson: faithful resume, `comp_hash`-style forced compaction on drift);
  **unanswered-tool-call closure at admission** (Gemini) before promoting new input.
- Checkpoints unchanged (shadow-git alternates) but promoted to every turn boundary including compaction events,
  and exposed as a fork-able boundary (Antigravity per-step fork).
- Revert/cleanup semantics move with the loop, not beside it.

## 6. Asynchronous task management (the new organ)

- `session_wake` rows (kind, payload, fence watermark, hard caps) + scheduler fiber in the process; Kilo's parking
  discipline (caps per kind, one wake per session coalesced); goose's recipe format becomes the **task automation
  artifact**: parameterized prompt + pinned model/effort + response schema + sub-recipes, schedulable, extractable
  from a session ("turn this into a nightly check" — goose's session→recipe, the single best long-horizon idea in
  Tier 2).
- `objective` + `done_predicate` (command contract, cleanroom-executed — **porting the goal plugin's shipped
  `--verify` contract (01 D1) from its in-memory Map into durable rows**) checked by the Executive **at boundaries**: done → settle; not done + progressing →
  continue with budget accounting; stalled → strategy-change input (solutionSpace-informed effort re-budget);
  budget exhausted → park with wake or escalate to human. This is the armed Executive: the dead core service
  finally wired, advisory first (logged `would:` decisions priced in the ledger), armed per E6 evidence only.

## 7. Multi-agent coordination

- Subagents = durable **child sessions** (goose/kimi shape): own transcripts, FTS-searchable, cited by
  `conversation_id`; anti-injection hand-back envelope (Claude); `branch`/`share` workspace modes with
  worktree-per-writer default for background writers; model tier per role (inherit|cheaper|frontier).
- Intra-turn: conflict-aware scheduling from declared accesses; read fan-out stays cheap and parallel.
- No role pipeline. Capability profiles (roo's enforceable half) as data for the model table, not personas.

## 8. Provider integration

- Keep both stacks during migration; converge on the native `packages/llm` protocols (V2 already streams through
  it) with the AI-SDK path as the fallback adapter until parity flags clear.
- **Model behavior table as data** (aider YAML / kimi traits / qwen presets pattern): per-model edit format
  preference, thinking knobs (`disableField` arbitration), window/budget math (`completion = window − used`),
  format-bar level, repair flags — replaces prompt `.txt` fork-duplication gradually (data first, forks fade).
- Frontier boundary calls (approve/verify/triage) are a provider-routed, ledger-priced client of the same turn
  kernel — aider's empty-history cheap-editor lesson applies: minimal context per review call.
- Sandbox lane (phase 3, optional): OS-level seatbelt/bubblewrap profile *beside* the permission lattice with
  classified denials + protected subpaths (Codex), so "denied because sandboxed" and "denied because user said
  no" stay distinct to the model (Antigravity denial-respect rule).

## 9. Observability

- Keep the telemetry ledger; make the loop *read* it: cache-warm policy from `prefix_broken`, tool-surface trims
  from declared-token census, model-table corrections from per-model tool-error rates.
- Add the missing decomposition (01 Part C): per-request prompt-token breakdown by component (system/instructions/
  map/tools/history/tool-output) — one field set on the ledger request row, cheap for local (tokenizer-known) and
  reported-elsewhere providers; compaction outcome ledger (tokens freed, archive path, summary-vs-factual);
  frontier-boundary call pricing. Metrics that feed policy or eval, zero dashboards beyond the existing local one.

## 10. Component boundaries & migration strategy

Boundaries (Effect services, respecting the existing dependency direction Schema→Core/Protocol→Server):
`ContextPortfolio`, `ToolKernel`, `TurnKernel`, `ObjectiveRuntime` (wake+oracle+executive), `ModelTable`,
`JournalLedger`. Each is independently testable against fixtures; each has a named kill-switch flag during
migration.

Migration keeps Zero Code functional at every step:
1. **Port reliability into V2** (retry/doom-loop/length/overflow typed; scheduler status rows; tool-call closure).
   V1 still default. Exit: V2 passes the existing V2 test suite + ports; E7/E8 harnesses exist and pass.
2. **Promote V2 as default** behind an off-by-default-to-on flip; delete `runLoop` once zero-eval long/ladder
   parity holds (Antigravity's 1.1.17 moment, made quotable). V1 code stays one release for revert, then dies.
3. **Ship S1 + S7** (portfolio + cache invariants) — biggest token-economics wins, no durability risk.
4. **Ship S2** (wake+objective+armed-by-evidence Executive) — the autonomy unlock.
5. **Ship S5 verifier-contract, S6 parallelism, S3 PTC** in that order (each gated by its 06 experiment).
6. **Sandbox lane** last (largest surface, least coupling to everything above).

Non-goals (evidence-backed): no rewrite of the tool layer; no cross-session shared caches until measured; no
distributed execution (V2's location seam stays the seam); no second interpreter; no workflow engine beyond the
wake kinds; no benchmark theater — the ledger and zero-eval are the only scoreboards that matter.

**Internal-consistency check**: every component above traces to at least one source-verified implementation
(V2/OpenCode/Codex for the kernel; aider/Gemini/pi for context; Gemini/Cline/DeepSeek for tools; goose/Kilo/omO/
OpenHands for autonomy; kimi/Antigravity for agents; aider/kimi/qwen for model data) and to a measured Zero datum
where economics decide an ordering (ladder decode/overhead numbers, 98% continuation cache, 2×/2.5× review
pricing). The plan in 09 attaches a number to every remaining guess.

## Correction note (2026-10-14 plugin audit — 01 Part D)

Parts of §6 and §10 were written believing Zero Code lacked the autonomy organs. It doesn't — it ships them
**non-durable, plugin-side**: the goal plugin is exactly the ObjectiveRuntime pattern (idle-boundary oracle with
`--verify` exit-0 contract, marker-vs-todos hold, continue prompts, default-on todo-continuation, measured live in
the session DB), guard is the intra-turn hygiene layer (read-table + nudge, loop-window, pitfall lints,
unfinished-reply continue), design is the shipped first **boundary kind** for S4, engine's `memory.ts` contains the
wake-on-process-end primitive S2 needs, and desktop ships a recurrence scheduler for new-session prompts. Two
consequences for this architecture, which strengthen rather than weaken it: (1) the design is **validated by
Zero's own usage** — the hardest autonomy questions (oracle design, continue economics, race-with-user ordering)
are already answered in working code that this very research ran on; the move is to promote that behavior into
durable rows + provider-turn boundaries where the plugin API cannot go (no durable state, no replay, no scheduler —
01 D5). (2) §10's phase order is unchanged, but Phase 3 is explicitly a **port**: GoalState→`objective`/
`done_predicate` rows, idle-continue→`session_wake(kind=event)`, memory.ts job-wait→`wake(kind=process)`, desktop
schedules→recipe format + in-core wake, guard/design gates→runner policy + S4 boundary kinds. The plugins remain
as the user-facing surface and third-party extension point; nothing here is thrown away.
