# 03 — Antigravity CLI vs Gemini CLI: a migration case study

Written 2026-10-13. Evidence base: source audit of `gemini-cli` at `42c51822084a` (v0.65.0-preview, 2026-10-13),
the docs-only `antigravity-cli` repo at `9eebbedf051f` (v1.3.4, 2026-10-13) plus its changelog dated via the GitHub
Releases API, black-box probes of the published `agy` v1.3.4 binary (now owner-deleted; `agy.strings` retained), and
the installed `gemini` 0.36.0 CLI. Classification tags follow the two audits: **[source]**, **[published]**,
**[observed]**, **[inferred]**, **[documented-only]**, **[unavailable]**. Deep detail lives in
`02-COMPETITOR_AUDITS/gemini-cli.md` and `02-COMPETITOR_AUDITS/antigravity-cli.md`; this file is the comparison.

## 1. The headline finding

**Antigravity CLI is not a rename, and not a fork. It is a clean-room-adjacent reimplementation in another language,
built on a different engine, that deliberately kept Gemini CLI's *migration surface* (config home, context files,
extension manifests, API-key route) while sharing almost none of its implementation or even its tool vocabulary.**
[observed + source + published; strongest single piece of evidence: tool-name sets collide on only `grep_search`
and `replace`, while `~/.gemini/config`, `GEMINI.md`, `.gemini-plugin`, `${extensionPath}` and `GEMINI_API_KEY` are
all explicitly honored (changelog 1.0.7, 1.1.13, 1.1.22).]

Both repos are alive on the audit date (gemini-cli HEAD 2026-10-13). "Successor" is a product-posture claim about
the consumer terminal experience, not repo death. Qwen Code is the one that actually froze the Gemini CLI codebase
(a v0.8.2 fork with the original file names still shipping — source-verified by file correspondence), which makes
the pair an unusually clean natural experiment in *iterate-in-place* vs *rewrite-and-migrate*.

## 2. Gemini CLI's original architecture (source-verified baseline)

Layered TypeScript loop: `GeminiCore.sendMessageStream` (`core/client.ts:940`) → serialized
`GeminiChat.sendMessageStream` (`core/geminiChat.ts:494`, `sendPromise` chain at :500, unanswered-tool-call repair
:528-535) → nested retry generator with separate connection-phase and mid-stream budgets (`geminiChat.ts:663-790`,
`MID_STREAM_RETRY_OPTIONS.maxAttempts=4`) → `Turn` typed event converter (`core/turn.ts`) → tool scheduler state
machine `Validating → Scheduled | AwaitingApproval → Executing → Success|Error|Cancelled`
(`scheduler/scheduler.ts:309-313`, one active batch :446, approval-time `tool-modifier.ts`). Headless driver is a
`while(true)` around the client (`packages/cli/src/nonInteractiveCli.ts:316`).

Its three genuinely distinctive systems, verified in source:

1. **Measurement-driven context management** (`context/`): API-reported `usageMetadata` prompt-token ground truth
   feeds a deficit-threshold pipeline (`core/client.ts:859-867` → `contextManager.ts:377-418`) over composable
   processors — rolling summary, state snapshot, **tool-output masking to disk with a 50k-token protection window**
   (`toolOutputMaskingService.ts:25-41`), distillation, truncation.
2. **Two-tier loop detection with recovery, not abort**: heuristic (5 identical calls / repeated 50-char windows,
   `loopDetectionService.ts:29-33`), then a rare LLM double-check with confidence-scaled interval (threshold 0.9,
   :41-64), and on detection one injected "take a step back" feedback turn (`client.ts:1300-1328`).
3. **JSONL sessions that are patch-aware**: message records + `$patch` + `$set` + `$rewindTo` replayed on resume
   (`chatRecordingService.ts:79-104,307-422`), plus shadow-git `/restore` (`gitService.ts:229`).

Next-speaker LLM continuation ("Please continue.") still ships but is **off by default** (`config.ts:1288`) — a
documented technique that stopped paying for itself. Known weaknesses: two parallel context systems behind a flag,
god-object core (`config.ts` ~4k lines), no durable admission — JSONL is replay state, not execution state.

## 3. Antigravity's observable architecture

Repo is docs-and-issues only (1 squashed commit, 969-line undated changelog, 85-line README, two example scripts)
**[source-of-repo]**. The engine: a Go 1.28 binary built from a Google internal CL (`go version ./agy` →
`go1.28-20260921-RC02 cl/985595000`) **[observed]**, 181 MB single static file, self-updating, 75 releases in 19
weeks **[observed]**. String-mining reveals the internal loop vocabulary (`BattleMode` ×839 with durable trajectories,
"step handlers", "final invocation of a turn"; `PolicyGuardian` pre-tool checks; `google3/` paths; gRPC to
`v1internal.CloudCode` and `exa.language_server_pb.LanguageServerService`) **[inferred]**. The same engine backs
Antigravity 2.0 desktop: SQLite conversation import from desktop (1.0.4), shared permission merge (1.0.5), shared
`~/.gemini/config`, `antigravity_hub`/`antigravity_desktop` strings **[published + inferred]**.

Long-running execution, all from changelog prose (behavior-level, unusually verifiable prose) **[published]**:
durable queue with explicit `Queue | Send Immediately` semantics (steer equivalent); `/goal` unbounded goal loop;
`schedule` tool with timers and `MaxIterations` wake conditions; `manage_task`; daemon background tasks outliving
turns; auto-resume of conversations with active turns after restart (1.2.4); unlimited default headless timeout
(1.2.6); `/rewind`/`/fork` from per-step workspace snapshots in SQLite (1.2.6); `remote-control` as a reboot-persistent
daemon (1.2.0); subagents as child conversations with `conversation_id`/`log_uri` trajectory correlation, worktree
`branch`/`share` workspace modes, model tiers, roster injection, `@<subagent>` addressing.

## 4. What Google changed, and what each change was *for*

Concrete transitions with dates pinned via the Releases API **[observed for dates; published for content]**:

| When | Change | Why it reads as the lesson |
|---|---|---|
| pre-June | Runtime TS/Node → Go static binary (SIGILL/AES-NI/ARM64 fix trail 1.0.8–1.0.10) | Zero-dependency reach: SSH, Termux, low overhead. Gemini CLI's npm/esbuild startup and Node footprint were the thing being escaped. |
| 1.0.4–1.0.5 (Jun 1–) | Sessions become SQLite, interchange format with desktop; permissions merged across three layers | The CLI was demoted from *the product* to *a frontend* of a shared engine. JSONL-per-project couldn't carry desktop-grade resume/search. |
| 1.1.0 (Jul 8) | `request-review` becomes the default mode; `/fast` deleted | The iterate-fast posture of `--yolo`-style flows was rejected in favor of diff-preview-gated autonomy. |
| 1.1.8–1.1.15 (Jul 28–Aug 19) | print mode → typed machine API: closed-vocabulary NDJSON `step_type`, `--json-schema`, `AGY_ERROR`, exit-code discipline, `--input-format stream-json` long-lived drivers | An agent CLI is now also an *interface other agents/CI build on*. This is the surface an eval harness wants (see 06). |
| 1.1.17 (Aug 20) | "consolidating onto a single execution path" | Explicit admission that two execution paths had coexisted — exactly Zero Code's V1/V2 disease, fixed with a headline entry. |
| 1.2.x (Sep) | remote-control daemon, unlimited headless timeout, tool retirement w/ opt-in retention, 20k rules budget protecting skills/subagents/MCP, denial-respect rule, retry caps 4min→30s | The autonomy package: unattended long runs, context-budget *partitioning* instead of global eviction, and permission semantics hardened from user-trust incidents. |
| 1.3.5 (Oct 14) | Auto mode with a **separate reviewing agent** for sandbox-escaping commands; `/compact` | Permission escalation increasingly delegated to a cheap second model (mirrors Codex "Guardian" — convergence signal). |

**What was retained**: config home `~/.gemini/config`, `GEMINI.md`, `.gemini-plugin` manifests, `${extensionPath}`,
`GEMINI_API_KEY`/`GOOGLE_GEMINI_BASE_URL`, `mcp_config.json`, MCP-first posture, a `migrate-workflows` skill.
**What was discarded**: the GeminiChat loop and its next-speaker mechanism entirely, JSONL session format,
`--approval-mode`/`--yolo` vocabulary, ACP mode, the `--worktree` flag (worktrees moved to subagent workspace modes),
npm distribution. **Notably NOT retained from Gemini CLI's own best ideas either**: the measurement-driven context
pipeline is not mentioned in Antigravity prose (its compaction story is budget-based: full-window sizing, checkpoint
boundaries, rules budget) [documented-only], and next-speaker continuation vanished with the old loop.

## 5. Runtime & agent-loop comparison

| Dimension | Gemini CLI [source] | Antigravity CLI [observed/inferred/published] |
|---|---|---|
| Language/runtime | TS/Node, esbuild bundle, npm | Go 1.28 static binary from google3, self-updating |
| Loop | in-process async generator chain; `maxSessionTurns` bound | `BattleMode` engine with durable step trajectories, invocation limits, hooks on final invocation; continuation implied tool-call-driven (`NO_TOOL_CALL` self-correction, 1.2.4) |
| Durable execution | no (JSONL = replay state) | per-step SQLite snapshots; auto-resume of active turns after restart (1.2.4) [published] |
| Steering | mid-turn queue absent; headless driver sequential | `queuedMessages: queue \| send-immediately`, `/btw` side trajectory, remote interruption [published] |
| Tools | ~19 base declarations + background-process tools; edit cascade exact→flexible→regex→fuzzy (`edit.ts:82,145-347`) | `view_file`/`write_to_file`/`replace`/`edit_file`/`run_command`/`manage_task`/`invoke_subagent`/`schedule`/… [inferred]; schema validation bidirectional, validation failures returned to the model (1.2.4) |
| Permissions | `default|auto_edit|yolo|plan`, `--policy` files | `PolicyGuardian` deterministic pre-checks [inferred], 3-file rule merge, non-regex default, pinned-subcommand allow-suggestions, per-OS sandbox with `proceed-in-sandbox`, plan-review axis separate from permission bypass (1.3.2) |
| Context | token ground truth + processor pipeline; masking to disk | window-proportional budgets; 20k rules floor; compaction checkpoints; `/compact` [published]; percentage exported to statusline [observed] |
| Sessions | JSONL + `$patch`/`$rewindTo`, shadow-git `/restore` | SQLite per conversation, WAL, FTS5 search, `@conversation:`, step snapshots, fork, desktop interchange |
| Machine API | `--output-format json`, ACP mode | bidirectional closed-vocab NDJSON, `--json-schema`, `AGY_ERROR`, exit codes, quota-free read-only commands |

## 6. Where the comparison cannot be measured, and why

No head-to-head execution on identical tasks was possible in this run: `agy -p` demands OAuth before *any* model
turn (probe P11/P12 — even read-only slash commands authenticate), and operating notes forbid credentials or new
accounts → **comparative task outcomes are blocked**, now permanently for this machine under the binary-download
rule unless the owner approves a re-download and signs in. No measured Antigravity-vs-Gemini performance numbers
exist in published sources either (no benchmark prose in the 969-line changelog). **Every outcome-level claim below
is therefore structural, not measured, and is marked as such.** The 06 experiment design specifies the
same-task/same-model runs (local model where possible, quiet-night timing) that could close this gap.

Structural confidence ranking (reasoned, not measured): Antigravity's durable-step SQLite + auto-resume + unlimited
headless timeout + remote-control give it a *strictly stronger substrate* for long-horizon unattended work than
Gemini CLI's process-lifetime loop — this is architecture, and it is well-evidenced. Whether its *task outcomes*
beat a modern Gemini CLI on identical models is unknown; Gemini CLI's context-pipeline and loop-detection-recovery
machinery are, on source evidence, more principled than anything the Antigravity changelog shows.

## 7. What Google learned from Gemini CLI (the assignment's key question)

1. **The loop's durability was the weak point, not its intelligence.** Everything new (SQLite steps, snapshots,
   auto-resume, queue vocabulary, remote control) makes the run survive process/device/session boundaries. Gemini
   CLI's loop kept quality innovations (ground-truth compaction, loop recovery) invisible to users precisely because
   the run still died with the process.
2. **One engine, many frontends beats one smart CLI.** The rewrite bought desktop parity, a Python SDK, and
   multi-model support (Claude slugs in the catalog) that a Gemini-first TS CLI structurally resisted.
3. **A stable machine interface is worth weeks of explicit investment** (the 1.1.8–1.1.15 sprint). Agents and CI are
   now first-class users of a coding CLI.
4. **Permission UX, not model quality, gates unattended autonomy**: request-review default, pinned subcommand
   allow-lists, denial-respect, artifact-review axis, and now a reviewing agent — all reaction to real-incident
   classes a `--yolo` flag cannot cover.
5. **Depreciate softly, migrate loudly**: retire tools to opt-in, keep the config home, ship a migration skill.
6. **Two execution paths are a named, fixed bug class** (1.1.17) — they hit it on the *other* side of the rewrite.

## 8. Lessons for Zero Code (each maps to a baseline finding or roadmap item)

1. Finish V1→V2 convergence and treat it as a quotable milestone (baseline finding 1; Antigravity 1.1.17).
2. Make Zero's headless surface a *closed contract*: stable `step_type` enum NDJSON in both directions, structured
   `error` objects with retryability, exit-code discipline, quota-free read-only commands, `--json-schema` result
   enforcement. Feeds 06 (zero-eval speaks exactly this dialect) and any external SDK.
3. Durable execution *status*, not just durable history (baseline finding 3): per-step snapshot rows, restart
   auto-resume of eligible sessions, `denied_actions` reporting. Gemini CLI's per-call scheduler status machine is
   the in-source pattern; Antigravity's SQLite steps are the product-grade version.
4. Adopt Gemini CLI's three underrated source patterns directly (they are open-source and testable): token
   ground-truth compaction triggering, loop-detect-with-recovery-turn, tool-output masking with a protection window.
   Antigravity's equivalent features are closed or absent — the predecessor beats the successor *in source* here.
5. Partition the context budget with named floors (rules/skills/subagents/MCP) instead of one global compaction
   lever; export used-percentage + state to a JSON statusline contract so users (and evals) can watch pressure.
6. Permission roadmap: keep tree-sitter bash verdicts, add pinned-subcommand allow-suggestions, keep plan/checkpoint
   review orthogonal to any bypass flag, and pilot a cheap-model reviewing lane for irreversible actions (both
   Codex Guardian and Antigravity Auto arrived here independently — strong signal, test before adopting).
7. Keep subagent transcripts durably queryable (`conversation_id`/`log_uri` correlation, `@conversation:` FTS search
   over sessions) — Claude Code doesn't persist them, Antigravity makes them a first-class object; both gaps are
   opportunities for Zero (see 07 synthesis).

## 9. Remaining unknowns (do not invent past these)

BattleMode's real loop internals (prompt assembly, tool selection, retry policy); whether desktop embeds the binary
or shares a hub process/daemon boundary; the client/server split implied by `v1internal`/`exa` gRPC endpoints; the
Python SDK surface; `gemini-4-argon`'s identity; any Antigravity behavior requiring a model turn (compaction
quality, sandbox UX, remote-control flow, quota mechanics); Gemini CLI's official end-of-life posture (none found).
These bound what this case study can claim; §7's inferences are drawn from dated release behavior, which is
strongest for *direction* and weakest for *internals*.
