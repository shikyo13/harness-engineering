# Pi-derived execution plan for Zero Code

Date: 2026-09-17

## Decision

Yes: Pi should be the primary speed reference for Zero Code's coding
execution path. The current OpenBench matrix gives Pi the best median wall time
(`48.106 s`) with a strict pass rate and checker pass rate of `1.0`. Zero is
second (`71.543 s`) with the same correctness on those tasks. The taskflow
repeat also puts Pi first by median (`165.625 s` versus Zero `233.785 s`), but
that comparison has warm MLX prefix-cache effects and stochastic trajectories.
It is evidence for an execution design, not proof that every Pi run is faster.

The target is to adopt the parts of Pi that reduce avoidable harness work while
keeping Zero's desktop product, durable session semantics, permissions, browser
and computer tools, and richer UI. We should not route the desktop app to a
Pi-style runner until the existing Zero V2 runner has feature parity with the
desktop tool surface.

## What the comparison established

- Pi keeps an `Agent`/`AgentSession` and an in-memory context snapshot through a
  run, then calls one direct streaming loop per provider turn.
- Pi's default coding surface is four tools: `read`, `bash`, `edit`, and
  `write`. Zero's `coding-minimal` profile now exposes the same set as an
  opt-in profile; `full` remains the desktop default.
- Pi's Qwen `--thinking medium` setting maps to
  `enable_thinking=true`. Zero's Qwen configuration already sends that switch,
  so Pi's lead is not explained by a weaker reasoning mode.
- Zero has an opt-in converted-message cache and stable system/tool prefixes,
  but the legacy desktop path still reloads durable history and reconstructs
  execution state at each step.
- Zero's V2 runner already provides the right durable boundary and direct
  `llm.stream` seam, but its built-in tool set is incomplete for desktop use.

## Work plan

### 1. Finish measurement before changing defaults

Use the pinned OpenBench runner and one local Qwen process at a time. Record
per-provider-turn:

- time to first token, total wall time, prefill and decode rates;
- input, output, and reasoning tokens;
- tool-call count, tool-result bytes, system/tool schema bytes;
- context size and prefix-cache reuse;
- external checker result and task completion state.

The runner now records MLX prefix-cache reuse. Zero also emits request-prep,
history-load, and context-assembly timings so model latency can be separated
from harness latency. Compare repeated cold and warm trials and retain every
trial instead of reporting a single average.

Promotion gate: at least three cold and three warm trials for short coding,
multi-file coding, and taskflow, with strict checker pass rate `1.0` and no
regression in tool-call correctness.

### 2. Optimize the legacy path behind flags

Keep the current durable `SessionPrompt` behavior as the compatibility path.
Use measured, reversible changes in this order:

1. retain a per-run converted message snapshot and append only new tails;
2. retain immutable system and tool definitions for the stable session/model
   key while rebuilding permission-sensitive execution closures per turn;
3. avoid duplicate history and metadata serialization when the provider
   request is unchanged;
4. preserve compaction, interruption, permission, and retry invalidation rules.

Each change must pass an external checker on short, multi-file, and long tasks
before the next change is attempted. No speed claim is accepted from a build or
typecheck alone.

The first V2 implementation slice is now present: the runner keeps a
session/model/agent/context-epoch keyed converted-message prefix and appends
only entries that have become stable. The newest projected message is always
translated again, which preserves assistant/tool settlement updates. It is
covered by the V2 runner integration suite and remains isolated from the legacy
desktop route until the parity gate below is met.

The first rebuilt packaged taskflow control after this slice passed the
external checker in 369.732 s. Its trace had 16 main provider requests, a
54,681-token maximum prompt, 5,980 streamed output tokens, and 31 tool
messages, despite 86.19% mean provider prefix-cache reuse. The transcript
shows repeated repository rereads and investigation turns. This makes the
next Pi-derived experiment a trajectory/tool-loop reduction, not another
blind context-cache repeat; the cache remains valuable for reducing assembly
work but has not yet demonstrated a long-task wall-time win.

A matched legacy-path trial with `ZERO_EXPERIMENTAL_IN_MEMORY_CONTEXT=true`
also passed all 25 hidden tests but took 418.116 s. Its trace had 13 main
requests, 57,254 maximum prompt tokens, and 7,666 streamed output tokens.
Since this was slower than the 369.732 s no-cache control, the flag stays
opt-in. The next experiment should reduce unnecessary reasoning/tool turns and
repository rereads before changing context-cache defaults.

### 3. Complete the V2 execution seam

Treat `packages/core/src/session/runner` as the long-term Pi-like engine rather
than copying Pi's in-memory loop into the desktop handler. Complete the missing
Zero-specific built-ins and adapters, especially task/subagent, plan/code-mode,
LSP, browser, computer, and desktop permission events. Keep:

- durable prompt admission separate from model execution;
- process-global, Session-ID-based execution ownership;
- one `llm.stream(request)` call per provider turn;
- durable history reload at continuation boundaries;
- concurrent tool fibers and explicit steer/queue semantics.

Add an opt-in desktop feature flag that can select V2 for a disposable test
session. Keep legacy routing until V2 produces identical visible events,
permission behavior, tool results, compaction, cancellation, and recovery.

The current V2 runner and durable prompt seam pass the focused runtime suite
(`112` tests, `0` failures), including concurrent tool settlement, steering,
queued input, interruption, compaction, and provider error recovery. Zero's
location-service map already composes the canonical core `BuiltInTools` node,
including `bash`, `read`, `edit`, and `write`, so these four tools are available
through the intended V2 contract rather than a legacy adapter. That suite still
does not establish desktop readiness: task/subagent, browser/computer, and
desktop event and permission parity remain incomplete. The next implementation
slice should exercise the existing four-tool V2 surface with a real external
checker, then add the missing parity adapters before an opt-in desktop route is
considered.

### 4. Validate desktop parity and real tasks

Run the packaged desktop app, not only CLI adapters. Exercise the same coding
tasks through the actual composer, tool activity rows, file explorer, browser,
computer control, subagent flow, context meter, and cancellation UI. Then run
the Minecraft/Field Emitters pipeline as a long-running acceptance task:
launch the game, control it, capture evidence, iterate, and report the checker
result.

The desktop `full` profile stays the product baseline. A future coding mode may
select `coding-minimal` when the user does not need browser/computer tools, but
it must be explicit and reversible.

### 5. Cherry-pick only after interface and license review

Pi's loop and provider adapters are useful reference implementations. For each
candidate snippet, record the upstream commit, license, interface assumptions,
and the Zero adaptation in the research ledger. Prefer reimplementing the
behavior at Zero's durable boundaries; copy code only when the license and
dependency direction allow it. Do not import Pi's session state in a way that
duplicates Zero's database, permissions, or desktop event model.

## Promotion criteria

Promote a Pi-derived change only when all are true:

1. strict external checker pass rate remains `1.0` across the required task
   cells;
2. cold and warm medians improve or stay within the recorded noise band;
3. context growth, reasoning retention, and compaction remain correct;
4. desktop tool, permission, cancellation, and recovery behavior remain
   equivalent;
5. the change has a rollback flag and a saved runtime artifact.

Until then, Pi is the benchmark leader and architectural reference. Zero's
desktop default remains the correctness and capability baseline.


The compact system-prompt experiment also passed `make-it-run` but took
30.225 s versus the 21.076 s control. Keep it gated; reducing prompt bytes
does not guarantee a faster Qwen trajectory.

The switch now flows through `RuntimeFlags`; a rebuilt packaged binary passed
the same compact-prompt checker cell at 27.133 s. It remains an experiment,
with no default change.

## Tool search for MCP catalogs (implemented, opt-in)

`ZERO_EXPERIMENTAL_TOOL_SEARCH=true` (or the umbrella `ZERO_EXPERIMENTAL`)
replaces individually declared MCP tools with two fixed tools once the visible
catalog reaches `ZERO_TOOL_SEARCH_MIN` tools (default 8):

- `tool_search` returns the names, descriptions and full input schemas of the
  best keyword matches, so a large catalog costs nothing on every turn and a
  server that connects late no longer invalidates the frozen prefix.
- `tool_exec` dispatches {tool, args} to the real MCP client with the same
  permission ask, argument coercion, doom-loop guard and truncation as the
  direct declaration.
- Calls to undeclared names are rewritten to `tool_exec` by
  `LLMRepair.repairToolCall` (the schemas ride on the dispatcher under the
  `zero.deferredToolSchemas` symbol, which the ToolFreeze spreads preserve).

The declared tool set is therefore constant for the session, which is what the
2026-09-15 measurement (one changed description cost a 92 s re-prefill at 32k)
says direct MCP declaration pays on every MCP catalog change.

Verified 2026-09-17 against the live playwright, peekaboo and minecraft
servers (100+ tools): search then dispatch worked, and a direct call to the
undeclared name `peekaboo_space` was rewritten to `tool_exec` by the repair
hook and executed for real. One bug found and fixed this way: embedding the
MCP Client object in the search listing, which JSON.stringify rejects as
cyclic; the test client is now cyclic too so it cannot come back.
