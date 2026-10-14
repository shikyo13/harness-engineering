# 02 — Competitor audit: Qwen Code

Audited 2026-10-13 at head `6212189f1f8334da78a69e3fb6b63ff6ca88251e`
(`feat(managed-agent): Add the public-event transport and the wake/materialization consumers (MQ2c)`, PR #14080).
Clone is shallow depth-1, branch `main` only — commit-history archaeology impossible; lineage established from
residual source identifiers instead. Version `0.25.1`, pnpm workspace, Node ≥22, 23 packages. Claims are
`file:line`-backed against implementation unless marked **[doc-only]**.

## 1. Fork lineage (verified in source)

- README admits it: "originally based on Google Gemini CLI v0.8.2 … stopped syncing at Qwen Code v0.1"
  (`README.md:214`). Source confirms it: **643 non-test TS/TSX files still carry `Copyright 2025 Google LLC`**
  vs 2,269 Qwen-authored headers — diverged hard, skeleton never rewritten.
- Gemini-CLI skeleton retained verbatim in name and shape:
  - `core/coreToolScheduler.ts` (`CoreToolScheduler` at :1748), `core/turn.ts` (`Turn` at :702).
  - `utils/nextSpeakerChecker.ts` — Google header plus the exact `next_speaker` side-query prompt (:18-35).
  - `services/loopDetectionService.ts`, `shellExecutionService.ts`, `fileDiscoveryService.ts`, `gitInit.ts`.
  - `@google/genai` `Content`/`Part` types remain the **internal wire format for every provider** —
    OpenAI/Anthropic/Qwen requests convert through genai shapes.
- Renames are recent and shimmed, not inherited: `GeminiClient`→`LlmClient` (`core/client.ts:458`),
  `GeminiChat`→`LlmChat` (`llm-chat.ts:2282`), with `core/geminiChat.ts` kept as an 8-line deprecated
  re-export. `geminiRequest.ts` and `geminiContentGenerator/` still exist. `.gemini`→`.qwen`
  (`utils/paths.ts:14`), `GEMINI.md`→`QWEN.md` (`utils/memory-constants.ts:7`); `GEMINI.md`/`CLAUDE.md`
  remain recognized context-file aliases (`permissions/classifier-prompts/system-prompt.ts:89`).
- Not Gemini CLI at all (Qwen-authored): `agents/`, `subagents/`, `managed-runtime/`, `goals/`, `memory/`,
  `skills/`, `code-mode/`, `omni/`, `hooks/`, `channels/*` (8 messaging adapters), `sdk-java|python|typescript`,
  `acp-bridge`, `web-shell`, `browser-use`, `cua-driver`, `audio-capture`, `mobile-*`. The fork grew ~10× its
  parent's surface: the CLI package alone is a daemon (`serve/` ≈ 250 files), not an Ink app.
- Inheritance-in-practice lesson (for the Antigravity-vs-Gemini study): the fork kept the upstream loop and
  scheduler bones untouched and reinvested *around* them (adapters, multi-agent, hosting). Predictable
  Gemini-CLI fingerprints to look for in any other fork: next-speaker LLM continuation, exact-match edit,
  genai Content/Part as lingua franca.

## 2. Agent loop

- One `LlmClient.sendMessageStream` async generator per model turn (`core/client.ts:3479`). Consumers:
  TUI hook `ui/hooks/use-llm-stream.ts` (6,815 lines) and headless `nonInteractiveCli.ts` (3,508 lines).
  Continuation is decided inside core, not the UI.
- **Continuation = inherited NextSpeaker side-query**: after a turn with no pending tool calls, a cheap-model
  check answers user-vs-model (`client.ts:5572-5600`); "model" recurses `sendMessageStream` with steer input
  or `"Please continue."`, bounded by `boundedTurns-1` (:5586). Hard depth cap `MAX_TURNS = 100`
  (`client.ts:222`); per-session `maxSessionTurns` gate (`client.ts:4386-4392`; config `:3507`); runtime
  Goal turns deliberately exempt from both (:4303, :4384).
- **Steering is first-class, three-variant**: `getSteerInput` polled before continue/hook/steer re-entry
  (`client.ts:5494-5600`); mid-turn messages carry distinct durable prefixes for "during tool execution" vs
  "interrupted my response" vs "while responding" (`cli/src/utils/midTurnUserMessage.ts:8-16`) — the
  distinction persists in history, not just display.
- **Parallel tool calls**: scheduler classifies each call concurrency-safe; consecutive safe calls merge into
  one parallel batch, unsafe calls run solo-sequential, order preserved (`coreToolScheduler.ts:1518-1680`,
  capped concurrent execution :5339-5394). Same classifier across runtimes (:1547-1556).
- **Loop safety**: inherited `loopDetectionService.ts`, extended to fingerprint results by the
  full-output sha256 embedded in truncation envelopes rather than the preview, so mutating outputs aren't
  misflagged and frozen ones are (`truncation.ts:28-37`).
- Headless loop drains durable inputs beyond prompts: teammate messages merged into pending parts
  (`nonInteractiveCli.ts:2521-2545`), cron goal ticks, monitor wakes, structured-output retries
  (:2838-3320). Dedicated interruption module (`turn-interruption.ts`); argument repair
  (`incomplete-tool-call-args.ts`); **XML tool-call text fallback** for `<invoke name=…>`/`<function=…>`
  emissions with a marked-lexer quadratic-blowup guard at 64KB (`xml-tool-call-fallback.ts:11-20`).
- No durable turn state in the *local* loop (crash replays from chat records) — durability ships only in the
  hosted managed path (§6), and there behind a default-off flag. Same V1/V2 topology as Zero Code.

## 3. Tools, context, compression

- ~60 built-ins (`tools/tool-names.ts`): read/edit/write/notebook-edit, shell + `exec` (code-mode),
  glob/grep/ripgrep/ls, web-fetch + web-search (DashScope variant `web-search-dashscope.ts`), todo,
  plan enter/exit, memory manage/search, skills, agent, team-create/delete, task-*, cron-*, loop-wakeup,
  send-message, monitor, workflow, `tool_search`+`tool_call` (deferred tools), advisor, image gen/zoom/display,
  worktree enter/exit, ask_user_question.
- **Edit = `old_string`/`new_string`** (`edit.ts:107-115`), Gemini-CLI-lineage Claude-style schema, not
  block-based; `replace_all` flag; prior-read enforcement before mutation
  (`priorReadEnforcement.ts`, wired `edit.ts:42`); line-ending/BOM preservation via `fileSystemService`
  observations; commit attribution. Matching is exact/count-based (`editHelper.ts`) — **no fuzzy cascade**;
  recovery relies on augmented old-string errors.
- **Shell**: one process per command, no persistent PTY session; PTY launch validates `PWD == cwd`
  (`shellExecutionService.ts:851-853`); cwd continuity via execution environments + worktrees.
  Background shells: `backgroundShellRegistry`, foreground→background promotion on timeout (`shell.ts:1318`),
  `is_background` param, and model-facing liveness caveats — "do NOT infer liveness from the output file …
  block-buffering" (`shell.ts:127-135`). Windows: `conpty-host` service.
- **Truncation**: single service; 2,000-char preview, full output spilled to file labeled
  `Full output sha256:`, idempotency sentinel prefix preventing double truncation, 50 MB/file, 500 MB/session
  budgets (`truncation.ts:19-44`). Very close in spirit to Zero Code's spill design.
- **Context assembly**: startup `<system-reminder>` snapshot; deferred-tool summaries capped at 160 chars;
  skills listing capped at 8,000 chars explicitly "to limit cached-prefix growth"
  (`environmentContext.ts:20-29`). QWEN.md hierarchy + rules files; safe-mode skip (`config.ts:5396`);
  always-on-context token estimate warned to user (`config.ts:5665`).
- **Compression**: `chatCompressionService.ts` — auto-compact inside `LlmChat.sendMessageStream`
  (`client.ts:4405-4408`); reactive forced re-compaction on context-overflow errors
  (`llm-chat.ts:4432-4545`); summary capped at 20k tokens, comment cites claude-code `autoCompact.ts:30`
  p99.99 basis (`chatCompressionService.ts:47-55`); pre/post-compact hooks; post-compact state reminders +
  restored file/image attachments (`postCompactAttachments.ts`); **microcompaction** — time-based in-place
  clearing of stale tool outputs/images with no summarizer (`services/microcompaction/microcompact.ts`,
  `QWEN_MC_KEEP_RECENT`); `sessionRecap.ts` away-summaries.
- **Prompt caching is provider-engineered, end to end**:
  - OpenAI path: `prompt_cache_key = "qwen-code:<sessionId>"` + two explicit `prompt_cache_breakpoint`s at
    recent user/tool boundaries, skipping the trailing compression directive (`prefix-caching.ts:73-117`).
  - DashScope path: stable-prefix `cacheControl: 'system_only'|'all'` marking; comment "cached prefix shifts
    every turn (issue #11627)" (`provider/dashscope.ts:787-843`).
  - Forks keep model-visible tool declarations byte-identical "for prompt-cache sharing" even when the fork
    is tool-restricted (`tools/agent/agent.ts:862`).
  - `captureCacheSafeParams` snapshots slimmed last-40-message history every turn so background forks reuse
    the parent cache prefix (`client.ts:1471-1489`, `agents/forkedAgent.ts:10-21`).
  - Cache stability treated as a cross-cutting invariant (compaction, fork, skills listing) — deepest
    prompt-cache engineering seen outside Anthropic's own harness. [source-verified]

## 4. Qwen-model-specific tuning

- **Qwen-first provider stack**: `QwenContentGenerator` = OpenAI generator + DashScope provider + shared
  OAuth tokens (`qwen/qwenContentGenerator.ts:24-45`). Presets pin per-model capability descriptors —
  `qwen3.8-max`: 1M context, efforts `[low,medium,xhigh]`, default `xhigh`, `disableField:
  'reasoning_effort'`; older hybrid models expose only on/off with `disableField: 'enable_thinking'`
  (`providers/presets/alibaba-standard.ts:100-140`, `alibaba-coding-plan.ts:75-78`).
- **Thinking-knob arbitration**: resolver picks exactly one of `enable_thinking` / `thinking_budget` /
  `reasoning_effort` per model because "qwen3.8 rejects reasoning_effort with thinking_budget"; precedence
  across config layers vs samplingParams vs user `extra_body`, null-knob sanitization
  (`provider/dashscope.ts:47-162, 398-437`). Vision models force `preserve_thinking: true` for **multi-turn
  reasoning continuity** + `vl_high_resolution_images` (:438-476).
- **Reasoning transport**: OpenAI-compatible converter round-trips DashScope's nonstandard
  `reasoning_content` into thought parts and back, guarding providers that 400 when it's echoed
  (`openaiContentGenerator/converter.ts:66-81, 549-551, 843-852`). Unified `/effort` abstraction over all
  providers (`core/reasoning-effort.ts`, `reasoning-overrides.ts`).
- **Tool-call quirks**: XML fallback parser (above); malformed-arguments repair (`tool-call-arguments.ts`,
  `incomplete-tool-call-args.ts`); canonical repeat keys for loop detection.
- **Tokenizer/limits**: no HF dependency; conservative char model, 1.1 tok/char for non-ASCII (CJK-lean)
  (`utils/request-tokenizer/textTokenizer.ts:10-28`); output clamp margin `max(10k, 5% of window)` so
  prompt+max_tokens never overflows (`tokenLimits.ts:59-99`).
- **Model-aware orchestration**: build-generated model catalog (`scripts/generate-model-catalog.ts`);
  `modalityDefaults.ts` gates image/video parts per model; PDF/image vision-bridge fallback for non-vision
  models (`services/visionBridge/` + design doc 2026-07-13); web-search routed to DashScope on Qwen OAuth.
- **Auto-memory is Qwen-authored**: extraction agents, `search-memory`/`manage-memory` tools, `/dream` idle
  consolidation (`memory/dream.ts`), `/remember`, `/forget`; recall prefetch pipelined into the turn
  (`client.ts:1460-1497`).
- Verdict: strongest published example of a harness tuned to one model family while staying multi-protocol
  (Gemini/Anthropic/OpenAI/Qwen generators behind one genai-shaped core).

## 5. Daemon, IDE, sessions, checkpointing

- **`qwen serve` is a real daemon**: 4,247-line `serve/server.ts`, ~332 route registrations incl. **A2A**
  (`routes/a2a.ts`), goals, scheduled tasks, channel webhooks, daemon update-restart, rate limits, SSE with
  `last-event-id` resume (`sse-last-event-id.ts`), trusted workspaces with hot reload, per-session git
  worktrees, memory-pressure admission (`daemon-memory-pressure.ts`). Web UI (`packages/web-shell`) served
  by the daemon; hosted harness mode is `--profile hosted-harness`.
- **IDE posture is ACP-first**: full Agent Client Protocol agent (`@agentclientprotocol/sdk`,
  `cli/src/acp-integration/`), reusable `packages/acp-bridge`, Zed extension, legacy VS Code companion.
  Channel bots (Telegram/WeChat/DingTalk/Feishu/QQ/GitHub/GitLab/Email) are themselves ACP clients of the
  daemon, supervised with stall-restart (`channels/base/src/index.ts` — `AcpBridge`, `PollingChannelBase`).
- **Session persistence**: JSONL chat records under `~/.qwen/tmp/<project_id>/chats/`
  (`services/sessionService.ts:858`); session registry, writer leases (`session-writer-lease.ts`), `/resume`
  picker with token counts, `/branch`, artifacts, PR service. Resume deserializes file-history snapshots so
  `/rewind` works across sessions (`sessionService.ts:431`).
- **Checkpointing = file-history snapshots; there is no `/checkpoint` command.**
  `fileHistoryService.ts` keeps versioned per-file backups with explicit `failed` flags so a stale backup is
  never silently restored as current (`:37-48`); `/restore` reverts files to a chosen tool-call point and
  replays conversation state (`restoreCommand.ts:72-166`); `/rewind` is a code+conversation dialog
  (`rewindCommand.ts:12`). The managed journal formalizes `checkpoint.committed`
  (`managed-runtime/managed-session-records.ts:63`). Content-addressed copies, not shadow git.
- Sandbox story: macOS Seatbelt, container, bwrap backend — each explained *to the model* in the system
  prompt so it attributes EPERM/EROFS correctly and tells the user how to fix it (`prompts.ts:631-648`).

## 6. Multi-agent + the managed-agent subsystem (head commit)

Local multi-agent (all source-verified):

- `agent` tool with **fork** subagent type: inherits parent conversation; cap
  `FORK_DEFAULT_MAX_TURNS = 200`; recursion guard via AsyncLocalStorage *chosen because* stripping the tool
  from declarations would break cache parity with the parent (`tools/agent/fork-subagent.ts:42-58`);
  `fork_turns`/`fork_tools`/`fork_profile` knobs. Top-level regular subagents default to **background** with
  completion notifications (`agent.ts:1003-1026`); background task view + resume
  (`agents/background-tasks.ts`, `background-agent-resume.ts`).
- **Teams**: `team-create`, named concurrent teammates, message passing, shared task lists, plan approval
  (`team-*.ts`); leader wait/stall policy in headless loop (`nonInteractiveCli.ts:2838-2905`).
- **Workflows**: JS scripts in a `node:vm` sandbox exposing `agent()/parallel()/pipeline()/phase()`,
  CPU-affinity-derived concurrency, agent caps, 30-min wall clock, per-run journal files, pause/resume
  (`tools/workflow/workflow.ts:1499`). **Arena**: parallel competing agents with diff summary
  (`agents/arena/`). **Advisor**: separate advisory model whose contract is a policy system-reminder —
  "trace each finding against actual code… advice is not user approval" (`core/advisor-policy.ts:10-25`).
- **Long-horizon autonomy quartet**: `goals/` (durable goal state, turn budgets, verifier, continuation
  prompts; runtime turns bypass `maxSessionTurns`), `cronScheduler` + `loop-wakeup` (agent schedules its own
  future prompts; permission 'ask' so the LLM classifier still vets them, `loop-wakeup.ts:36-45`), monitors
  (`monitorRegistry`, hosted monitor wakes), and background tasks.
- **Permissions**: AUTO mode is gated by an LLM security classifier with built-in hard/soft blocks and a
  denial-retry-ban ("retrying via shell indirection, generated script, symlink… completes the same denied
  action", `permissions/classifier-prompts/system-prompt.ts:34-73, 112`).

Managed-agent — the hosted durability story, half of it in this repo:

- Roles: WebShell → **Java Spring control plane** (`packages/sdk-java/managed-agent-server`, 820 Java files,
  Flyway migrations to V66; owns admission, state, event projection, client APIs, MySQL) → **Hosted
  Harness** (this TS repo running the agent loop) → Runtime Broker → tool-only Runtime. Ownership table:
  public events = short-lived SQL journal + MQ; model context/checkpoints = Harness journal, never
  reconstructed from public message tables
  **[doc `docs/design/2026-09-20-managed-agent-storage-event-architecture.md` §1-4; both code sides exist]**.
- TS durable foundation: closed journal vocabulary `input.accepted, wake.requested, activation.changed,
  model.attempt, message.committed, tool.intent, action.changed, tool.receipt, checkpoint.committed,
  context.compacted, cancel.requested, turn.settled, …` (`managed-session-records.ts:54-75`) with
  tenant-scoped ordering key + events digest; local JSONL journal store; writer leases;
  **durable prompt admission** — admit to inbox, mark activation ready, reconcile pending after restart
  (`managed-prompt-admission.ts:36-64`) — concept-for-concept Zero Code V2's admitted-input model;
  lease/fence/slot `embedded-harness-scheduler.ts` (lease renewal, memory headroom, per-tenant caps).
- **Wake semantics**: "inputs still owed a consumer" are *derived from journal replay* — an input is
  consumed iff some `turn.settled` carries its `turnId`; no separate ledger
  (`hosted-wake-intake.ts:12-52`). Recovery replays `managed-harness-checkpoint` wait phases
  (`hosted-runtime-recovery.ts`).
- **MQ2c (head commit) precisely**: extends `managed-event-envelope/1` with a second stream `public_event`
  — open lowercase-dotted kind grammar, 128-byte cap, digest-bound body with the payload never traveling,
  forbidden-field blocklist (pod, PID, absolute paths, secretHandle, runtimeEndpoint)
  (`managed-event-envelope.ts:41-90, 104-120`). Consumers in Java: `LiteTopicPublicEventTransport`
  (RocketMQ LiteTopic publish from the commit seam, bounded async queue), `TransportWakeActivator`
  (cross-node wakes on a shared bootstrap LiteTopic; fenced + deduped against SQL via live-ACTIVE-writer
  check, committed sequence, `journal_wake` watermark claim; loaded through new-work admission),
  `MessageMaterializer` (public events deduped on projection watermark, coalesced per session onto one
  thread); owner-lane subscriptions narrowed by SQL from connector attachments. TS↔Java mirror via shared
  JSON contract fixtures (`managed-runtime/contracts/*.fixtures.json`). Flag
  `qwen.managed-agent.event-transport.enabled`, **default off**.
- Classification: envelope contract, journal, admission, scheduler, wake derivation, both Java consumers —
  **source-verified, tested on both sides**. Multi-node production operation, MQ outage buffering, DR,
  tenant isolation — **[doc-only]**; the design doc itself disclaims production validation and admits SQL
  outage blocks event acceptance (§1 tradeoff).

## 7. Strengths / weaknesses

Strengths (vs Gemini CLI upstream, and relative to Zero Code):

1. Provider engineering discipline: per-model capability descriptors, knob arbitration, cache-breakpoint
   placement — the harness-tuned-to-model work we do for local models, done as explicit invariants.
2. Managed journal (closed event vocabulary + digest + wake-from-replay + exactly-once-by-turnId) is a
   *shipped* durable-session design while our V2 is flag-off and single-process.
3. Loop-adjacent product features with teeth: steer-variant durable prefixes, background-by-default
   subagents, `/dream` memory consolidation, advisor policy, code-mode `exec` script tool
   (`tools/exec.ts:12-28`), deferred tools via `tool_search`/`tool_call` with declaration fingerprints
   (`tool-registry.ts:62-75, 251-266`).
4. 1,154 dated bilingual design docs in `docs/design/` with explicit "proposed vs implemented" status lines.

Weaknesses:

1. Surface sprawl (CLI+daemon+desktop+mobile+voice+channels+arena+live+3 SDKs); reliability lives in
   megafiles — `client.ts` 6,027 lines, TUI hook 6,815 — harder to audit than our split.
2. Loop bones remain Gemini-era: next-speaker LLM side-query + "Please continue." continuation is wasteful
   and less deterministic than ours; local loop has no durable turn state.
3. Edit matching exact-only — weaker recovery than our fuzzy cascade; truncation envelope lacks
   capability-aware "do X next" hints.
4. Managed path tax: 4-process topology (server/harness/broker/runtime) + dual-language contract mirror;
   SQL stays on the event-acceptance critical path.
5. Shallow clone: upstream drift since the fork point unmeasurable; fork base (v0.8.2) documented +
   corroborated by residual headers, not history-verified.

Works vs unproven:

- **Works (source + tests)**: local loop, steering, teams/forks/workflows, DashScope tuning,
  compression/microcompaction, caching paths, file-history restore, serve daemon, ACP, channel bridge,
  managed journal/admission/wake/materialization consumers.
- **Documented-only**: managed multi-node production, MQ buffering/DR/compliance deletion, tenant-isolation
  guarantees, sandbox container image behavior, `desktop`/`live-host`/`mobile-shell` (workspace-excluded),
  quality outcomes of Qwen3.8 effort tiers (plumbing verified, efficacy not).

## 8. What Zero Code should learn

1. **Qwen-tuning lessons for local/weak models**: per-model capability descriptor with explicit
   `disableField` per generation; exactly-one-knob arbitration with documented provider rejects (their
   `reasoning_effort×thinking_budget` conflict maps directly to our provider-quirk handling);
   `preserve_thinking`-style continuity flags; reasoning-field round-trip guards; CJK-aware char
   tokenization for guardrails; output clamp margin as a % of window.
2. **Prompt-cache as cross-cutting invariants, not per-call hacks**: byte-identical tool declarations for
   restricted forks; caps on volatile content inside the stable prefix (skills listing); session-keyed cache
   keys with bounded explicit breakpoints; capture cache-safe params every turn + digest-check for
   background reuse. Adopt their forbidden-field list for any public event surface we emit.
3. **Managed-agent subsystem**: the pending-input set *derived from replay* (`input.accepted` minus
   `turn.settled.turnId`) is an elegant answer to our finding "no durable execution status anywhere" —
   no extra ledger, restart-safe by construction. Their lease/fence/watermark claim pattern is the template
   for taking V2 multi-process. Ours leads on delivery vocabulary (steer/queue) and context epochs; theirs
   leads on transport/recovery — converge there rather than inventing again.
4. **Gemini-CLI inheritance in practice** (feeds Antigravity-vs-Gemini study): what survives a fork is the
   turn/scheduler interfaces and the wire-type vocabulary (genai Content/Part as lingua franca made
   multi-provider nearly free — worth copying conceptually); what dies is branding, memory filename, and
   everything product-shaped, within months. Any Gemini-derived harness will carry the next-speaker
   continuation and exact-match edit fingerprints unless rewritten — check Antigravity for exactly these.
5. **Cheap wins worth porting**: steer-variant prefixes encoded in history; background-by-default subagents
   with the "don't infer liveness from the output file" contract text (`shell.ts:127-135`); `/dream` idle
   memory consolidation; advisor-consultation policy text; truncation idempotency sentinel + full-output
   digest fed to loop detection; dated design docs with proposed/implemented status lines.
