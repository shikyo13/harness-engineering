# Roo Code — source audit

Local HEAD `b867ec914575` (2026-05-15) == **origin/main tip** (verified via `git ls-remote --symref`); all
2,167 remote branches have tips dated ≤ 2026-05-15, so the repo is dormant on GitHub, not "stale default".
CHANGELOG 3.53.0 explains: the team went all-in on Roomote and a community team is taking the plugin over;
README.md:68 also points to ZooCode (community fork). Cline-fork lineage is self-confirmed (README.md:68
"Cline (from where Roo Code originated)") and structurally visible (same XML `assistant-message` parser
family, `attempt_completion`/`read_file`/`execute_command` tool vocabulary).
Comparison frame: Zero Code baseline (01-ZERO_CODE_BASELINE.md, HEAD 123cb9262f).

## Architecture

Classic VS Code extension, monolith-in-`src/core`: `task/Task.ts` loop, `assistant-message/` XML streaming
parser **and** a parallel `NativeToolCallParser` (both live: `Task.ts:101-102`, native path processes
`tool_call_partial` stream chunks at `:2801-2812`), `tools/` with 24 tool classes, `config/CustomModesManager`,
`condense/`, `context-management/`, `auto-approval/`, `checkpoints/` (shadow-git —
`checkpoints/index.ts:73,195` "initializing shadow git"), `message-queue/MessageQueueService.ts` (queued
mid-run user messages), `mentions/`, `context/context-tracking`. Types + mode schemas extracted to
`packages/types` (zod `modeConfigSchema`, `packages/types/src/mode.ts:96-105`).

## Key techniques (file:line)

- **Modes as user-extensible roles.** Built-ins: architect / code / ask / debug / orchestrator
  (`packages/types/src/mode.ts:168+`). A mode = `{slug, name, roleDefinition, whenToUse, customInstructions,
  groups}` (`:96-105`) where `groups` are **tool-group permission grants with options** — e.g. architect is
  `read` + `edit` restricted to `\.md$` via `fileRegex` + `mcp`. Custom modes are YAML `.roomodes` at project
  root or global (`CustomModesManager.ts:19,101-103`), schema-validated, global/project precedence
  (`source: "global"|"project"`). The model itself can request a mode switch via `SwitchModeTool`
  (user-approved).
- **Orchestrated chained modes.** Orchestrator mode has `groups: []` (holds no working tools) and delegates
  through `NewTaskTool` — `new_task {mode, message, todos}`, optional enforced todos
  (`NewTaskTool.ts:24-59`); each subtask is a full independent Task; `attempt_completion` results flow back
  to the parent ("Boomerang Tasks", renamed "Task Orchestration", CHANGELOG.md:1974). Changelog shows real
  churn: parent-state-loss fix (CHANGELOG.md:153), metadata-driven subtasks with parent resume +
  single-open safety (:753), "nasty hang, particularly in orchestrator mode" fix (:2324), tools removed from
  orchestrator groups (:2341).
- **Auto-approval policy system.** `checkAutoApproval({...})` pure decision fn + `AutoApprovalHandler`
  (`src/core/auto-approval/index.ts:47`, `AutoApprovalHandler.ts`), split by surface
  (`read|edit|command|mcp`, `tools.ts`/`commands.ts`/`mcp.ts`), layered over per-request
  `AutoApprovalState`/`AskOptions` (`:17-45`) — approvals are a policy lattice (action × scope × always/one-shot),
  notably more systematic than Cline's flat toggles.
- **Condensing.** Percentage-of-context-window auto threshold, bounded 5–100%
  (`src/core/condense/index.ts:109-110`); `summarizeConversation` (`:254`) replaces history prefix with an LLM
  summary; `getEffectiveApiHistory` (`:539`) / `getMessagesSinceLastSummary` (`:512`) keep the summary chain
  replayable; `foldedFileContext.ts` folds superseded file-reads; synthetic tool results injected so
  Anthropic-style tool_use/tool_result pairing survives truncation (`injectSyntheticToolResults` `:132`).
  Manual "Summarize Task" is the same entry point.
- **Reliability bits.** `ToolRepetitionDetector` class wired into Task state (`Task.ts:96,293`) — Zero-like
  doom-loop detection; consecutive-mistake guards; message queue for steering without interrupt.
- **What got deleted:** interactive browser feature removed entirely (PR #11392, `progress.txt` Batch 3
  cleanup notes) and the built-in skills mechanism removed/reworked. **Memory Bank: absent at this HEAD** —
  no implementation, no CHANGELOG mention in-window; treat as documented-only legacy (old docs-site feature
  superseded by `.roomodes`/rules).

## Evidence for role divisions — the honest read

The repo contains **no usage data, no evals, no A/B of modes vs single-mode**. Available in-repo signals cut
against the roleplay half and for the policy half: (1) every mode's distinctiveness is mostly
`roleDefinition` prose + `customInstructions` — unmeasured; (2) the mechanically enforceable part is
`groups`/`fileRegex` permissions, i.e. modes double as **permission profiles in disguise** (architect
*cannot* edit code — that's a real guarantee); (3) orchestrator subtask chaining produced a visible bug-and-
rework trail (hangs, parent-state loss, TODOs handoff) — the pattern works but was never stable enough to
look free; (4) the product ultimately pivoted away, and community forks, not experiments, carried it.

## Works vs unproven

- **Source-verified:** modes/`.roomodes` format and loading, per-mode tool grants incl. fileRegex,
  new_task chaining + parent resume, auto-approval lattice, condense with synthetic tool-result repair,
  shadow-git checkpoints, dual XML/native parsing, tool repetition detector.
- **Documented-only:** Memory Bank; any claim that role prompts improve outcomes.
- **Unproven in-repo:** orchestrator quality economics; native-vs-XML parser superiority; skills replacement.

## What Zero Code should learn / reject

- **Learn:** the **permission-profile reframing of modes** — one declarative file binding
  {prompt, tool set, edit-scope regex} with global/project layering is a clean, user-extensible package;
  Zero's agent config could expose per-mode `fileRegex` edit scoping (cheap over Zero's existing
  tree-sitter bash/edit permissions). `injectSyntheticToolResults`-style tool_use/tool_result repair during
  compaction — Zero's verbatim-tail selection should verify the same pairing invariant. Auto-condense as a
  bounded percentage-of-window setting is a good default surface. `new_task` metadata-driven subtasks with
  *single-open safety and parent resume* is the right shape for Zero subagents (fixes the "orphaned child"
  class of bugs).
- **Reject:** roleDefinition prose theater as a quality mechanism (no evidence it helps; costs prefix-cache
  stability — Zero already freezes prefixes); orchestrator-with-empty-tools chaining as a default path
  (subtask isolation throws away shared context and the bug trail shows it — Zero's subagent-with-
  transcript-injection is the better trade); Memory Bank flat-file ritual; keeping two live tool-call
  parsers forever.

## Depth note (correction pass, 2026-10-14)

This is the survey's thinnest audit (owner-flagged, 6.8 KB vs 10–18 KB for comparable repos). Disposition:
**light audit, kept as such** — nothing in the synthesis or roadmap adopts a Roo mechanism as a design source; Roo
is cited only as *negative* evidence (reject role-theater; borrow the declarative mode/permission format idea,
which the correction pass showed Zero partially ships already — PROGRESS C3-P5). The claims that evidence supports
were re-verified at the correction pass against the same head `b867ec91`: default-branch tip 2026-05-15 with
2,166 remote branches (dormancy confirmed); `.roomodes` custom-modes machinery present in
`src/core/webview/ClineProvider.ts` + i18n strings (declarative roleDefinition/groups confirmed); the "zero
measured benefit" claim is an absence-of-evidence finding, correctly labeled. What a deep audit would add (mode
prompt-cache cost measurements, orchestrator Boomerang bug-churn quantification, Roomote pivot evidence) would not
change any verdict in 04–10 — if someone revisits Roo, measure those.
