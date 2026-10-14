# Cline — source audit

HEAD `26cea2fa8bc4` (2026-10-13), `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/cline`.
Comparison frame: Zero Code baseline (01-ZERO_CODE_BASELINE.md, HEAD 123cb9262f).

## Architecture

**The headline fact: Cline has rebuilt its engine as a standalone SDK and the VS Code extension is now a
thin host.** The workspace is `sdk/packages/{shared,llms,agents,core}` + `apps/{vscode,cli,hub}` with a strict
layering rule (`sdk/ARCHITECTURE.md`: llms→shared, agents→llms, core→agents, apps→core). The extension's old
`core/task` is gone (`apps/vscode/src/core/` has no `task/`; `apps/vscode/src/core/README.md` still describes
the pre-SDK tree — stale doc). Task execution goes through `apps/vscode/src/sdk/` coordinators
(SdkController, message-translator, task-control-coordinators) into `@cline/core`.

- **Loop** — `sdk/packages/agents/src/agent-runtime.ts` (2,910 lines): `AgentRuntime.execute()` `:787`,
  `while` loop `:834-1096`. Native tool calls via Vercel AI SDK — **the XML tool-streaming era is over**.
  Legacy vocabulary survives only as a migration map: `apply_diff→editor`, `replace_in_file→editor`,
  `attempt_completion→submit_and_exit` (`sdk/packages/core/src/runtime/orchestration/runtime-builder.ts:102-109`).
- **Session/persistence** — `sdk/packages/core/src/session/`: checkpoint diff/restore, session versioning
  (`session-versioning-service.ts`), forks (`fork-metadata.ts`), snapshots, display-message projection.
- **Tools** — `sdk/packages/core/src/extensions/tools/`: ~10 native tools with Zod schemas
  (`definitions.ts`: read_files `:275`, search_codebase `:372`, run_commands `:517`, fetch_web_content `:563`,
  apply_patch `:657`, editor `:706`, skills `:776`, ask_question `:822`, submit_and_exit `:846`, commit) +
  team/subagent tools (`extensions/tools/team/spawn-agent-tool.ts`, `delegated-agent.ts`).
- **Providers** — one AI-SDK gateway + factory registry (`sdk/packages/llms/src/providers/ai-sdk.ts`,
  `factory-registry.ts`), ~53 provider IDs (`providers/ids.ts`), generated model catalog
  (`catalog/catalog.generated.ts`). Provider quirks are isolated in `llms` per the layering rule.

## Key techniques (file:line)

- **Completion-by-tool + reminders.** A run ends on `submit_and_exit`; if a turn produces no tool calls, the
  runtime first injects completion-reminder user messages and continues before finishing
  (`agent-runtime.ts:823-826, 947-966`). Same pattern as Zero's completion nudges.
- **Failure taxonomy per turn.** Fresh "error slate" each iteration (`:847-849`); typed handling of
  `aborted` / `error` / `content-filter` / `unknown` finish reasons (`:861-940`); provider-error retry wrapper
  (`:1240`) and **max-tokens truncation recovery** — a turn cut off at output limit is continued with bounded
  attempts (`:923-936`, `recoverFromIncompleteMaxTokensTurn` `:1161`).
- **Context-window overflow recovery via compaction.** Overflow is caught and triggers
  `retryTruncatedTurnWithCompaction` (`agent-runtime.ts:1361-1595`; named failure messages `:106-120`).
  Compaction itself: token-estimate triggers with floor on char-based estimates
  (`extensions/context/compaction.ts:50-57`), budget projection + target ratios 0.5 default capped 0.05–0.95
  (`:100,197-223`), and an **agentic summarizer** that serializes the old span, asks a (possibly cheaper)
  model for a summary, and force-preserves a file-operations section
  (`agentic-compaction.ts:29, `ensureFilesSection`/`extractFileOps` imports `:17-27`; min summary input 1,024
  tokens `:29`). This is Zero's compaction+spill idea re-divided: their summary keeps an explicit
  touched-files ledger.
- **Plan/act concretely.** Mode enum `act|plan` (`core/src/types/chat-schema.ts:8`) plus `yolo`
  (`runtime-builder.ts:372`); mode selects a **tool preset** (`runtime-builder.ts:144-157`, `tools/presets.ts`)
  — plan mode is tool restriction + prompt framing, with plan output flowing through
  `plan_mode_respond` / turn-end-inferred `plan_completion_result` message kinds
  (`apps/vscode/src/shared/ExtensionMessage.ts:212,238`). No separate planning model or plan file format.
- **Planning/checklist = Agenda, a durable cross-session task board.** `tasks/agenda-task-tool.ts` +
  `agenda-task-manager.ts:262`: task types `suggestion|follow-up|todo|handoff|idea|reminder`, statuses incl.
  `pending_approval|approved|in_progress|expired`, priorities 0–5 (`agenda-task-tool.ts:17-33`) — a model
  tool + human-approval workflow, not just an in-task todo list.
- **Context management.** @-mentions (`apps/vscode/src/core/mentions`), and
  `context/context-tracking/FileContextTracker.ts` — records which files entered context and detects
  user/linter edits outside the agent to mark context stale ("modified externally" reminders); no eager
  reload, purely dirty-tracking (header comment `:7-10`).
- **Checkpoints are no longer shadow-git.** New restore path uses the **user's own repo**: `git stash push
  --include-untracked` + `git clean -fd` with prior-stash preservation (`session/checkpoint-restore.ts:26-66`).
  Cheaper and transparent; requires a git repo, and can collide with the user's own stash/worktree state.
- **Steering.** `consumePendingUserMessage()` at loop boundaries (`agent-runtime.ts:2292`) — mid-run user
  messages promote at the next turn, the same "safe boundary" semantics Zero documents for V2.
- **Browser: deleted, mostly.** No browser tool exists in the new toolset (`definitions.ts`/`presets.ts`);
  only a legacy Puppeteer `UrlContentFetcher` is reused for `fetch_web_content`
  (`apps/vscode/src/sdk/SdkController.ts:46`, `services/browser/`). The interactive browser tool survives
  only as extension settings cruft (`core/controller/browser/`). Documented feature, source-absent.

## Strengths (long-horizon)

- Durable session core with versioning/fork/snapshot and checkpoint restore — closest existing analogue to
  Zero's V2 ambitions, and it *ships*.
- The runtime's turn-level failure engineering (finish-reason taxonomy, max-tokens continuation, provider
  retry with fresh state, overflow→compact→retry) is exactly the machinery Zero's baseline audit found
  missing from V2; Cline's copy is dense but complete and well-commented.
- Compaction that preserves a touched-files ledger is a better verbatim-tail complement than summaries alone.
- Layering discipline (llms isolated; extension as host) made the engine reusable (CLI, hub, agents SDK).

## Weaknesses

- 2,910-line god-object runtime with heavy stateful bookkeeping (`this.state.*` resets at `:793-808`) —
  Zero's monolith critique applies verbatim, at a larger scale; plus dual message taxonomies
  (extension ⇄ SDK translators in `apps/vscode/src/sdk/`).
- Checkpoints now require the user repo to be git-clean-ish; stash-based restore is fragile against
  concurrent user git work; no equivalent of shadow-git isolation for non-repos.
- Plan mode is thin (preset + prompt); no durable plan artifact, no plan→act handoff state beyond mode flip.
- History: Cline oscillated strategies (XML tools → native; browser tool → deleted; diff strategy →
  editor/apply_patch), which shows in migration maps and dead code paths.

## Works vs unproven

- **Source-verified:** loop + tool schema set, overflow-recovery path, compaction trigger math, agenda store
  schemas, git-stash checkpoints, mode presets, context tracker.
- **Documented-only / source-absent:** interactive browser; marketing claims about evals/leaderboards;
  "Cline passes" billing/subscription pieces are integration-shipped, not auditable here.
- **Unproven in-repo:** effectiveness of Agenda vs simple todos; whether plan mode measurably improves
  outcomes (no evals in `evals/` isolating it).

## What Zero Code should learn / reject

- **Learn:** finish-reason–keyed turn handling with per-turn error-slate reset (validate against Zero's
  doom-loop redirection); max-tokens *continuation* (bounded) rather than only length-error reporting;
  compaction summaries carrying a forced file-ops section (pair with Zero's verbatim tail — different
  information, not redundant); checkpoints-in-user-repo as an explicit product choice (transparent but
  repo-required) — Zero's shadow-git remains the better default for non-clean states; agenda-style durable
  task board with `pending_approval` status is a genuinely good V2-side idea (handoffs survive sessions).
- **Reject:** a bespoke session-versioning/fork parallel to Zero's V2 projections (don't duplicate);
  the extension⇄SDK double-message-translation tax (Zero's single protocol is cleaner); mode-as-tool-preset
  *without* Zero's registry visibility — presets alone are weaker than Zero's permission-scoped toolsets;
  stash-based checkpoints as the only mechanism.
