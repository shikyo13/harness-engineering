# Goose — source audit

Head `e9e0ab62c273` (2026-10-13). Source-verified claims cite paths relative to `crates/`.

## Identity / lineage

Remote is `aaif-goose/goose`, and that **is** block/goose, not a fork: `GOVERNANCE.md` — "Founded by Block and now
stewarded by AAIF (Agentic AI Foundation)" of the Linux Foundation; README carries the LF insights badge. The org
moved after donation; block/goose redirects here. Rust workspace, Apache-2.0.

## Architecture

Rust workspace mid-extraction into focused crates: `goose` (core lib: agents, recipe, scheduler, session,
providers glue, permission, hooks, skills), plus `goose-agent` (protocol-ish machine/tool/events/inference seam),
`goose-context-management`, `goose-providers` + `goose-provider-types`, `goose-mcp` (bundled MCP servers:
computercontroller, peekaboo, memory, tutorial), `goose-cli`, `goose-sdk` (+ `goose-sdk-types`, custom
notifications/requests for embedding), `goose-local-inference`, `goose-roaming` (experimental P2P),
`goose-acp-macros` (ACP: run over existing Claude/ChatGPT/Gemini subscriptions — README).

Agent loop is a **re-entrant state machine over persisted conversation state**
(`goose/src/agents/state_machine/mod.rs:1` — "Callers persist incoming messages, construct `Step`s … and choose
whether to call `StateMachine::step`, `apply`, or `run`"). ~21 composable operations in
`state_machine/ops_*.rs`: llm, toolcalling, compaction, steer, subagent, retry, maxturns, stop-hook, skills,
recipe, bang-shell, tool-pair-compaction, unknown-tool, tool-approval, doctor, empty-response, exit-on-error.
Correctness detail: batch completion is decided by tool-request/tool-response id matching, not by message role
tail (`mod.rs:36-50`).

**Extensions are MCP servers.** External ones are spawned subprocesses over stdio
(`goose/src/agents/extension_manager/stdio.rs`, `goose/src/subprocess.rs`, `goose-mcp/src/mcp_server_runner.rs`);
~13 are compiled in as "platform extensions" (`goose/src/agents/platform_extensions/mod.rs:37-83`): developer,
analyze, chatrecall, code_execution, ext_manager, scheduler, summarize, summon, todo, tom, apps. Per-session
enabled-extension state is persisted and restored (`session/extension_data.rs`, `EnabledExtensionsState`).

## Key techniques (file:line)

- **Steering.** `state_machine/ops_steer.rs:1` — "Adds queued user guidance when the agent is between model and
  tool turns"; an `Arc<Mutex<VecDeque<Message>>>` steer queue drained at the safe boundary (same idea as Zero
  Code's V2 promote-at-boundary, done in the middle of a turn).
- **Recipes = declarative task specs.** `goose/src/recipe/mod.rs:43-83`: version/title/description,
  `instructions` or `prompt`, pinned `extensions`, `Settings{goose_provider, goose_model, temperature,
  max_turns}` (:93-107), typed `parameters` (required/optional + input types, :158-208), `response.json_schema`
  structured output (:110-114), `retry` config, and `sub_recipes` (:79). `SubRecipe{name, path, values,
  sequential_when_repeated}` (:120-127) — parameterized children, explicitly parallel or serialized repeats;
  `ensure_summon_for_subrecipes` (:255) auto-injects the delegation tool. YAML format + variants normalized by
  `recipe_extension_adapter.rs`; validation gate `validate_recipe.rs` (schedule-safety, :392); parameterized
  templates via `template_recipe.rs`/`build_recipe` (:10 imports in scheduler).
- **Scheduler (cron daemon + agent-callable tool).** `goose/src/scheduler/full.rs`: `tokio_cron_scheduler`
  `JobScheduler` running *validated recipe files* (symlink-safe open + size cap, :37-60) with per-task
  `CancellationToken` dedupe map (:32-33); runs the same `Agent` + permission manager as interactive sessions
  (:15-16). The agent itself can create schedules via platform extension tool `scheduler__manage_schedule`
  (`platform_extensions/scheduler.rs:16-17`, `agents/schedule_tool.rs`). Docs:
  `documentation/docs/guides/recipes/` (index, recipe-reference, subrecipes, session-recipes, storing-recipes)
  + `goose scheduler` command.
- **Session→recipe extraction.** `goose-cli/src/recipes/extract_from_cli.rs` — a finished run becomes a reusable
  recipe (params + sub-recipes); plus `search_recipe.rs`, `recipe_deeplink.rs`, `local_recipes.rs` library,
  `workflow_recipes/` examples.
- **Subagents are child sessions.** `agents/subagent_handler.rs:31-70` clones the parent's session config, swaps
  in a generated subagent system prompt with max-turns bound; parent/child enforced via `SessionType::SubAgent` +
  `parent_session_id` checks (:115-127). `platform_extensions/summon.rs` exposes delegation over
  recipes/subrecipes/agent definitions (description budget 160 chars, `DELEGATED_META_KEY` tagging delegated
  messages). No "advisor" feature exists (grep: only "advisory lock" in goose-roaming) — delegation is
  summon + child sessions.
- **Code mode = dynamic tool loading (verified).** `documentation/docs/guides/managing-tools/code-mode.md`: tools
  from enabled extensions discovered on demand and batched into one programmatic execution to protect the context
  window; backed by the built-in `code_execution` platform extension. This is Goose's answer to Zero Code's
  tool_search/codemode layer — discovery-by-programming rather than a search tool.
- **Compaction.** `goose-context-management/src/summarize.rs`: a removal ladder `REMOVAL_PERCENTAGES = [0, 10, 20,
  50, 100]` (:17) — escalate summary aggressiveness until it fits — with `StructuredSummary` (structured-output
  schema) and versioned prompt templates (`prompts/compaction.md`, `compaction_summary.md`). Context limits
  resolved per provider with user override (`goose/src/context_limit.rs:7-25`); `ops_tool_pair_compaction` drops
  stale tool-request/response pairs; `agents/large_response_handler.rs` truncates oversized tool outputs.
- **Sessions in SQLite.** `goose/src/session/session_manager.rs`: sqlx `SqlitePool`, WAL (:1068), versioned
  migrations under `BEGIN IMMEDIATE` (:1088-1109), indexed messages (:3428); legacy YAML migration
  (`session/legacy.rs`), auto session naming (`session_naming.rs`), HTML/Markdown export, and cross-session
  SQL recall via the `chatrecall` extension (`session/chat_history_search.rs` returns message hits grouped by
  session). Resume: `goose session --resume` (docs `sessions/`).
- **Edit approach.** `platform_extensions/developer/edit.rs:111-146`: native Rust `file_edit` = exact
  `string_replace(before, after)` on read content, error on mismatch, line-count receipt. No fuzzy cascade, no
  read-before-edit enforcement.
- **Provider abstraction.** `goose-providers/src`: anthropic, openai, google, azure_foundry, bedrock (via
  context-limit special-case), databricks, snowflake, ollama, openrouter, openai_compatible, plus
  **`declarative/`** — providers defined by YAML definitions (`declarative/definitions/`), `local_inference`, and
  realtime voice transports (`live*.rs`). Subscription reuse through ACP.
- **Permission judging.** `goose/src/tool_inspection.rs:11-31`: pluggable inspectors return
  `Allow/Deny/RequireApproval` with a confidence score; `permission/` + `tool_confirmation_coordinator` +
  `ops_tool_approval`. Hooks (`hooks/`, `ops_stop_hook`), skills (`skills/` incl. builtins), and `analyze`
  (secondary-model review) round out the platform set.

## Strengths

- The strongest declarative task-automation stack surveyed so far: recipe YAML (params, pinned model/provider,
  JSON-schema response, retry, max_turns) × cron scheduler × agent-callable scheduling × sub-recipes ×
  session→recipe extraction × deep links. Long-horizon runs are durable artifacts, not chat transcripts.
- State-machine loop with named, composable, individually-tested ops is a clean decomposition of exactly the
  concerns Zero Code has inlined in `SessionPrompt.runLoop`.
- Everything is MCP — extensions, platform tools, and code-mode are the same protocol; the same `Agent` serves
  CLI, desktop, scheduler, and SDK embedders (`goose-sdk`).
- SQLite session store with real migrations + cross-session recall.

## Weaknesses

- Editing is exact-string only — weakest edit reliability story in the survey; zero measured recovery machinery.
- Compaction is template-summary based; no cache-awareness or verbatim-tail concept (cf. Zero Code baseline §context).
- `goose` crate remains a monolith; the crate extraction (goose-agent, context-management) is in flight, so
  boundaries are still shifting (`goose-agent/src/machine.rs` is a thin seam, not yet the loop).
- Feature/release gaps: roaming is explicitly "not included in released binaries"
  (`documentation/docs/guides/roaming-agents.md` warning); several platform extensions are opt-in per session.
- No durable admission/idempotent-retry semantics for the interactive loop (steer queue is in-memory;
  scheduler recipes are the only durable unit).

## Works (source-verified) vs unproven

Source-verified: state machine + steer, recipe format/validation/templates/sub-recipes, cron scheduler +
`scheduler__manage_schedule`, summon child sessions, code mode plumbing, compaction ladder + structured summary,
SQLite persistence, declarative/ACP providers, permission inspector scaffold.
Documented-only or unproven: managed tool execution / malware checking quality (`extension_malware_check.rs`
presence ≠ efficacy), marketplace extension ecosystem behavior, voice features, roaming (experimental),
code-mode context savings in practice, recipe UX at scale, subagent output quality.

## What Zero Code should learn

1. Recipe-shaped durable automation: a YAML task spec (params, pinned model, response JSON-schema, retry,
   max_turns, extension pinning) admitted either by cron, by API, or by an agent-callable tool — a natural
   external layer on Zero Code's durable `session_input` + scheduler-free V2; Zero Code currently has no
   declarative recurring-task story.
2. Session→recipe extraction: cheap, high-leverage way to turn ad-hoc runs into automation assets.
3. Subagents as *child sessions* (`SessionType::SubAgent` + `parent_session_id`) instead of in-loop fibers:
   durable, resumable, separately observable delegation — directly applicable to V2's unfinished subagent slot.
4. Steer queue drained between model turn and tool turn is the minimal in-turn steering complement to V2's
   promote-at-boundary admission.
5. Escalating compaction ladder with a structured-output summary schema is a simple fallback policy Zero Code's
   compaction could adopt for hard fits.
6. Do not copy: exact-string editing.
