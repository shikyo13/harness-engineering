# 02 — Pi (earendil-works/pi) competitor audit

Audited 2026-10-13 at head `3daa20280f6a` ("fix(coding-agent): keep codemode cancellation prompt and bounded").
Repo: `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/pi`. **Shallow clone (1 grafted commit)** —
file history and branch list are unavailable, so §7 is reconstructed from code, commit message, and issue ref (#10748), not `git log`.
All `file:line` refs are relative to `packages/`. Classification: **[V]** = source-verified, **[T]** = talk-claimed-only
(`docs/research/2026-09-15-harness-and-engine/pi-armin-ronacher-transcript.md`), **[V→T]** = talk claim that source confirms.

## 1. Architecture

Monorepo layout (actual, from `packages/`): `ai` (`@earendil-works/pi-ai`, provider abstraction),
`agent` (`@earendil-works/pi-agent-core`, 2,519 LOC of loop+state in `agent/src`), `coding-agent` (the `pi` CLI:
TUI, session tree, compaction, tools, extensions, RPC/print modes), `codemode` (QuickJS sandbox, dep `quickjs-wasi@3.6.2`,
`codemode/package.json:70`), plus `durable` (experimental durable harness, `durable/README.md:5-7`), `chord` (document state),
`evals`, `mcp`, `tui`, `env`, `telemetry`, `pi-messages` server API. [V]

Layering: `agent` depends only on `ai` + typebox (`agent/package.json` deps). `coding-agent` layers everything else —
session tree, compaction, tool *definitions*, extensions, modes — on `agent`. The core loop has zero knowledge of
sessions, JSONL, compaction, or TUI. This is the "harness separates model decisions from host execution" claim made concrete:
the loop is a pure function over `AgentContext` + a config of injected callbacks (§2). [V]

The pivotal abstraction is `AgentMessage` = LLM messages `∪` app messages via declaration merging
(`agent/src/types.ts:352-374`). The loop runs entirely on `AgentMessage[]` and transforms to provider `Message[]` only at
the LLM-call boundary via mandatory `convertToLlm` (`agent/src/agent-loop.ts:394-395`, contract `types.ts:196-222`),
with optional `transformContext` before it (`types.ts:244`). Custom roles (compactionSummary, branchSummary, custom,
bash-execution) never reach the provider unless a converter maps them (`coding-agent/src/core/messages.ts:100-117`). [V]

**System prompt and tool set are transcript state, not request fields.** The loop passes a "normalized transcript" where
system messages inside the transcript carry the prompt and tool declarations; `context.systemPrompt`/`context.tools` are
never sent (`agent/src/types.ts:22-26`). When the executable tool set diverges from what the transcript declared, the loop
synthesizes a system message with `toolsAdded`/`toolsRemoved` deltas before the next request so replay always converges to
`context.tools` (`agent-loop.ts:323-375` `declareToolChanges`). This is deferred-tool-loading infrastructure and it is real. [V→T]

## 2. The agent loop (`agent/src/agent-loop.ts`, 949 lines)

Entry points: `agentLoop(prompt…)` / `agentLoopContinue(context…)` (`:38,:71`) both funnel into one `runLoop`
(`:163-321`). `agentLoopContinue` requires the last message to not be `assistant` (`:81`) — the retry/resume primitive.
Returns an `EventStream<AgentEvent, AgentMessage[]>` resolving on `agent_end` (`:153-158`).

Loop shape: **outer loop = follow-up drain, inner loop = tool/steering turns** (`:179-318`).
Per iteration: (1) `prepareNextTurn` hook may swap context/model/thinking between turns — this is where long-running
compaction lives, with a second steering poll after it returns (`:185-206`); (2) queued messages appended with
`message_start/end` events (`:211-216`); (3) `prepareRequest` hook gets a last look at context/model per request (`:219-239`);
(4) `streamAssistantResponse` streams and mutates the context tail with the partial message, replacing it in place on each
delta (`:381-469`); (5) `error`/`aborted` stop reasons are hard exits (`:245-256`); (6) tool calls execute; (7) `finishTurn`
hook returns `{action:"end"|"continue"|void}` — `continue` guarantees exactly one more provider request even with no tool
calls or queued input (`:286-314`, `types.ts:146-158`).

**Steering/interrupt API is poll-based, not signal-based.** The config exposes `getSteeringMessages` (mid-run injection
after the current turn's tools settle) and `getFollowUpMessages` (only when the agent would otherwise stop)
(`types.ts:283-306`; polled at `agent-loop.ts:176, 205, 295, 302`). The `Agent` class wires these to two
`PendingMessageQueue`s defaulting to `one-at-a-time` drain (`agent/src/agent.ts:247-248`), with `steer()`, `followUp()`,
`clear*Queue()`, `peekQueuedMessages()` (`agent.ts:299-330`) and `abort()` via per-run `AbortController` (`agent.ts:341-343`).
Steering is never mid-tool-batch: current tool calls always settle first. [V]

**Tool settlement.** Calls parsed from `message.content.filter(c => c.type === "toolCall")` (`:259`). Mode: parallel by
default (`agent.ts:253`); any `executionMode:"sequential"` tool forces whole-batch sequential (`agent-loop.ts:516-522`).
Parallel path: sequential *prepare* (lookup → `prepareArguments` → schema validation → `beforeToolCall` gate), then
concurrent execute (`:586-660`). `beforeToolCall` returning `{block:true}` converts to an error tool result without
execution (`:745-755`); `afterToolCall` can field-wise override content/details/isError/usage/terminate (`:859-910`).
Early termination only when *every* finalized result sets `terminate` (`:690-692`). Tool throws → `isError` result; nothing
rejects (`:821-857`, contract `:808-809`). Duration measured with monotonic clock (`:828-841`). `runToolCall` is exported
so tools-that-call-tools (codemode) run nested calls through the identical hook pipeline (`:790-819`); coding-agent wires
nested calls with `parentToolCallId` provenance (`agent-session.ts:744-760`). [V]

**Length-guard (the one Zero Code ported).** If `stopReason === "length"`, *all* tool calls in that message fail with a
"re-issue with complete arguments" error instead of executing, because streamed args are finalized by a best-effort JSON
salvage parser and may be silently truncated (`agent-loop.ts:264-270, 478-503`; salvage = `partial-json` + `repairJson`,
`ai/src/utils/json-parse.ts:1,32,117`). [V]

**Event model:** `agent_start/end`, `turn_start/end`, `message_start/update/end`, `tool_execution_start/update/end`
(`types.ts:516-539`). Tool partials stream via an `onUpdate` callback whose post-settle calls are dropped (`:832-843`).
Listeners are awaited and part of run settlement (`agent.ts:250-268`). [V]

## 3. Toolset: four tools, deliberately

Default set is **read, bash, edit, write** (`coding-agent/src/core/tools/index.ts:164-171`; system-prompt default
`selectedTools` `system-prompt.ts:69`). `grep`, `find`, `ls`, `powershell` exist as first-class definitions but are not
default; when absent, the prompt instead tells the model "use bash for file operations like ls, rg, find"
(`system-prompt.ts:103-118`). So "pi just gives you bash" [T] is accurate for the default prompt; the read-only tools are
an opt-in alternative set (`createReadOnlyToolDefinitions` `tools/index.ts:173-180`). [V→T]

Schemas are tiny: `bash {command, timeout?}` (`bash.ts:38-41`, no default timeout); `read {path, offset?, limit?}` with
2000-line/50KB head truncation and images-as-attachments (`read.ts:94-97`, `truncate.ts:10-12`); `write {path, content}`;
`edit {path, edits:[{oldText,newText}]}` (`edit.ts:21-41`). No todo, no task/subagent tool, no web fetch, no plan mode,
no read-before-edit enforcement (grep of `write.ts` finds none), no permission prompts (§5). Absent-by-design:
the entire "orchestration" surface lives in extensions (todo/task patterns are prompt-template/extension examples).

**Bash output discipline:** dual limits (2000 lines OR 50KB, whichever first), never partial lines except tail edge
(`truncate.ts:1-13`); truncated output spills to a file whose path is printed into the model-facing text
("Full output: <path>", `bash.ts:348-352`); structured 1MB `output` for programmatic callers like codemode (`bash.ts:22-59`).
`grep`/`find` spawn `rg`/`fd` with auto-download via `ensureTool` (`grep.ts:119-121`, `find.ts:171-172`). [V]

**Edit match strategy** (`edit-diff.ts`): normalize (strip BOM, CRLF→LF) → per-edit exact `indexOf` first
(`:207-218`) → on miss, fuzzy space: NFKC + per-line trimEnd + smart quotes/dashes/unicode spaces → ASCII
(`normalizeForFuzzyMatch` `:34-55`) → uniqueness enforced *in fuzzy space* (`countOccurrences` `:247-251`, error "must be
unique" `:328-331`) → overlap between edits rejected (`:341-350`) → if any edit matched fuzzily, replacements are computed
in normalized space but written back line-block-by-line-block so untouched lines keep their original bytes
(`applyReplacementsPreservingUnchangedLines` `:132-173`) → identical-output throws (`:357-359`). Multi-edit is matched
against the *original* file, reverse-applied by offset (`:111-120`). Argument repair `prepareArguments` handles models that
stringify the `edits` array or send legacy top-level `oldText/newText` — comment names Opus 4.6 and GLM-5.1
(`edit.ts:103-134`). Per-path `withFileMutationQueue` serializes concurrent edits (`edit.ts:163`, abort-safe lock comment
`:164-170`). This is a *two-tier* matcher (exact + normalization), notably simpler than Zero Code's 9-stage cascade with
65%-similarity tiers — pi never does similarity matching, trading recall for predictability. [V]

## 4. Session storage: JSONL tree + compaction

`SessionManager` (`coding-agent/src/core/session-manager.ts`, 2,013 lines): append-only JSONL under
`~/.pi/agent/sessions/--<encoded-cwd>--/` (`:585-593`), header `{version:3, id, cwd, parentSession?}` (`:41-50`), every
entry `{type, id, parentId, timestamp}` (`:57-62`) — entries form a tree; "branching moves the leaf to an earlier entry,
allowing new branches without rewriting history" (`:979-981`). Entry types: `message`, `thinking_level_change`,
`model_change`, `usage`, `compaction`, `branch_summary`, `custom`, `context_edit` (branch-local edits to earlier
model-visible entries, `:1359-1391`). Context = walk leaf→root (`buildSessionPath` `:390-415`), then apply the latest
compaction as a cut: `[compaction] + kept-since-firstKeptEntryId + everything-after` (`buildContextEntries` `:474-521`).
Model/thinking level are *derived from the path* (last `model_change` or assistant message) — replayable config state
(`:418-433`). Fork = new file with `parentSession` header (`:49`, CLI `--fork`, `cli/args.ts:138`). Portability is the
file itself: plain JSONL + partial-UUID resume + HTML export (`--session <path|id>` `args.ts:134`, `export_html` RPC
`rpc-types.ts:60`). No DB, no index. [V]

**Compaction — pi does compact, threshold-based, in-session.** Defaults: enabled, `reserveTokens 16384`,
`keepRecentTokens 20000` (`compaction/compaction.ts:126-131`); trigger `contextTokens > window - reserve` using real
provider usage when available, chars/4 estimate otherwise (`:267-270`, `:322-325`, agent-side check
`agent-session.ts:3095-3113`), plus an "overflow" path that compacts and *retries the interrupted turn*
(`_runAutoCompaction(reason, willRetry)` `agent-session.ts:3120-3130`). Cut point walks backwards accumulating estimated
tokens to `keepRecentTokens`, cutting only at user-or-assistant messages, never tool results; may split a turn
(`findCutPoint` `compaction.ts:428-470`). Summary is produced by a dedicated summarization system prompt ("structured
summary, do NOT continue the conversation", `compaction/utils.ts:161-164`) through `retryAssistantCall` with backoff
(`compaction.ts:636-640`; `ai/src/utils/retry.ts:113-192`), incrementally **merged with the previous summary** on
re-compaction (`generateSummary(… previousSummary …)` `compaction.ts:642-666`). The compaction entry also stores the
`systemMessage` snapshot at the boundary (full prompt/tool state replay, `session-manager.ts:83-100`) and
`readFiles/modifiedFiles` carried forward across successive compactions (`compaction.ts:49-83`). Summary re-enters context
as custom role `compactionSummary` (`messages.ts:109-117`). [V]

**Branch summarization**: navigating the tree away from a branch generates a summary of the abandoned branch, stored as a
`branch_summary` entry and projected as a message on the new path (`compaction/branch-summarization.ts:1-8`,
`session-manager.ts:456-459`). Tree navigation is a first-class citizen (`get_tree` RPC, tree selector tests). [V]

**Prompt-cache economics** go beyond compaction: a `cache-warmer` refreshes Anthropic-style cache entries at 90% of TTL
while idle, only when expected savings ≥ $0.05, using a 0.15 "idle continuation probability" constant "measured from our
own usage" (`cache-warmer.ts:15-31`). TTLs come from per-model `promptCache` tiers with `cacheRetention: short|long|none`
(`cache-warmer.ts:39-46`). I saw nothing equivalent in the Zero baseline notes — Zero froze prefixes for stability; pi
additionally *pays to keep them warm*. [V]

## 5. Provider abstraction (pi-ai), extensions, RPC, permissions

**pi-ai unified format:** every API implements the same SSE-ish event contract → `AssistantMessageEvent` stream
(`text_start/delta/end`, `thinking_*`, `toolcall_*`, `start/done/error`) yielding one `AssistantMessage` with
`stopReason ∈ stop|length|toolUse|error|aborted|deferred` and normalized `Usage` (`ai/src/types.ts:446-469`; consumed at
`agent-loop.ts:414-457`). ~40 providers (`ai/src/providers/`), each with a `.models.ts` catalog. StreamFn contract:
*must not throw*; failures encoded in-stream (`agent/src/types.ts:27-37`). Quirk handling lives in per-API shims: image
downgrade to text placeholders for non-vision models (`api/transform-messages.ts:13-14, 37+`), OpenAI `prompt_cache_key`
64-char clamp (`api/openai-prompt-cache.ts:1-9`), Anthropic `cache_control: ephemeral(+ttl)` on last system/user block
and — only behind a per-model `supportsCacheControlOnTools` compat flag — on the last tool (`anthropic-messages.ts:1164-1255,
1528-1549, 1658`), strict-JSON-schema keyword pre-validation with graceful degradation
(`api/constrained-sampling.ts`; tools opt in via `constrainedSampling: {strict:"prefer"}`, e.g. `edit.ts:156`). There is
even a first-party wire protocol (`api/pi-messages.ts:1-10`, Radius gateway) and local **classifier** APIs
(`llama-cpp-classify` — small-model JSON answering used by codemode's `models.*`) [V].

**Extension model:** coding-agent's `AgentSession` installs the loop's `beforeToolCall`/`afterToolCall` once and routes
them to extension `tool_call`/`tool_result` hooks — so *all* interception (permissions, rewriting, budgeting) is extension
space, and nested codemode calls inherit the same pipeline (`agent-session.ts:654-680, 744-760`). The `ExtensionAPI` is
event-rich: session lifecycle, compaction start/failed, tree changes, context transforms (`context`,
`context_with_system`), provider headers/response/stream interception, agent lifecycle, custom UI prompts
(`core/extensions/types.ts:1592-1650`). Tools register with `defaultActive`, exposure class
`direct|model-only|codemode|deferred|hidden` (`types.ts:510-516`), and `prepareLoadout` which rewrites what the model
sees per request (`types.ts:629-638`) — this is the deferred/MCP-scaling story, paired with the inactive-by-default
`tool_search` and `codemode` extensions (`extensions/tool-search/index.ts:12-17`, `extensions/codemode/index.ts:33-47`).

**Headless:** RPC mode is a JSONL stdin/stdout protocol: `prompt|steer|follow_up|abort|clear_queue|fork|clone|
switch_session|compact|set_auto_compaction|set_auto_retry|bash|export_html|get_tree|get_entries|…`
(`modes/rpc/rpc-types.ts:22-74`) plus print mode. The TUI is just another client of `AgentSession`. [V]

**Permission posture: pi does not ask.** There is no built-in approval prompt, no yolo flag, no allowlist in core
(grep `permission|approve|yolo` finds only extension hooks and MCP plumbing). Default = full access; a blocking policy is
something an *extension* implements via `tool_call` → `{block, reason}` (`agent/src/types.ts:66-74`,
`agent-session.ts:660-680`; extension failure blocks execution — `agent-session.ts:677-679`). Project trust
(`project-trust.ts`, `project_trust` event) gates *extension loading*, not tool execution. This is a philosophical
baseline: autonomy by default, guardrails as composable code. [V]

## 6. codemode (head-commit subsystem)

`codemode` = a tool where the **model writes raw JavaScript that runs in a QuickJS sandbox** (`quickjs-wasi`, worker
thread; `packages/codemode/src/runtime/execution.ts:6`, `vm.ts`, `worker.ts`) and calls the agent's tools as
`await tools.<name>({...})` (`extensions/codemode/tool.ts:1-20`). Rules from the tool header: only the script's returned
value reaches the model — nested tool results do not enter context unless the script emits them; tools with
`outputSchema` resolve to typed `structuredContent`; failed calls reject as `Error`; results carry a
"Script completed/failed" header; `store()/load()` persist JSON across calls as branch-scoped `codemode-store` session
entries (`tool.ts:9-19`, `index.ts:35-45`). Nested calls go through the real loop pipeline (validation + hooks +
permissions) with `parentToolCallId` recorded on the result for file-op tracking (`tool.ts:6-8`,
`compaction/utils.ts:29-33`). Script-visible extras: `models.*` catalog/classifiers/image-gen, `exit()`, `// @options:`
line. `codemode.mode: on|only` setting changes how the tool presents the loadout; a token **budget** selects which tool
declarations fit in the description ("like OpenCode's catalog", `tool.ts:198-207`). Infinite-loop protection: QuickJS
interrupt handler with an **interrupt budget** → `kind:"timeout"` failure (`codemode/src/runtime/inline.ts:11-24, 94`).
The head commit ("keep codemode cancellation prompt and bounded", #10748) is about the `cancelled` nested-call status
(`CodemodeNestedCallStatus` `tool.ts:86`; "calls still running when the script ends are cancelled" `tool.ts:119`) staying
described in the prompt and bounded on abort — details beyond the grafted commit are [T-adjacent, unverifiable here].
This is the same idea Zero Code's experimental code-mode pursues, but shipped, sandboxed (QuickJS vs Zero's approach), and
wired into compaction/tracking. [V]

## 7. Recent work (limited by shallow clone)

Only one commit visible, so recency comes from feature surface: codemode (active development, issue-tracked),
tool-exposure/`prepareLoadout` deferred-loadout machinery, `pi-durable` (explicitly experimental durable-execution harness
where "conversations, model turns, tool calls … committed to storage before anything is shown", `durable/README.md:5-7` —
same problem space as Zero's V2 `session_input` admission), the `compat.ts` shim being deleted in a stated "ModelManager
migration" (`ai/src/compat.ts:1-10`), and behavioral evals incl. **documentation-lift evals** (with/without docs in Docker
containers, measuring prompt-docs AUC — `evals/README.md:8-24`). Evals are run against real providers (`PI_PROVIDER`,
`PI_MODEL`), i.e. they regression-test harness behavior end-to-end. [V]

## 8. Weak/local models; what the minimalism costs

Evidence pi is *used* with cheap/open models: per-provider default model map picks kimi-k2.6 / GLM-5.2 / MiniMax-M2.7 for
20+ providers (`core/model-resolver.ts:33-53`); model-specific argument repairs name GLM-5.1 (`edit.ts:110`). Works with
weak models (source-verified defenses): strict-but-fallback constrained sampling; `prepareArguments` shape repairs;
partial-JSON salvage; length-stop tool-call fail-fast; uniqueness/duplicate feedback errors that teach the model to add
context (`edit-diff.ts:264-273`); output truncation with spill paths; non-vision image placeholders; retry-with-backoff;
fuzzy Unicode-tolerant edit matching. What the minimalism *costs*: no doom-loop detection anywhere (grep "doom" = only a
DOOM-player example extension), so a weak model ping-ponging identical failing edits loops until the user aborts — Zero
Code's doom-loop redirection has no pi counterpart; no read-before-edit enforcement, so old-text hallucinations surface
only as match failures (cheap, but a wasted turn each); no similarity fallback, so whitespace-garbling models fail edits
exact+fuzzy can't (Zero's tier-65% matcher would land); no automatic context *shaping* beyond compaction (no stable-index
pressure notes, no repo map); no subagents in core. [V]

**Talk claims checked:** "pi gives you basically bash" [V→T] (default 4-tool set, bash-rules fallback
`system-prompt.ts:103-118`); "interleaved system messages enable deferred tool loading" [V→T] (transcript-native system
messages + `declareToolChanges` + `deferred` exposure + tool_search); "harnesses converged on bash/pipeline-style
context efficiency" [T, out of scope]; "Pi is beating Claude Code/Codex on benchmarks" — **unverifiable in-repo**
(no leaderboard artifacts; evals are behavioral smoke/docs-lift only) [T].

## 9. Beyond what Zero Code already ported — learn / avoid

Zero Code already has pi-derived: tool-alias repair (superset of pi's `prepareArguments` repairs — pi has *no* name-alias
repair, only arg-shape repair), length-guard (pi's is per-message all-calls-fail, `agent-loop.ts:264-270` — same design),
doom-loop (pi lacks it entirely).

**Genuinely beyond Zero Code's current ports, worth learning:**
1. **Transcript-native prompt/tool state with replayable deltas** (`declareToolChanges` `agent-loop.ts:323-375`,
   `ToolExposure`, `prepareLoadout`): one mechanism gives deferred tool loading, mid-session tool swaps, replay-stable
   audits, and cache-stable prefixes. Zero's four overlapping "reach the right tool" layers (baseline finding #6) are what
   one declarative loadout delta replaces.
2. **`finishTurn`/`prepareNextTurn`/`prepareRequest` as the extension grammar of the loop** (`types.ts:146-281`):
   compaction, model routing, retry, and continuation all live outside the loop; the loop stays 300 readable lines.
   Zero's V2 should adopt this hook vocabulary rather than growing `SessionRunner`.
3. **Durable tree sessions as the only artifact**: no DB, leaf-pointer branching, branch summaries on abandonment,
   `context_edit` branch-local corrections, compaction entries carrying their own system-message snapshot
   (`session-manager.ts:57-130, 474-521`, `compaction.ts` + `session-manager.ts:83-100`). Directly compatible with Zero
   V2's durable-input direction, and cheaper to operate.
4. **Incremental summary merging + file-ops ledger across compactions** (`compaction.ts:49-83, 642-666`) — keeps the
   read/modified-file set alive through N compactions; Zero's cache-continuation summaries should verify they carry this.
5. **Cache warmer with explicit economics** (TTL-refresh at 90%, $0.05 expected-savings floor, measured continuation
   probability `cache-warmer.ts:15-31`): a quantified, modelable approach to Anthropic TTL gaps.
6. **Documentation-lift evals** (`evals/README.md`): A/B the system prompt/docs themselves in CI — Zero's prompt fixes
   currently ship unmeasured.
7. **Nested-call discipline**: tools calling tools must reuse the loop pipeline verbatim (`runToolCall`
   `agent-loop.ts:790-819`) and record `parentToolCallId` for tracking/permissions. Zero's code-mode should mirror this
   invariant exactly.

**Avoid:** pi's *no-permission-by-default* posture (fine for a personal CLI, wrong for a product with unattended
sessions); absence of any stall/doom detection; absence of name-alias repair for weaker callers (pi relies on strict
sampling that local servers often ignore); and single-file JSONL growth without pruning for very long-lived trees.

**Bottom line:** pi's moat is not the minimal toolset Zero already copied — it is (a) the transcript-as-source-of-truth
loadout system and (b) the tree-session/compaction substrate with cache economics, both implemented with radically less
code than either Zero V1 or V2 carries today.
