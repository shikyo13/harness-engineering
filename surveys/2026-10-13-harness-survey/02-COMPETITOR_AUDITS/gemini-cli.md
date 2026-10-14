# Gemini CLI — source audit

Audited 2026-10-13 at head `42c51822084ac25cd5d97ac0023822c5235f88c8` ("Changelog for v0.65.0-preview.0"),
version `0.65.0-nightly.20261006.gfb972b2f8`. Shallow clone at
`~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/gemini-cli`. All paths relative to
`packages/core/src` unless stated. This is the open-source ancestor of the Antigravity CLI and of
qwen-code; the qwen-code lineage claim below is file-name-verified against the local qwen-code clone.

## Architecture

Workspace split (source-verified, `packages/`): `core` (agent loop, tools, services), `cli` (Ink TUI,
settings, slash commands), **`sdk` — a real TS SDK package now exists** (`packages/sdk/`, publishes
`@google/gemini-cli-sdk`), `a2a-server` (Agent2Agent HTTP server with `src/persistence`), `devtools`,
`test-utils`, `vscode-ide-companion`.

The agent loop is layered, not one monolith:

- `GeminiCore.sendMessageStream` (`core/client.ts:940`) — outer prompt loop; fires BeforeAgent/AfterAgent
  hooks, tracks `prompt_id`, resets loop detection per prompt (`core/client.ts:952`), MAX_TURNS bound.
- `GeminiChat.sendMessageStream` (`core/geminiChat.ts:494`) — serialized via `sendPromise` chain
  (`geminiChat.ts:500`); repairs tool-call pairing before accepting a new user turn
  (`closeUnansweredToolCallsTurn`/`closeUnansweredToolResponseTurn`, `geminiChat.ts:528-535`); durable
  history repair runs once per send outside the retry loop so retries never append duplicate synthetic
  turns (`geminiChat.ts:658-660`).
- Nested retry generator (`geminiChat.ts:663-790`): outer `maxAttempts` for connection-phase errors, a
  separate smaller mid-stream retry budget (`MID_STREAM_RETRY_OPTIONS.maxAttempts = 4`,
  `geminiChat.ts:106`) with RETRY stream events so the UI discards partial output
  (`geminiChat.ts:78-80`), and retry-aware model-config re-resolution (`isRetry: true, lastStreamError`,
  `geminiChat.ts:681-684`). Empty-response retries inject distinct "nudge" messages depending on whether
  the empty turn had thoughts (`geminiChat.ts:259-271`).
- `Turn` (`core/turn.ts`, 542 lines) converts the raw stream into typed `GeminiEventType` events
  (Retry, InvalidStream, ContextWindowWillOverflow, Content, Finished, …).
- Tool execution moved from the old `coreToolScheduler` into a proper state machine,
  `scheduler/scheduler.ts` (986 lines): per-call status `Validating → Scheduled | AwaitingApproval →
  Executing → Success|Error|Cancelled` (`scheduler.ts:309-313`), a request queue drained under one active
  batch (`scheduler.ts:446`), parallel execution, confirmation via a `confirmation-bus/`, hooks at
  validation and execution, and `tool-modifier.ts` for approval-time parameter edits.
- Next-speaker continuation (`utils/nextSpeakerChecker.ts`) is still in the loop but **off by default**:
  `skipNextSpeakerCheck ?? true` (`config/config.ts:1288`). When on, a "Please continue." recursive send
  happens at `core/client.ts:908-936`.

## Key techniques (file:line)

1. **Two-tier loop detection with LLM double-check and recovery.** Heuristic first: 5 identical tool
   calls or repeated 50-char content windows (`services/loopDetectionService.ts:29-33`), history capped at
   5000 entries. After 30 turns, an LLM checker runs on a dynamic 5–15 turn interval scaled by its own
   confidence, threshold 0.9, with a second cheap "double-check" model
   (`loopDetectionService.ts:41-64, 736`). The system prompt is a careful no-forward-progress rubric that
   explicitly refuses to flag cross-file batch edits and retry-with-variation. On detection the run is not
   aborted: `_recoverFromLoop` clears the flag once and injects a "take a step back" feedback message
   recursively (`core/client.ts:1300-1328`).
2. **4-tier edit matcher with telemetry.** `tools/edit.ts:82` declares
   `strategy: 'exact' | 'flexible' | 'regex' | 'fuzzy'`; cascade tries exact (CRLF-normalized,
   `edit.ts:145-176`), flexible whitespace/line-based, regex-built-from-search-string
   (`edit.ts:245-299`), then fuzzy (fast-levenshtein) behind `ENABLE_FUZZY_MATCH_RECOVERY`
   (`edit.ts:342-347`). Every successful strategy logs an `EditStrategyEvent` (`edit.ts:322`). Failure
   guidance tells the model to re-Read before retrying (`edit.ts:362-366`).
3. **Context management as a pluggable processor pipeline** (`context/`): `contextManager.ts` receives
   *measured* prompt-token ground truth from API `usageMetadata` after every finish
   (`core/client.ts:859-867 → contextManager.getEnvironment().eventBus.emitTokenGroundTruth`) and triggers
   work only when the deficit crosses a threshold (`contextManager.ts:377-418`). Processors: rolling
   summary, state snapshot, tool-output masking, node distillation, blob degradation, truncation
   (`context/processors/`). Tool-output masking (`context/toolOutputMaskingService.ts`) is a "hybrid
   backward-scanned FIFO": protect newest 50k tool tokens, mask older bulky outputs to disk files under
   `tool-outputs/`, batch-trigger only at 30k prunable tokens, never mask high-signal tools
   (`toolOutputMaskingService.ts:25-41`). Legacy path: `tryCompressChat` + `ChatCompressionService`
   (`core/client.ts:1226`).
4. **Shadow-git checkpointing.** Hidden repo with dedicated gitconfig and copied `.gitignore`
   (`services/gitService.ts:133-181`), path-locked snapshot staging (`gitService.ts:208`),
   `restoreProjectFromSnapshot(commitHash)` (`gitService.ts:229`) surfaced as `/restore`
   (`cli/src/ui/commands/restoreCommand.ts` via `BuiltinCommandLoader.ts:54`).
5. **Append-only JSONL chat recording with patches and rewind.** `services/chatRecordingService.ts`
   distinguishes message records, `$patch` message patches, `$set` metadata updates and `$rewindTo`
   rewind records (`chatRecordingService.ts:79-104`), replayed on resume (`chatRecordingService.ts:307-422`);
   disk-full disables recording instead of corrupting (`chatRecordingService.ts:47-51`). Resume paths in
   `cli/src/utils/sessionUtils.ts`, `useSessionResume.ts`. History repair on resume pairs with the
   unanswered-tool-call closure above.
6. **Model availability / fallback layer** (`availability/`, `fallback/`): error classification,
   `modelAvailabilityService`, policy catalog, auto-routing fallback wired into retry config
   (`geminiChat.ts:988` `availabilityMaxAttempts`) — quota errors route to fallback models instead of
   failing the prompt.
7. **OS-native process sandbox in addition to container mode.** New `sandbox/` managers: macOS Seatbelt
   strict allowlist `(deny default)` importing Apple's base profile, deny-by-default file read/write,
   process-exec, mach-lookup (`sandbox/macos/baseProfile.ts:15,164,180,200`), Linux bubblewrap
   (`sandbox/linux/bwrapArgsBuilder.ts`), plus the older docker/podman execution modes
   (`cli/src/utils/sandbox.ts`, `Dockerfile`, published sandbox image `config.sandboxImageUri` in root
   `package.json`). `services/sandboxManager.ts` resolves workspace/allowed paths with symlink expansion.
   This replaces the devcontainer-only posture described in older audits.
8. **MCP done wide**: tools, prompts and resources all discovered into registries
   (`tools/mcp-client.ts:239-245`), OAuth provider + token storage (`mcp/oauth-provider.ts`,
   `mcp/oauth-provider.ts:59-61`), MCP prompts surfaced as slash commands (`prompts/mcp-prompts.ts`),
   resource reading as tools (`tools/read-mcp-resource.ts`).
9. **Shell**: node-pty first with child_process fallback (`services/shellExecutionService.ts:463-475`),
   inactivity-timeout aborts, `is_background` param moves commands to background with PID-capturing
   subshell wrapper (`tools/shell.ts:97,119`), background control tools in `tools/shellBackgroundTools.ts`;
   `executionLifecycleService.ts` unifies lifecycle for shell + background agents.
10. **Subagents + skills + A2A**: `agents/` has a registry, loader with acknowledgement gating
    (`acknowledgedAgents.ts`), local and **remote** subagent invocations over an A2A protocol
    (`local-subagent-protocol.ts`, `remote-invocation.ts`), built-in `codebase-investigator`,
    `generalist-agent`, and a `skill-extraction-agent` that mines sessions for new skills
    (`agents/skill-extraction-agent.ts`); skills loaded from disk and activated via a tool
    (`skills/skillManager.ts`, `tools/activate-skill.ts`).
11. **Settings**: system/user/workspace hierarchy in `cli/src/config/settings.ts` with a generated JSON
    schema and generated docs (`scripts/generate-settings-schema.ts`, `docs:settings` in root
    `package.json`); extensions system (`config/extensions`).
12. **SDK**: `packages/sdk/src` wraps core with `GeminiCliAgent` / `session()` / `resumeSession()`,
    custom `tool()` with zod schemas, dynamic instructions functions, and its own fs/shell overrides
    (`sdk/index.ts`, `SDK_DESIGN.md` — hooks/skills/subagents/ACP explicitly not yet exposed).

## Gemini CLI → qwen-code lineage (file-name evidence)

The names the Antigravity/qwen lineage should carry were checked directly against the local qwen-code
clone (head `6212189`, 2026-10-13). qwen-code still ships the **original gemini-cli file layout**:
`packages/core/src/core/coreToolScheduler.ts`, `core/geminiChat.ts`, `core/turn.ts`,
`core/baseLlmClient.ts`, `core/contentGenerator.ts`, `core/geminiRequest.ts`,
`utils/nextSpeakerChecker.ts`, `services/loopDetectionService.ts`, `services/shellExecutionService.ts`,
`services/chatRecordingService.ts`. Gemini CLI at this head has since renamed/refactored the same
components (`scheduler/scheduler.ts`, `context/`, `sandbox/`), so qwen-code forked an older gemini-cli
and diverged (adding openai/anthropic content generators, its own scheduler tests), while Gemini CLI
kept evolving the shared skeleton. Verdict: **source-verified ancestry** by naming, structure, and
component-for-component correspondence.

## Strengths

- Loop hardening is the deepest of any audited harness: two-tier loop detection with *recovery* rather
  than abort, mid-stream retry separation, history self-repair for orphaned tool calls.
- Context management is measurement-driven (API-reported token ground truth, not estimates) and split
  into composable processors with a disk-backed output spill — the most principled compaction system in
  the survey set besides Zero's own V1 machinery.
- Checkpoint/restore is simple and real (shadow git), and resume is patch-aware, not just replay.
- Sandbox has moved from "container or nothing" to OS-native deny-default profiles.
- The SDK + a2a-server show a credible path from CLI to embeddable harness.

## Weaknesses

- Two parallel context systems (legacy `ChatCompressionService` + new `contextManager` pipeline) with a
  feature flag between them — same "two harnesses, one shipping" pattern as Zero Code.
- Default execution is still a per-process async generator: no durable admission, no cross-process
  resume of an *in-flight* prompt; JSONL recording is UI/replay state, not execution state.
- nextSpeaker continuation pays an extra LLM call and was defaulted off (`config.ts:1288`) — a
  documented-in-name technique that stopped being worth its cost.
- Edit cascade fuzzy tier is behind a global constant, and nothing enforces read-before-edit
  (same semantic-corruption risk profile as Zero V1 tier 6).
- Core is a god-object (`config/config.ts` ~4k lines, `GeminiChat` 2,093, `client.ts` 1,329);
  heavy telemetry coupling inside loop code.

## Works vs unproven

- **Source-verified working** (implemented + wired): loop-detect recovery, edit cascade, mid-stream
  retries, context pipeline + token ground truth, shadow-git checkpoints, JSONL rewind records, MCP
  tools/prompts/resources, seatbelt/bwrap sandbox, scheduler state machine, availability fallback.
- **Implemented but off-by-default / partial**: nextSpeaker check (off), hooks (flagged,
  `config.getEnableHooks()`), new context pipeline vs legacy compression, SDK exposure of
  skills/subagents/hooks (explicitly missing, `SDK_DESIGN.md`). Fuzzy edit recovery is on by default but
  hardcoded (`edit.ts:68`), not user-configurable.
- **Documented-only / unverified here**: container sandbox runtime behavior (image not executed),
  a2a-server durability claims, session-summary quality, Antigravity-specific differences (separate audit).

## What Zero Code should adopt (independent of Antigravity findings)

1. **Token-ground-truth context triggering**: feed API-reported `promptTokenCount` back into a deficit
   model and trigger compaction/masking on measured crossings (`core/client.ts:859-867`,
   `context/contextManager.ts:377-418`). Zero estimates; this eliminates estimator drift and the
   compaction-that-doesn't-compact class of bugs.
2. **Loop detection with a bounded recovery turn**: heuristic fast path + rare LLM double-check with
   confidence-scaled interval, then one injected "take a step back" feedback instead of abort
   (`loopDetectionService.ts:41-64`, `core/client.ts:1300`). Zero V1 has doom-loop redirection only;
   V2 has neither.
3. **Unanswered-tool-call history repair at admission time** (`geminiChat.ts:528-535, 658-660`): close
   orphaned tool calls/responses with synthetic turns *before* recording the new user message, once per
   send outside retries — directly relevant to Zero V2's promote-at-safe-boundary design.
4. **Tool-output masking to disk with a protection window**
   (`context/toolOutputMaskingService.ts:25-41`): newer-than-50k-tokens protected, older bulky outputs
   replaced by `tool_output_masked` file references in batches ≥30k saved, high-signal tools exempt.
   This is a cheaper first lever than summarization and composes with Zero's spill files.
5. **Approval-time tool-modification and a per-call status machine in the scheduler**
   (`scheduler/state-manager.ts`, `scheduler/tool-modifier.ts`): making Validating/AwaitingApproval/
   Executing explicit statuses (not vibes) is exactly what Zero needs for durable execution status
   (baseline finding 3).

## Classification

All `file:line` claims above were read in source at head `42c51822084a` (source-verified). The qwen-code
ancestry is source-verified by file correspondence at qwen-code `6212189`. Runtime behavior (sandbox
enforcement, fallback routing under real quota, SDK usability) was not executed: documented-only.
