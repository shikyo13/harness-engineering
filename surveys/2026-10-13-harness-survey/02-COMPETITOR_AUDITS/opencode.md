# 02 — Competitor audit: OpenCode (anomalyco/opencode)

Head audited: `3ec856b6c265` (2026-10-14, "feat(desktop): refresh question composer (#54822)"), branch `dev` (repo default).
Origin `https://github.com/anomalyco/opencode`; `origin/dev` already ~1 commit ahead (`6a7cedc`). ~2,069 remote branches
(99% short-lived feature branches); named candidates for more advanced work: `dev` (default, where all V2 work lands),
`2.0`, `beta` (release lines; contents not fetch-inspected — cheap `ls-remote` only). Note: the historical
`packages/opencode` layout is gone; the harness now lives in `packages/core` (+`packages/ai`, `packages/codemode`,
`packages/server`, `packages/tui`). Every claim below is source-verified at file:line unless tagged **documented-only**.
This repo is the direct ancestor of Zero Code's V2 design; the V2 vocabulary in Zero Code's AGENTS.md is *this* code.

## Architecture

Bun + Effect (generators, Layer, Scope, Fiber) end-to-end. Monorepo: `core` (agent loop, tools, session), `ai`
(hand-rolled provider protocol layer — no Vercel AI SDK at runtime), `codemode` (custom JS interpreter + tool
runtime), `server` (Effect HttpApi, SSE event feed, PTY sockets), `tui` (SolidJS on @opentui), `plugin` (public
plugin API), `client`/`sdk` (generated), `desktop`/`web`/`console`/`enterprise` shells.

Layering: global nodes (Session, DB, execution) → Location-scoped nodes (tools, context, compaction, MCP, per
directory/workspace) via `LayerNode`/`makeLocationNode` (`core/src/session/context.ts:71-73,188`). A session is bound
to a `Location` and requests are executed through `Instance.Service.provide(session)` which selects the location's
layer set (`core/src/session/execution.ts:95-98`, `core/src/instance/service.ts:10-17`).

Storage is event-sourced SQLite: every mutation is a durable event on a per-session aggregate with monotonically
reserved seq (`core/src/bus.ts:21-45`); `SessionProjector` folds events into Drizzle tables (`session/projector.ts`,
`session/sql.ts`). Messages, pending inbox rows, compactions, instruction state, permission grants, event log all
live in one DB (`opencode.db`, `cli/src/database-path.ts:10`). Clients (TUI/desktop/web) are thin: they call the HTTP
API and follow SSE (`server/src/handlers/event.ts:22-25`, `server/src/event-feed.ts`), with durable replay from the
bus log by seq (`core/src/session/session.ts` via `Session.log`, `core/src/session.ts:405-415`).

## Agent loop (source-verified)

- **Entry**: `POST session.prompt` → `Session.prompt` admits a durable inbox row then *returns immediately*;
  execution is an advisory wake (`core/src/session/session.ts:145-178`, wake at :174; handler
  `server/src/handlers/session.ts:307-330`). Contrast: Zero Code's V2 HTTP entry blocks for the whole drain.
- **Admission/durable inbox**: `core/src/session/inbox.ts`. Typed items `user|synthetic|compaction|move`, delivery
  `steer|queue` (`:43-48`). `admit` is idempotent-reconciling on message ID (first-writer-wins, `:155-202`); rows are
  pending-only — promotion deletes the row and inserts the visible message in one event transaction
  (`projectDelivered :324-336`). `promote` (`:496-531`): all steers at a step boundary; exactly one queued item only
  at the idle boundary; compaction controls jump ahead of earlier steers but never across a `move` (`:541-548`).
- **Coordinator**: `core/src/session/run-coordinator.ts:60-196`. One fiber per session ("busy period"), doorbell
  `pendingWake` coalescing wakes (scope lattice: `input` subsumes `steer`, `:136-145`), trampoline re-drain
  (`:74-85`), join-on-run, interrupt claims recorded wakes but wakes arriving during cleanup start a successor
  (`:147-167`). Process-local only — no cluster ownership yet.
- **Crash continuity**: write-ahead *claim* event committed with Execution.Started; terminals release it, shutdown
  interruption *keeps* it so the next server start resumes the turn ("recovery is a property of the database, never
  of a shutdown hook", `execution.ts:76-86,129-134`). On drain start, orphaned running compactions and
  streaming/running tool calls are settled as interrupted before any provider call
  (`runner/llm.ts:332-385`). This is the durable execution status Zero Code's baseline says is missing.
- **Drain loop**: `runner/llm.ts:57-199`, a `while(true)` over `advanceToStep` (durable boundary: promote inbox,
  handle compaction/move control items) + `runStep` (provider attempts). Not a state machine object; explicit
  tagged `DrainResult` (`Complete|Moved|Reloaded`) re-entered recursively (`execution.ts:105-109`).
- **Step attempt**: `runner/step.ts:73-286`. Exactly one `llm.stream(...)` per attempt, read to end (`:107-137`).
  **Parallel tools**: each `tool-call` event forks an uninterruptible-mask fiber immediately *while the stream is
  still open* (`:122-134`); join+classification after stream close (`:151-153, :298-321`). Provider order preserved
  without a shared writer. Final event settlement is uninterruptible (`:141`).
- **Continuation decision**: `record.needsContinuation` (tool calls present) → `Completed({needsContinuation})`
  (`:281-283`). Outcomes are a tagged enum: `Completed|Retry|Continue|RecoverFull|Compacted|OutputLimit`
  (`:32-42`) — runner maps each: transport fail before output → full retry; fail *after* output → synthetic
  "previous response was interrupted, continue" message (`runner/llm.ts:35-39,281-289`); `stop=length` → continue
  with anti-apology instruction, hard-fail after 3 consecutive length-finishes (`llm.ts:290-319`); provider
  "too long" rejection with nothing output → compact, re-attempt once (`:155-167`, `:265-271`).
- **Limits**: per-agent `steps` cap; at the cap the request is sent with `toolChoice:"none"` plus an assistant-role
  "MAX STEPS REACHED" force-summary message, deliberately keeping tool definitions in the request to preserve the
  provider's cached prefix (`llm.ts:226-246`, `runner/max-steps.ts`). Any tool call then errors "Tools are disabled"
  (`step.ts:92-93`).
- **Retry**: exponential 2s→10s ×10 with jitter, honors provider `retry-after` bounded at 15min, timeout retries
  capped at 3 (`runner/retry.ts:31-59`); retry decisions are durable events and plugin-tunable via the `retry` hook.

## Tools and dispatch

- Registry: location-scoped `State`-transformable editor (plugins add/update/remove), permission-filtered snapshot
  per step, alias/repair hooks (`core/src/tool.ts:225-295`). Declared Effect Schema in/out per tool.
- **CodeMode is the dispatch spine.** All tools except `shell` (and anything `codemode:false`) are *not* advertised
  as provider tool definitions; they are exposed as one `execute` tool running JS in a **hand-written AST JS
  interpreter** (`packages/codemode/src/interpreter/` — 16 files: promises, generators, iterators, intrinsics;
  imports/fs/timers banned, `fetch` allowed; `codemode/tool.ts:62-69`). Tool catalog is rendered into Instructions
  and memoized per registry revision+visible-set so the big catalog text stays cache-stable across steps
  (`tool.ts:246-254`). Model writes `tools.read(...)`, composes with `Promise.allSettled`; per-call progress rows are
  streamed to the UI (`codemode/tool.ts:142-176`). `shell`, `subagent`, `question`, `skill` are pinned direct tools.
- **Edit**: exact `oldString`→`newString` + `replaceAll`, with unicode-quote/dash/nbsp normalization, line-wise
  trailing-whitespace-tolerant tier, and LCS-based line-ending preservation for CRLF/mixed files
  (`tool/plugin/edit.ts:49-118`) — a smaller cascade than Zero Code's 9-stage matcher. Separate multi-file `patch`
  tool (`tool/plugin/patch.ts`).
- **Bash**: NOT a persistent shell — each call spawns one command (`core/src/shell.ts:57,254`); `workdir` param per
  call. Timeout default 2min; `background:true` returns instantly with a spill-file path and an explicit
  "DO NOT poll, you will be notified" contract (`tool/plugin/shell.ts:24-61,254-257`); a *foreground* command that
  hits its timeout is **auto-backgrounded** and converted to the same notification contract (`:260-266`). Timeout
  kill keeps the shell alive for capture (`:237` interruption hook). Command string is parsed with a tree-sitter
  WASM bash parser to derive per-command permission resources before execution
  (`shell.ts:129-145`, `#shell-parser-wasm`). Env markers `AGENT=1/OPENCODE=1/AI_AGENT=opencode` (`:213-216`).
  A separate persistent-pty daemon exists for *user* terminals, not the tool.
- **Truncation**: one central post-processing service: 2,000 lines / 50 KiB, spill full output to
  `~/.local/share/.../tool-output/tool_*` with marker `[showing lines 1-N of M; full output saved to <file>]`,
  tool-settable `truncated` metadata to opt out, 7-day retention sweeper hourly
  (`core/src/tool-output.ts:13-15,67-118,129-138`). Image outputs normalized/resized per provider limits, with
  `[N images omitted: reason]` placeholders (`tool.ts:74-101`).
- Tool set: `shell, read, write, edit, patch, grep, glob, webfetch, websearch, question, skill, subagent, models,
  mcp-resource` + `execute` + MCP servers as namespaced tools
  (`tool/plugin/` dir; `tool/mcp.ts:40-46`). Read: line-number-prefixed pages capped at 2,000 lines, images/PDFs
  passed as native media, "Continue reading with offset: N" hint (`tool/plugin/read.ts:19-52,175-208`).
  Grep: bundled ripgrep (`tool/plugin/grep.ts:68-119`).

## Context engineering

- **System prompt is tiny**: 15-line generic base (`session/runner/prompt/system.txt`) + tool-conditional
  guidance rendered only for tools present (`session/system-prompt.ts:9-27`). Model-specific personas are *plugins*
  that override/append by model-id match (gpt/gpt-astra/kimi/meta/trinity/anthropic, 2-97 lines each,
  `plugin/optimize.ts:17-59`) — and two disabled experiments that *delete grep/glob tools for GPT/Claude families*.
- **Instruction stack order is cache-engineered**: user-level → project → date/environment, "most to least shared"
  so the baseline is a reusable prefix (`session/context.ts:146-158`). Instruction files (AGENTS.md etc.) are
  watched and mid-session changes are injected as *diff events* ("New instructions apply from…", "The instructions
  from X no longer apply") rather than silent re-render (`instruction-discovery.ts:85-145`,
  `session/instruction-state.ts`).
- **Prompt caching**: declarative cache policy injecting up to 4 breakpoints (tools + first/last system + message
  tail), skipped entirely for protocols with implicit caching (OpenAI/Gemini) (`ai/src/cache-policy.ts:1-57`);
  `promptCacheKey` = session ID, with subagents/forks inheriting the parent's affinity ID so a family shares one
  cache lineage (`session/affinity.ts:7-9`, `session/model-request.ts:294-295`); affinity also emitted as routing
  headers (`model-request.ts:283-292`).
- **Token budget**: output limit = min(catalog output, 256k, context − measured prompt − 1.15×estimated) with
  4-chars/token estimation +15% error margin, floor 1,024 (`model-request.ts:43-98`). Measured prompt tokens come
  back per response and are persisted for next-step math (`llm.ts:246`). Image payload budget: at >25 MiB inline
  images, drop oldest to 15 MiB, replaced by a "do not claim from memory" notice (`:43-46,189-216`). Media the
  model can't ingest is replaced with an explicit error text telling the model to inform the user (`:132-147`).
- **Compaction** (`session/compaction.ts`, 963 lines): auto at ~90% of window (buffer default 10%, floor
  reserve 16k, `:96-97`); keeps 15k tokens of recent exchanges verbatim beside the summary (`:197`). Structured
  checkpoint template (Objective/Requirements/Decisions/Work State/Next Move/Relevant Files/Important Context,
  `:104-143`) that is *incrementally updated* on re-compaction ("update the existing checkpoint… newer history takes
  precedence"), with a one-time rewrite path for the old bloated template (`:160-175`). Overflow-rejection ladder:
  re-aim at 70%→50%→35% of the rejected estimate, `payload-too-large` first retries text-only (`:95,384-455`).
  If the summary must be re-sent smaller it is *flattened to text* and oldest exchanges dropped, previous summary
  always kept in front (`:457-463`). **Native provider compaction** (`model.compaction.type === "native"`,
  `:324-372`): asks the provider for an encrypted/compact context window (trigger-mechanism or endpoint variant),
  stores it fenced by endpoint *provenance* hash and refuses to replay it at a rerouted endpoint
  (`session/provider-context.ts:18-45`; guard `model-request.ts:321-335`).
- **Session WebSocket transport**: optional per-session persistent WSS to the provider with rotation at 55min,
  30-min idle allowance for reasoning models, and circuit-breaker to HTTP after 5 socket losses
  (`session/model-transport.ts:21-30`).
- Reasoning persistence: reasoning parts keep provider metadata and are only replayed to the producing model/
  endpoint; interrupted reasoning is downgraded to plain text (`runner/to-llm-message.ts:157-213`).

## Session features / multi-client / permissions / subagents

- Per-session durable log stream: replay-after-seq + `Synced` marker + live follow (`session.ts:169-173`), served
  over SSE; PTY over websocket (`server/src/pty-socket.ts`). Multiple clients converge by replay; prompt IDs make
  retries idempotent (`session/session.ts:149-173`).
- Interrupt: fire-and-forget ack semantics ("interrupt resolves on acceptance, cleanup settles on its own time",
  `run-coordinator.ts:147-167`); interrupt-with-resume re-wakes steers/controls but leaves queued prompts parked
  (`execution.ts:153-170`).
- **Revert**: stage (with optional file restore from per-step shadow snapshots)/clear/commit; per-step snapshots
  captured *concurrently with the provider request* so the start snapshot costs no latency (`step.ts:74-76,236-245`);
  `Session.diff` gives structured per-turn file diffs between snapshots (`session.ts:143-152`).
- **Subagents**: child sessions with inherited location/permissions, foreground or background, continuable by
  `sessionID`, completion injected as a durable synthetic message with idempotent notification IDs
  (`tool/plugin/subagent.ts:20-60,96+`; same pattern for background shell, `shell.ts:157-189`). User command
  "background everything" converts blocking jobs to background + synthetic instruction (`session.ts:442-459`).
- **Permissions**: last-match-wildcard ruleset evaluate/ask/assert (`core/src/permission.ts:87-94,166-188`);
  declines throw as *defects* so tools cannot swallow a user "no" (`model-request.ts:400-418`); file access has a
  separate `external_directory` authorization before edit approval (`edit.ts:1-6`); grants persist per project
  (`permission/saved.ts`).
- Fork/export/import: session fork at a message boundary inheriting instruction state (`session.ts:319-358`);
  transferable export/import bundle (`session/transfer.ts:35-45`). Cloud share beyond that: **documented-only**, not
  verified in core.

## Provider abstraction (`packages/ai`)

Hand-rolled: ~50 wire protocols (`src/protocols/`: anthropic-messages, openai-responses (+continuation/channel),
bedrock-converse, gemini, xai-responses, …) behind typed `route/` transports (HTTP/SSE/WebSocket/event-stream),
~70 provider packages. `LLMClient.stream` yields normalized `LLMEvent`s; errors are classified (`provider-error.ts`:
context-overflow detection, retryability, rate-limit with retry-after). Quirks visible as code: xAI image-MIME
allowlist (`model-request.ts:129-131`), per-provider cache-hint capability set (`cache-policy.ts:43-57`), unknown
finish-reason treated as incomplete-stream retry (`step.ts:171-179`). Catalog from models.dev plus `small`-model
resolution for titles with minimal-reasoning variant preference (`session/context.ts:95-119,186`).

## Plugins / MCP / skills

- Public plugin API in two flavors sharing one surface: Effect-native and Promise (adapter `fromPromise`), ~20
  domains (session, tool, permission, provider, model, mcp, skill, shell, vcs, worktree, rpc…)
  (`packages/plugin/src/{effect,promise}/`). Hooks observed in core: `tool.execute.before/after` (input repair,
  result rewriting), `session.context/compaction/generate` (request/tool surgery incl. *renames by object identity*,
  `model-request.ts:246-274`), `model.request` (URL/header rewrite), `http.request/response` (full body access),
  experimental WS frame taps, `retry`, `title`. Plugins install from npm at runtime (`plugin/module.ts:104-109`);
  RPC host for cross-process plugins (`plugin/host.ts:25-63`).
- MCP: official TS client, stdio+http, OAuth (`mcp/oauth.ts`), **elicitation both form and URL modes** wired to
  user interaction (`mcp/client.ts:188,207-212`); server tools registered into the same registry under
  `<server>_<tool>` names, default-routed through CodeMode (`tool/mcp.ts:40-46`). MCP tool list flushes into each
  step's snapshot (`context.ts:127`).
- Skills: `SKILL.md` discovery from configured dirs + npm-published skills (`skill/discovery.ts:112,169-194`),
  `<available_skills>` catalog in instructions, activation as a durable session event with the skill text as
  content (`session/session.ts:223-245`).

## Distinctive strengths

1. **Durable admission + doorbell coordinator + write-ahead claims**: crash/resume semantics designed into the
   loop, not bolted on; stale tool/compaction settlement on restart.
2. **CodeMode with a custom sandboxed JS interpreter** — one `execute` tool replacing dozens of tool definitions,
   cached catalog text, concurrent composition inside the sandbox, per-call progress streaming.
3. **Incremental compaction checkpoints** (update-don't-rewrite) + native provider-compaction windows fenced by
   endpoint provenance + overflow shrink ladder.
4. **Cache engineering everywhere**: instruction ordering by sharedness, 4-breakpoint declarative cache policy,
   family-level promptCacheKey, keeping tool defs on the final capped step, memoized catalog rendering.
5. Timeout→auto-background conversion with push notifications and explicit anti-polling prompt text.
6. Live instruction-file diffing injected into the transcript.
7. Tiny base system prompt with model-family persona plugins, shipped as ordinary plugins.

## Weaknesses

1. **Extreme ceremony/complexity**: ~10.8k lines in `session/` alone; lifecycle conflicts propagate through
   `Effect.die` defect-catching across module boundaries (`inbox.ts:293,316,451-456`) — correct but hostile to
   contribution; Location/Instance/LayerNode graph is hard to reason about. Monorepo sprawl (desktop, console,
   workerd variants, 2k branches).
2. **Two-harness residue again**: `core/src/v1/` config/permission shims and `cli` v1 paths persist alongside V2;
   V2 still has process-local ownership only (no cluster), TODOs for durable cache lineage and workspace replay
   (`model-request.ts:294`, `session.ts:316`).
3. CodeMode correctness surface is a whole JS spec implementation (promises, generators, IteratorClose semantics)
   maintained by the harness team; model-facing contract is brittle (exact catalog paths, bracket notation rules,
   no imports) — heavy prompting burden, and the two "optimize tools for GPT/Claude" plugins are shipped *disabled*
   (`plugin/optimize.ts:31-47`).
4. Compaction stringifies recent history (own TODO says it "confuses some models", `compaction.ts:74-75`); summary
   template enforcement relies on a nudge retry (`:178-179`).
5. Prompt-cache key is the session ID — no content-addressed validation that the provider actually honors keys;
   forks inherit affinity only by convention.
6. Foreground HTTP prompt + SSE split means clients must handle replay/backfill themselves (mitigated by the log
   stream, but the prompt call itself gives no streaming handle).

## What works (source-verified) vs unproven

- Verified as implemented and internally consistent: everything with file:line above, incl. prompt-ID idempotency,
  claim-based restart continuity, parallel tool fibers, auto-backgrounding, native-compaction plumbing (capability
  gated by `LLMClient.canCompact`, `compaction.ts:355-368`).
- Unproven externally: whether native compaction measurably beats summarization (no benchmark in repo); CodeMode
  token economics on weak models; session WS stability at scale (circuit breaker suggests past pain, but no data);
  cloud share/sync (`V2_HTTP_API_AUDIT.md`, STATS.md, README = **documented-only**, not audited here);
  `2.0`/`beta` branch contents (not fetched).

## What Zero Code should learn (transferable, mapped to baseline gaps)

1. **Admit-and-return HTTP semantics + doorbell wake** — replaces our blocking V2 prompt handler
   (`handlers/session.ts:335-355` in baseline). The `run-coordinator.ts` doorbell (scope lattice `input⊇steer`,
   wake-claims vs cleanup-wakes rule) is ~190 lines and directly transplantable into our `SessionExecution`.
2. **Write-ahead execution claim kept on shutdown interruption** (`execution.ts:76-86,129-134`) — closes baseline
   gap #3 ("crash mid-drain repaired only lazily"). Pair with restart settlement of stale running tools/compactions
   (`runner/llm.ts:332-385`) — we already have the tool-state fields to do this.
3. **Outcome-enum continuation policy** (`step.ts:32-42` + `llm.ts:273-328`): the synthetic-message variants for
   interrupted-stream ("continue without repeating") and output-limit (hard cap at 3 consecutive `length` finishes,
   anti-apology text) are exactly V1's doom-loop/output-length machinery expressed in V2 terms — a concrete
   migration target for baseline gap "V2 lacks retry/doom-loop handling".
4. **Overflow ladder before compaction**: 70/50/35% re-aim + payload-too-large→text-only resend
   (`compaction.ts:95,428-442`) — cheap provider-agnostic insurance we lack.
5. **Declarative cache policy + family promptCacheKey + instruction ordering by sharedness + "keep tool defs on
   the capped step"** — complements our frozen-prefix work with an explicit breakpoint budget and fork/child cache
   lineage (`affinity.ts`).
6. **Update-style compaction checkpoint** with the Objective/Work-State/Next-Move template and the legacy-template
   one-shot rewrite — directly improves our cache-continuation summaries.
7. **Timeout→auto-background with push notification + explicit anti-poll contract** (`shell.ts:254-266`,
   `BACKGROUND_INSTRUCTION` at `:26-27`) — we have background jobs but not the automatic conversion.
8. **Provenance-fenced native provider compaction** — design to adopt when we wire Anthropic/OpenAI native
   context-editing; the endpoint-hash guard (`provider-context.ts:18-45`) is the piece everyone forgets.
9. **Declines-as-defects** so no tool can convert a permission "no" into model text (`model-request.ts:400-418`).
10. **Model-persona prompts as tiny override/append plugins** matched on model id (`optimize.ts`) — cheaper than
    compiling provider quirks into the core; note their grep/glob-for-CLI-models experiment as a hypothesis to test,
    not adopt blindly.
