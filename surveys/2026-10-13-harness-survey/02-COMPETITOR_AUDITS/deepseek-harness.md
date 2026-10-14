# 02 — DeepSeek Harness (dsh): code-execution tool composition audit

Audited 2026-10-13 at head `d743267388641bc76f17c45ce8b4c231aed1d32c` (2026-10-09, release `0.2.1-alpha.2`, MIT), cloned at
`~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/deepseek-harness`. Method: direct source reading of the PTC
runtime, tool registry, agent loop, sandbox, persistence, LLM adapters, presets, and design notes. Classifications used
throughout: **[SV]** source-verified (file cited), **[D]** documented-only (README/agent-note claim), **[C]** marketing claim.
All paths below are relative to the repo root.

Context: cited in the assignment for CODE-EXECUTION-BASED TOOL COMPOSITION. Their feature is called **PTC (Programmatic Tool
Calling)** — a `run_code` tool plus a generated typed SDK, in explicit opposition to native function calling. Their design note
records Codex Code Mode as an explicitly considered (and partially rejected) model, so this is a direct comparator to Zero's
codemode package.

## 1. Architecture

**Layout** [SV]: pnpm monorepo, ~44 domain packages under `packages/*/*`, `apps/{cli,desktop,desktop-host,web}`, `native/system`
(N-API addons incl. a Linux landlock launcher), `python/{sdk,sdk-runtime}` (client SDKs), `vendor/cordis` (DI/plugin framework,
"cordis" = their composition substrate), `website`, `benchmarks`, `docs`, and `.agents/notes/` (an unusually rich internal
design-decision record). TypeScript throughout, Node 22/24, Electron desktop, MIT.

Everything is composed as **Cordis plugin rows in `cordis.yml`** — services (`llm`, `tools`, `session`, `sandbox`, `ptcRuntime`)
are injected into plugin classes, scoped layers shadow globals, and shipped products are patch-files over a base composition
(`packages/bundle/base/cordis.patch.yml`, web-app presets at `packages/bundle/web-app/presets/*.patch.yml`). [SV]

Core layout relevant to this audit:

| Area | Package | Key file |
|---|---|---|
| Tool registry + `run_code` | `packages/core/tools` | `src/index.ts` (1991 ln), `src/ptc.ts` (775 ln) |
| SDK codegen TS/Py | same | `src/ts-types.ts`, `src/py-types.ts` |
| Agent loop | `packages/core/agent-loop` | `src/agent.ts` (705 ln), `src/tool-calls.ts` |
| PTC runtime seam + Node backend | `packages/ptc-runtime/*` | `ptc-runtime/src/types.ts`, `ptc-runtime-node/src/index.ts` |
| Python backend (experimental) | `packages/experimental/ptc-runtime-python` | `src/index.ts` (2382 ln) |
| Sandbox | `packages/sandbox/*` | `sandbox-local/src/index.ts`, `sandbox-policy/src/*` |
| Session durability | `packages/session/*` | `session-persistence-jsonl/src/*` |
| Providers | `packages/llm/*` | `llm-deepseek/src/*`, `llm-pi-ai/src/*`, `token-meter/src/estimate.ts` |

## 2. Code-execution tool composition (the core mechanics)

### 2.1 Mode switch — presentation, not plumbing [SV]

The registry has one `mode: 'native' | 'ptc'` presentation setting (`core/tools/src/index.ts:674-683`; default `native`,
`index.ts:811-813`). Mode resolution is **scope-chained** (`modeFor`, `index.ts:925-938`): an agent preset declares presentation
once via the `@deepseek-ai/dsh-agent-tool-presentation` row (`core/agent-tool-presentation/src/index.ts:44-56`), so a PTC-mode
preset and native presets run side-by-side in one process, per-agent, not per-tool and not a global flag. Shipped web bundle
ships PTC as a first-class UI preset named "PTC mode" alongside the native preset (`bundle/web-app/presets/ptc.patch.yml`,
`client/ui-agent-preset/src/client/locales.ts:53`). Migration history shows the preset was renamed `code` → `ptc`
(`session/session-format-v2-to-v3/src/migration.ts:18`).

Under `ptc`, `wireSchemas()` collapses the wire surface to exactly one schema — `run_code` — and the executor enforces the same
collapse: "a model-direct call naming any other visible tool resolves to `UNKNOWN_TOOL` before policy — the announced surface
and the callable surface stay the same" (`index.ts:1017-1039` filter to `RUN_CODE_NAME`; enforcement comment `index.ts:343-348`;
README statement `core/tools/README.md:130`). A prompt section `tools:ptc-only` tells the model this *before* the SDK block:
"`run_code` is the only tool you can call directly — a tool call naming any other tool fails" (`index.ts:60`).

### 2.2 The generated SDK (the tool-as-function surface) [SV]

Same registry store, second projection: `schemas()` → native JSON schemas; `ts-types.ts`/`py-types.ts` → prompt text.

- **TypeScript flavor**: emits `interface ToolArgsMap`/`ToolOutputMap` + mapped `declare const tools` (`ts-types.ts:297-316`) —
  note the *output* schema of every tool is projected too (`ToolSdkSchema.output`, `ts-types.ts:13-16`), so `await tools.read(...)`
  resolves to a typed canonical JSON value the model can see before running. Deterministic: lexicographic tool order, byte-identical
  text for an unchanged tool set (`ts-types.ts:289-293`) — explicitly to keep the KV-cache prefix stable.
- **Python flavor**: one named `TypedDict` per object-shaped arg/output (not `dict[str, Any]`), because "under `mode: 'ptc'` the
  native tool schemas are omitted from the request, so this generated SDK is the model's ONLY source" for shapes (`py-types.ts:1-13`).
  Identifier hygiene is obsessive: bare-identifier predicate `isBareIdentifier` (XID_Start/Continue + NFKC-stability, with a
  documented Node-vs-CPython Unicode-table-skew analysis, `py-types.ts:46-110`), Python hard keywords routed to subscript access,
  soft keywords deliberately *not* excluded (`py-types.ts:112-135`).
- The `run_code` schema itself is **language-aware at emission time**: `description`/`parameters` are lazy getters that resolve the
  loaded runtime's flavor (`RUN_CODE_FLAVORS`, `ptc.ts:92-95`, getters at `ptc.ts:753-773`), so a Python runtime never ships a
  TS-shaped schema beside a Python SDK, and an unknown language fails loud (`ptc.ts:149-166`).
- Instructions teach the contract: `await tools.name(args)`, quoted access for exotic names, `try/catch` on `ToolCallError` (a real
  injected error class whose `toolName` names the failed tool), `Promise.all` for independent reads, "Only what you print or return
  is program output", images auto-attached after the run (`ts-types.ts:250-261`, mirrored in `ptc.ts:53-78`).

### 2.3 `run_code` tool + sub-dispatch bridge [SV]

`createRunCodeTool` (`ptc.ts:336`) — required `description` (5–10 word UI label) + `code` (async-function body), optional model-facing
`timeoutMs`, and `sandbox_permissions`+`justification` escalation pair (`ptc.ts:109-128`). Mechanics:

1. **Bindings** = one `PtcBindingFunction` per tool in the *calling agent's visible set* (`registry.schemas(exec.agent)`,
   `ptc.ts:694-697`), null-prototype + `defineProperty` so a tool literally named `__proto__` is an own key (`ptc.ts:684-697`).
2. Each call normalizes args to **lossless JSON twice** — one snapshot dispatched, a sibling parse logged, so a mutating tool cannot
   desync the durable record (`jsonNormalizeArgs`, `ptc.ts:187-203`; comment at `ptc.ts:595-598`).
3. Sub-calls do **not** bypass the pipeline: they re-enter the registry scheduler (`scheduler.prepare/dispatch/finalize/finish`)
   under a per-run driver lane that replicates the native concurrency contract exactly — ordered stages (start-event append,
   pre-execute/guards, post-execute, context deferral, settle append) run serially in one lane; only the tool *body* overlaps, up to
   `maxParallelSubCalls` (default 10, `index.ts:797-801`); mutating calls are exclusive barriers held through commit; classification
   is re-read immediately before start, fail-closed exclusive (`ptc.ts:420-535`, `drive()` at `ptc.ts:471`).
4. **Durability of inner calls**: a `tool/ptc-dispatch-start` / `tool/ptc-dispatch` event pair per started sub-call is appended to the
   session log with the full rendered result content (`ptc.ts:617`, `ptc.ts:590-603`); a `tools/ptc-dispatch-log` waterfall lets
   backends replace the durable copy with a spill preview/locator without touching the program value (`index.ts:175-191`, spill-local
   mounted in base bundle). These events are **log-only** — only the curated outer `run_code` result (printed lines + return value,
   rendered at `ptc.ts:374-380`) enters the model transcript.
5. **Images** in a successful sub-result are deferred into the transcript as a user message with `source.kind: 'ptc-mode'` after the
   run (`ptc.ts:644-648`); `additionalContexts` forward; `concludesTurn` forwards from a nested *success* only, so a recovering
   program can't be silenced by a policy-denied inner call (`ptc.ts:650-658`).
6. **Run teardown**: one run-scoped AbortController follows the outer signal and fires on any settle, aborting in-flight sub-dispatches
   and draining the commit lane before the outer tool result closes; queued-unstarted calls are abandoned and their promises reject
   into the program (`ptc.ts:411-418`, `714-720`).
7. Failure is a **field, not an exception**: the runtime returns `PtcRunResult {value?, logs, error?}` with an orthogonal failure
   taxonomy (`exception|timeout|abort|worker-exit|invalid-output|output-limit|protocol|sandbox-unavailable`,
   `ptc-runtime/src/types.ts:112-140`); the bridge converts to `CodeRunFailedError` whose text carries kind + captured logs +
   sandbox status "suitable for feeding back to a model to self-correct" (`ptc.ts:168-180`, `722-727`).

### 2.4 Execution substrate and sandbox [SV]

**Node backend** (default, mounted in the base profile `bundle/base/cordis.patch.yml:391-393`): each program is a **fresh Node child
process** — no worker, no persistent kernel. Program is wrapped as an async function body and type-stripped with
`node:module stripTypeScriptTypes` ("erasable syntax only", `ptc-runtime-node/src/index.ts:48-50,209`); a JSON control protocol runs
over an inherited binary pipe separate from stdout/stderr (`protocol.ts`, `channel.ts`): frames `ready/boot/call/reply/log/done`
with host-side validation of every frame because "model code can write directly to the inherited descriptor" (`protocol.ts:1-4`).
Caps (`index.ts:57-69`): wall deadline default **120 s, cap 600 s** (approval and nested-tool waits count); V8 old-gen heap **512 MiB**
via `--max-old-generation-size`; combined outer output **64 MiB**; control frame **128 MiB**; **128** simultaneous pending binding
calls (`index.ts:324`). `process.env` starts empty — the entire host environment is scrubbed for the child (`index.ts:233-239`).
Calls that arrive out of order, repeat ids, or name an unbound global are protocol failures, killed not guessed (`index.ts:316-324`).

**Python backend** (experimental package): fresh `python3` (3.10+, resolved+validated once at load under a 5 s probe) per program,
asyncio bootstrap giving top-level `await`, binding calls on **fd 3** JSON-lines (`experimental/ptc-runtime-python/src/index.ts:1-8`);
`RLIMIT_CPU` (SIGXCPU soft/+1 s SIGKILL hard) and `RLIMIT_AS` (skipped on Darwin, documented why), separate wall ceiling, per-byte
log/value budgets load-time-bounded against address space *and* the host's own JSON.parse headroom (`index.ts:38-101,744-1037`).
Per-call `timeoutMs` is rejected, not clamped (`index.ts:981`).

**File effects** go through the same OS sandbox as bash: `ctx.sandbox.confine(argv, policy)` (`ptc-runtime-node/src/index.ts:228`).
Backends: Linux bwrap → landlock launcher → macOS Seatbelt (`sandbox-exec`), Windows named-user ACL runner, each behind a *functional
probe* that actually runs `true` under the profile before trusting it; enforcement is reported `full|partial` and surfaces in the
model-visible output (`sandbox/sandbox-local/src/index.ts:68-142`; `ptc.ts:377-378`). Modes `read-only|workspace-write|danger-full-access`;
the service fail-safe default is `read-only` (`sandbox-policy/src/index.ts:113`) while shipped deployments set `DSH_PERMISSION_MODE ??
'workspace-write'` (`bundle/base/cordis.patch.yml:241`). **Escalation**: the model may request a wider mode *for this one program run*
with a `justification` shown to the approver; nested tools retain their own policies/approvals; guidance warns "Earlier effects may
already have completed… Programs are never replayed automatically" (`ptc.ts:130-133,396-408`). The design note is honest that this is
not multi-tenant isolation and that per-run process launch + sandbox setup is the latency price paid (`convsessions` section of
`.agents/notes/implemented/architecture/2026-09-11-sandboxed-node-ptc-runtime.md`). [SV/D]

**Data boundary**: bidirectional lossless JSON, no `undefined`/BigInt/bytes smuggled (host rejects non-lossless resolutions,
`ptc-runtime-node/src/index.ts:329-333`); intermediate binding values are **execution-local and unbounded by bytes** — cannot be
reconstructed from replay and may exhaust memory; only outer output is hard-capped; **no persistent REPL kernel** by design, because
cross-call state would be invisible to the log (`core/tools/README.md` Known Limitations, explicit bullets). [SV+D]

### 2.5 Versus Zero's codemode [SV both sides]

| | dsh PTC | Zero codemode |
|---|---|---|
| Surface | all registry tools (bash/fs/search/MCP…) as typed fns | MCP tools only |
| Interpreter | real out-of-process Node/CPython, direct host APIs | in-process custom interpreter (`packages/codemode/src/interpreter`) |
| Enforcement | OS sandbox (bwrap/landlock/seatbelt), escalation+approval | interpreter-level; "uncapped internally when on" (01 baseline) |
| Caps | wall/heap/output/frame/pending all configured | none noted |
| Transcript | only curated outer output; per-subcall durable events | flag-off path |
| Model visibility | tool *output* types generated into prompt | — |
| State | stateless per run, kernel rejected on log-reconstructibility grounds | — |

dsh traded launch latency for strong isolation + full-fidelity host APIs; Zero's in-process design trades capability for
cheapness — but without dsh's caps or durable sub-call record, the on-state is the riskier posture of the two.

## 3. Loop & durability [SV]

`ReactLoopAgent` (`core/agent-loop/src/agent.ts:98`) is a phase machine (`idle|maintenance|running`) over an **event-sourced
Session**: every request is *derived from the session log* (`agent.ts:4`), stored as append-only JSONL, **zstd-compressed by
default** with leases, torn-frame recovery, and versioned migrations v0→v4 (`session/session-persistence-jsonl/src/format.ts:35`,
`index.ts:68`, `catalog-migration.ts`). Events seen: `turn/start|end`, `step/start|end`, `user/assistant/system/developer/message`,
`tool/call`, `tool/result` (citing its `tool/call` seq via `sourceEventSeqs`), `tool/ptc-dispatch(-start)`, `request/header`
(canonical config+tools hash, `agent.ts:626-651`), `sandbox/mode`, compaction shadows.

- **Turns/steps**: `turn()` loops `step()` until no new inbox input at a stop boundary (`agent.ts:298-398`); `step()` streams one
  `llm.stream` per attempt (`agent.ts:453`). **Completion detection** = assistant message with zero tool-call blocks
  (`agent.ts:549-550`), or a tool result carrying `concludesTurn`; `max-tokens` is sticky within a turn (`agent.ts:338-343`).
  **No step-count cap** — long-horizon restraint comes from compaction pressure and goals, not iteration limits (grep: no
  maxSteps anywhere).
- **Delivery vocabulary**: `steer()` inserts at the next step boundary, `followup()` at next turn, `inject()` without wake
  (`agent.ts:163-172`) — same steer/queue semantics Zero is building in V2, but shipped.
- **Error handling**: stream failures settle `assistant/attempt` durably, then a plugin waterfall may return `retry`
  (`agent.ts:506-526`); a step that throws mid-tool-execution runs `ToolCallRecovery` to append conservative results for every
  pending call before rethrowing, keeping the log replay-valid (`agent.ts:333-358`); aborted/skipped calls get synthetic
  `TOOL_ABORTED_BEFORE_DISPATCH` results (`tool-calls.ts:249-260`). Tool failures are structured `isError` results, never turn-enders.
- **Context management**: token meter is a fixed 4-chars/token heuristic + provider usage (`llm/token-meter/src/estimate.ts:12` —
  no real tokenizer anywhere in the repo); compaction triggers on threshold pressure *and* context-overflow recovery
  (`compaction/compaction-basic/src/index.ts:164,205`), plus a tool-result pruner (head 4096/tail 1024 of >8192-char results) and
  image offload. PTC composes with this: only outer results enter the pruned/compacted surface.
- **Durable execution boundary**: crash between tool call and result is repaired on replay by appending a `TOOL_OUTCOME_UNKNOWN`
  marker (user-questions doc describes the resume repair); no automatic program replay after denial/timeout — explicitly:
  "A denial or transport failure never automatically replays a program whose effects may already have occurred" [D, agent note; SV
  for the recovery appends]. There is no separate durable-execution/status layer beyond the event log itself.

## 4. Composability & extensibility [SV]

- **Registration**: `ctx.tools.register(definition)`; scoped layers shadow globals; per-agent `ToolRestriction` allow/deny never
  touches the reserved `run_code` transport (`index.ts:944-947` "never enters the global layer"). Fixed 5-stage pipeline:
  `tools/pre-execute` waterfall (allow/deny/ask) → monotonic guards → `tools/execute` around-wrappers (timeout/retry/metrics) →
  `tools/post-execute` (replace content, attach contexts) → observe-only `tools/result` (`core/tools/README.md:109-141`).
  Every sub-call through PTC traverses the same pipeline (see §2.3.3).
- **MCP**: opt-in plugin row per server; tools registered as `mcp__<server>__<raw>` normalized to the **DeepSeek function-name
  contract** (≤64 chars, `[A-Za-z0-9_-]`, SHA-256 suffix on collision), raw name only ever sent on the wire, generation-swap
  re-sync (`mcp/mcp-client/src/tools.ts:1-10,45-107`). MCP tools therefore flow into the PTC SDK like anything else — hyphenated
  names land on the quoted/subscript path. MCP is a *tool source*, not the composition story.
- **Orchestration**: subagents as tools with pluggable backends — in-process `spawn` (fresh child agent, zero parent context) and
  `fork` (inherits history, KV-cache-eligible, model pinned), plus **external-agent bridges** to Claude Code, Codex, and any ACP
  agent (`packages/subagent/{subagent-spawn-in-process,subagent-fork-in-process,subagent-claude-code,subagent-codex,subagent-acp}`).
  `workflow` engine + `workflow-ptc` (orchestration guest programs run in the same sandboxed PTC executor; `vm` realms + process
  policy, `workflow/workflow-ptc/src/index.ts:1-6`) — both **disabled by default in the shipped preset**
  (`ptc.patch.yml`). Experimental `agent-team` (mailboxes/roster), `goal`+`goal-round-driver` (event-sourced goals re-waking the
  same session in rounds), `schedule`, `jobs` (background shells). Multi-agent exists but is visibly less hardened than the
  single-agent loop.
- **Config surface**: everything is a typed (`z`/schemastery) plugin config in cordis yml; the PTC python runtime brags "every
  cap, changeable from `cordis.yml` (no hardcoded tunables)" (`python index.ts:39`). Docs mirror config in
  `docs/config-catalog.md`. A `preset` mechanism (declarative plugin bundles with per-scope isolation flags) is how modes,
  compaction, delegation variants are packaged for users.

## 5. DeepSeek-model-specific work [SV]

- **Two providers shipped**: `llm-deepseek` (official "DeepSeek Messages" API — anthropic-flavored wire with `thinking` blocks and
  **reasoning signatures**) and generic `llm-pi-ai` (any pi-ai provider: openai/anthropic/etc., `llm-pi-ai/src/index.ts:1-20`).
- **Catalog defaults**: `deepseek-flash` (V4.1-Flash) `contextWindow: 1_000_000`, `DEFAULT_MAX_TOKENS = 256_000`
  (`llm-deepseek/src/defaults.ts:5-7`, `models.ts:6-17`). Model-specific prompt-cache tricks: `systemPromptUpdate: 'in-history'`
  and `toolUpdate: 'addition-only'` — tool set changes are appended as `developer/message` additions/removals instead of rewriting
  the prefix (`models.ts:13-15`, `agent.ts:652-665`).
- **Reasoning handling**: replay envelopes persist per-message `replayState` with per-block reasoning signatures; signatures are
  re-sent on continuation and refused across models; user/tool-result serialization strips reasoning and tool-call blocks
  (`llm-deepseek/src/replay.ts:29-61`, `serialize.ts:31-66`); reasoning-effort mapped with explicit UNSUPPORTED errors
  (`serialize.ts:146-148`).
- **Wire extensions**: `x-deepseek-harness-*` headers and `dsh_*` request-body fields with an accept-transaction protocol, incl.
  incremental **session-log upload** (`dsh_session_log`, watermark-based resend in canonical log) and image offload via a Files API
  with 7-day expiry (`docs/deepseek-llm-api-wire-extensions.md`, `session/session-log-deepseek/src/index.ts:1-8`,
  `llm-deepseek/src/defaults.ts:11-25`). This is server-side context/KV economics work, not model-format work.
- **The python-flavored-call question**: the repo contains **no tool-call argument repair/reparse layer** for DeepSeek quirks
  (grep for repair/single-quote/repr: only generic `parseArguments` falling back to raw text, `tool-calls.ts:104-111`; malformed
  provider streams *throw* `MALFORMED_RESPONSE`). The answer to the format problem is structural, not syntactic: in PTC mode the
  model emits *executed code* whose "schema violations" become ordinary exceptions inside the program with `try/catch`, and typed
  SDK declarations replace schema-following. That's a real alternative to per-provider call-repair — but only on the PTC path;
  native-mode DeepSeek calls get no special handling. [SV for absence; the strategic framing is D]
- **Tokenizer budgeting**: none. `CHARS_PER_TOKEN = 4` heuristic + provider-reported usage; explicitly "used until exact
  tokenization is needed" (`estimate.ts:12`). At 1M claimed context, compaction thresholds are heuristic — a weakness.

## 6. Benchmark claims & credibility

- **No capability benchmarks at all.** `BENCHMARK.md` is a 3-line pointer to running the SDK; `benchmarks/*` are **engineering
  perf budgets** (e.g. agent-continuation: 100 turns/800 real file reads through the shipped minimal profile, reviewed median CI
  budgets 900 ms ×1.25 headroom with per-machine scaling; session-corpus: 1k/3k synthetic sessions sized to measured local-corpus
  quantiles). Independently runnable locally (`pnpm build:bench` + vitest bench configs), reproducible, contamination-free — but
  they measure harness throughput, **not agent effectiveness** [SV].
- **No PTC-vs-native evidence.** Zero A/B data, no token deltas, no task scores for PTC mode. The design notes are unusually
  honest about it: "the Agent Note makes **no unconditional-savings claim**… SDK can rival the native schemas it replaces";
  "Measured guidance (when to prefer which mode) is explicitly **post-ship learning**"
  (`.agents/notes/implemented/feature/2026-06-15-ptc.md:128`). The stated mechanism-level benefit — one round-trip per *program*
  instead of per call, intermediate values never re-entering context, loops/branches/fan-out expressible in code — is coherent and
  source-consistent [SV mechanics], but the quality/cost outcome remains **[C]/unmeasured**.
- README makes no leaderboard claims (grep for score/SWE-bench: nothing) [SV].
- **Internal evidence quality is high**: ~59 test files in the four core PTC/loop packages alone, hostile-input validation
  everywhere, functional sandbox probes, real-Electron macOS regression for sandbox confinement of PTC writes, benchmark lanes with
  reviewed budgets enforced in required CI, and a design-note corpus that records rejected alternatives (including Codex Code Mode)
  with reasons. The repo validates *behavior*, it just hasn't measured *utility*.

## 7. Strengths / weaknesses

**Strengths**
1. PTC is a **first-class, per-agent presentation mode** with an enforced surface collapse (wire, executor, and prompt all agree),
   not a bolt-on flag; native and PTC agents coexist in one process (§2.1).
2. Sub-calls reuse the *entire* native tool pipeline — policy, guards, timeouts, concurrency contract, durable event pairing — so
   code composition adds no policy escape hatch (§2.3), while intermediate values stay out of the transcript (§2.3.4).
3. Honest boundaries: typed tool *outputs* in the SDK; lossless-JSON both directions; typed failure taxonomy designed for model
   self-correction; per-run OS-sandboxed real processes with explicit caps; escalation approvals that never auto-replay (§2.4).
4. The event-sourced log makes turns/steps/sub-calls/retries/resumes replayable and repairable; steering vocabulary shipped
   (§3). DeepSeek-specific cache economics (in-history system updates, addition-only tool updates, session-log upload) are clever
   (§5).
5. Documentation-to-reality ratio is exceptional for the genre — limitations are stated where the code lives.

**Weaknesses**
1. **No effectiveness data.** The core premise (code composition beats tool-calling for tool-heavy work) is asserted, not
   measured, by their own admission (§6).
2. Fresh-process-per-run: launch + sandbox cost per `run_code` call (their own "Consequences" section) and no persistent kernel —
   iterative exploration re-pays cold start; Python backend lacks per-call timeout and skips RLIMIT_AS on Darwin.
3. Heuristic 4-chars/token budgeting against a claimed 1M context; compaction/pressure decisions inherit that slop (§5).
4. Intermediate PTC values unbounded and unreconstructable from replay — a durability hole acknowledged in their own limitation
   list; approval waits consume the same wall budget as compute (open questions left in the note).
5. Multi-agent/workflow/teams are experimental or shipped-disabled; PTC SDK language is locked to the single loaded runtime and mode
   is per-agent, not per-tool; no mixed native+PTC within one agent.
6. No tool-call repair layer on the native path (relies on provider discipline) while Zero's loop assumes weaker-model tolerance
   machinery.

## 8. What Zero Code should learn

1. **Ship codemode as a presentation mode, not a flag.** The decisive dsh pattern: per-scope `mode: native|ptc` with *executor-enforced*
   collapse and a "you may only call run_code" prompt rule read before the SDK. Zero's flag-off MCP-only codemode with uncapped
   internals is the weakest of the three possible postures (not default, not broad, not bounded). The upgrade path is well-marked:
   keep native default, add a `ptc`-style preset row (Zero already has presets/agents), and let evals move the default.
2. **Make codemode real by covering *all* tools, not MCP.** dsh's composability argument only lands when bash/fs/search/jobs are
   callable in the program — that's where loops/fan-out/post-processing pay off. Zero's registry already has the tool store; the
   work is the SDK projector (their ts-types/py-types is directly imitable, ~300 lines for the TS flavor) plus a binding bridge
   that re-enters Zero's existing tool pipeline exactly as `ptc.ts:420-535` re-enters theirs.
3. **Log nested dispatch as durable event pairs, keep only curated output in the transcript.** `tool/ptc-dispatch-start`/
   `tool/ptc-dispatch` with spill-shaping gives auditability and inspector replay without context bloat; Zero's V2 `session_input`
   /event machinery can carry the same pair. Also copy: images/attachments deferred as post-run attributed context, and
   `concludesTurn` forwarding only on nested success.
4. **Project tool output schemas into the SDK, not just inputs, and generate deterministically** (lex order, byte-stable) for
   prefix-cache stability. This is the single highest-leverage prompt-side idea here.
5. **Copy the failure taxonomy and the caps.** One `PtcRunResult` with orthogonal kinds fed back verbatim for self-correction;
   default 120 s/cap 600 s wall, output/frame/pending-call budgets as config — even an in-process interpreter needs ledgered
   output and a pending-call cap before codemode ships on. The explicit "never auto-replay a program whose effects may have
   occurred" rule should be adopted verbatim.
6. **Don't claim token savings in the README.** Their formulation ("trades end-tool schemas for SDK text + one transport schema;
   savings are round-trips and intermediates, not prompt size") is the honest framing Zero should use too — and Zero's `zero-eval`
   can produce the native-vs-code A/B evidence neither dsh nor Codex has published, which is the cheapest way to decide whether to
   "make codemode real" or retire it.
7. **Statelessness is a defensible position**: their rejection of a persistent kernel because cross-call state would be invisible
   to the log aligns exactly with Zero's durable-log principle; if Zero's codemode ever keeps a kernel, it needs an event model for
   it first.
8. Steal the CI perf-budget pattern (reviewed medians + machine profiles + required lane) for Zero's benchmarks, and steal the
   agent-notes pattern (decision + alternatives + consequences, incl. rejected alternatives with links) for the harness-research
   follow-through.

## Verdict

PTC is the most *architecturally complete* code-execution tool-composition implementation seen in this survey: enforced surface
collapse, full-pipeline sub-dispatch with durable logging, OS-level confinement with per-run real interpreters in two languages,
typed generated SDKs, all wired into an event-sourced session core. Its evidence for *effectiveness* is nil by its own admission;
its evidence for *engineering soundness* is exceptional. For Zero Code: the design is adoptable incrementally and the open
question — is code composition actually better — is one Zero's own eval harness can answer before committing.
