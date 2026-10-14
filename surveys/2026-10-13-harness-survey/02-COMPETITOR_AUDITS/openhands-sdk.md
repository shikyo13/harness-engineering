# OpenHands software-agent-sdk — source audit

Audited 2026-10-13 at head `a430c778bd8a720234ce4674014ac292a0326670` ("fix(sdk): skip sub-agents whose
skill is missing instead of failing", #5754). Shallow clone at
`~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/openhands-sdk`. Python SDK for durable
software agents. Paths relative to `openhands-sdk/openhands/sdk` unless stated. Monorepo also ships
`openhands-tools/` (real tool implementations), `openhands-workspace/` (docker/apptainer/remote_api/
agent_sandbox/cloud backends), `openhands-agent-server/` (HTTP/WS server that hosts tools in a sandbox),
and `clients/`.

## Architecture

Three-layer split, cleanly enforced:

- **Agent** (`agent/`) is a *stateless step function*: `Agent.step` (`agent/agent.py:694`) takes the
  conversation state, builds messages, calls the LLM once, returns events. "Agents are stateless and
  should be fully defined by their configuration" (`agent/base.py:107`). Mixins: critic, response
  dispatch, parallel execution.
- **Conversation** (`conversation/`) is the durable, event-sourced engine: `LocalConversation`
  (`conversation/impl/local_conversation.py`, 3,174 lines) and `RemoteConversation`
  (`conversation/impl/remote_conversation.py`) share `ConversationState`
  (`conversation/state.py:82`) over a `FileStore` abstraction (`io/base.py:6`, local + in-memory
  implementations).
- **Workspace / agent-server**: tools execute locally, inside docker/apptainer, or remotely against the
  agent-server over HTTP + WebSocket; the conversation logic is identical across backends.

### The loop, precisely

`run()`/`arun()` (`local_conversation.py:1902, 2093`) is a `while True` step loop that: holds a FIFO
state lock per step but **releases it while awaiting LLM/tool I/O** so user messages can land mid-run
(`local_conversation.py:2373`, comment "lock free while awaiting them so incoming user messages can");
checks `WAITING_FOR_CONFIRMATION` for confirmation mode (`:1984, 2201, 2295`); nudges once on a
repeating action-error streak before declaring stuck (`_check_stuck_or_nudge`, `:744-762`,
`stuck_detector.py:24-56` pattern windows); enforces `max_iteration_per_run` (default 500, `:219`) and a
**dollar budget** `max_budget_per_run` across agent + condenser LLMs (`:720-726`).

`Agent.step` internals (`agent/agent.py:735-850`): resolve routed-model metadata first so the *real*
endpoint context limit drives condensation on step one (`:744-746`); build messages from the cached,
incrementally-maintained **View** (`context/view/view.py:22`, `state.view`) — not by re-walking history;
run the condenser over the View (`prepare_llm_messages`, `:750`); if it emitted a `Condensation`, persist
it as an event *before* sampling the next action (`:754-756`); call `llm.generate`; on context-overflow
errors trigger a condensation **retry** with condensed history (`:838-848`); dispatch multiple tool
calls concurrently via `asyncio.gather` (`:323`), with security analysis gating (`llm_security_analyzer`
forced on, `:486-494`) and user-rejection observations (`event/llm_convertible/observation.py:86`).

### Durability and resumability (concretely)

- The event log **is** the conversation: one JSON file per event, `event-{idx:05d}-{event_id}.json`
  (`conversation/persistence_const.py`), appended through `EventLog` which is "thread-safe and
  process-safe" using a length marker plus disk resync (`conversation/event_store.py:34-310`).
- Events form a **tree**: each append stamps a `parent_id` (`state.py:304`), the active branch leaf is
  resolved on read (`state.py:263`, `event_store.path_to_root:110`), and `conversation.fork()` copies
  events to start an alternative branch (`local_conversation.py:797, 174`). Condensations are
  non-destructive events, not history rewrites.
- `base_state.json` auto-persists on any public field assignment (`state.py:596-621`) — execution status
  (`ConversationExecutionStatus`: idle/running/paused/waiting_for_confirmation/finished/error/stuck,
  `state.py:48-58`), agent identity, secrets (encrypted), agent runtime state.
- Resume = `ConversationState.create(persistence_dir=...)` (`state.py:455-567`): loads base state,
  validates the provided Agent matches the persisted one (tools must match; agent identity mismatch
  fails), replays events, rebuilds the View. No DB, no server required.
- For ACP-backed agents whose remote session storage was wiped, `event/resume_transcript.py` renders the
  durable event history into a "bootstrap-prompt resume" opening message (`RESUME_CONTEXT_MARKER`,
  truncation budgets 60k/8k/2k chars) — durable events as fallback over provider-side sessions.

### Condenser (context compression), precisely

`CondenserBase` → `PipelinableCondenserBase` → `RollingCondenser` (`context/condenser/base.py:16-107`);
`LLMSummarizingCondenser` (`context/condenser/llm_summarizing_condenser.py:54`): defaults
`max_size=240` events, `keep_first=2` (`:64-67`); condenses for three reasons — unhandled condensation
request, token limit exceeded (counted against the *routed* model's real limit), event-count overflow
(`:136-173`); severity split: TOKENS and REQUEST are **HARD** (condense now or the request fails),
EVENTS is **SOFT** (`:175-204`). Forged tail events summarize forgotten events
(`_get_forgotten_events`, `:278`) with `hard_context_reset` as the nuclear option (`:355`). Default agent
config tightens to `max_size=80, keep_first=10` (`agent/base.py:298-299`). A `PipelineCondenser`
composes condensers; `handles_condensation_requests` lets the agent react to provider-side compression
signals.

### Tools: typed actions/observations

`ToolDefinition[ActionT, ObservationT]` is a generic pydantic class with discriminated-union
serialization (`tool/tool.py:347, 362-369`), `ToolExecutor` ABC (`:283`), `ToolAnnotations` for
readOnly/destructive metadata (`:215`), and **client tools** that execute in the UI process and are
recovered on resume (`client_tool.py`; `_recover_persisted_client_tools`,
`local_conversation.py:653`). Real tools live in `openhands-tools/openhands/tools/`: `file_editor`
(exact `old_str`/`insert_line` str-replace — **no fuzzy cascade**, `file_editor/impl.py:64-100`),
`terminal`, `apply_patch`, `grep`/`glob`, `task`, `delegate`, `workflow`, `task_tracker`, `browser_use`,
`ask_oracle`. Built-ins in-sdk: `finish`, `think`, `switch_llm` (live model swap mid-run),
`classify_and_switch_llm`, `invoke_skill`, `vision_inspect` (`tool/builtins/`).

### Sub-agents with skills (the head commit's area)

Sub-agent definitions are markdown+frontmatter files: `AgentDefinition` with `name`, `tools`,
`skills: list[str]` extracted from frontmatter (`subagent/schema.py:198-216, 72-82`), loaded per scope
(`subagent/scope.py`, `registry.py`). Execution: the `TaskTool` (`openhands-tools/openhands/tools/task/`)
spawns **child `LocalConversation`s** managed by `TaskManager` — each task is itself a full durable,
resumable conversation persisted under the parent's sub-agents directory (`task/manager.py:65-132`),
with `TaskStatus` lifecycle and head-commit behavior: skip a sub-agent whose referenced skill is missing
instead of failing the run (HEAD `a430c77`). Skills themselves are fetch/trigger/execute packages
(`skills/`).

### LLM abstraction

One pydantic `LLM` class over litellm-style providers with `num_retries=5`, exponential backoff
(`retry_multiplier=8.0`, min 8 s, max 64 s; `llm/llm.py:379-382`), tenacity-based `RetryMixin` with a
`retry_listener` hook (`:124, 882`), one auth-refresh + one retry per call chain (`:708-716`),
`RouterLLM` that selects a sub-model per message content with fallback ordering
(`llm/router/base.py:28-117`), `fallback_strategy.py` for provider failover, per-call
`llm_call_context_scope` for attribution, and a shared cost budget across agent/condenser/router calls.

## Strengths

- The event log as single source of truth with per-event files, process-safe append, branching, and
  non-destructive condensation events is the cleanest durability model in the survey set — resume is
  literally "load the files back", zero infrastructure.
- Agent/Conversation separation (stateless step vs durable engine) is exactly the decomposition Zero V2
  is converging on, and here it is enforced by the API shape.
- Hard vs soft condensation requirements and condensation-as-event with post-overflow retry is more
  honest than silent history rewriting.
- Mid-run `switch_llm` + budget ceilings + release-the-lock-during-I/O steering are practical production
  touches.
- Sub-agents are *the same durable primitive* as the parent, recursively.

## Weaknesses

- `LocalConversation` is 3,174 lines with `run()` and `arun()` as parallel near-duplicate state machines —
  the classic sync/async fork; every control-flow feature (confirmation, stuck, budget) is implemented
  twice.
- Durability is step-granular and process-local: a crash mid-`llm.generate` re-enters at the last event
  boundary with no durable in-flight marker, and there is no cross-process/clustered execution
  ownership — file locking is the only guard.
- Admission and execution are not separated: `send_message` appends a MessageEvent that a running loop
  opportunistically consumes; there is no admitted-vs-promoted vocabulary, so steering semantics depend
  on exactly when the loop releases its lock.
- Per-event JSON files: index rebuilds by disk scan (`event_store.py:320`), and branching leaves stale
  branches as garbage with no GC in sight.
- Exact-match-only file editor pushes all edit-recovery burden onto the model.
- Stuck detection is pattern-window heuristics only — no LLM double-check, no recovery-turn (aborts to
  STUCK status).

## Works vs unproven

- **Source-verified**: event store append/resync/marker logic, branch resolution + fork, auto-persist
  base state, resume-from-disk with agent validation, condenser requirement tiers + overflow retry,
  stuck nudge-then-stuck, parallel tool dispatch, client-tool recovery on resume, TaskManager
  child-conversation persistence, retry/backoff and router fallback, WS event forwarding with reconnect
  (`remote_conversation.py:149-296`).
- **Documented-only / not executed here**: docker/apptainer/agent_sandbox backends actually provisioning,
  agent-server persistence semantics, `RemoteConversation` state-sync consistency under partition,
  evals, cloud targets.
- **Implemented but niche**: ACP agents (`acp_agent.py`), automation/marketplace/plugin dirs not traced.

## What Zero Code should learn

1. **Hard vs soft compaction requirements + condensation as a durable event**
   (`llm_summarizing_condenser.py:175-204`, `agent/agent.py:754-756`): persist the compression itself in
   history, mark token-pressure as "must happen before next request", and retry once through the
   overflow path. Zero's compaction summaries should become replayable events, not mutations.
2. **Resume-by-files with identity validation** (`state.py:455-567`): refusing to resume when the agent's
   tool set differs from the persisted one prevents the class of bugs where a session runs with tools its
   history references but its registry lacks.
3. **Durable events as bootstrap transcript for wiped provider sessions**
   (`event/resume_transcript.py`): when the provider-side session/cache is gone, replay the durable log
   into a single bounded header message — directly applicable to Zero V2 crash recovery (baseline
   finding 3: post-crash continuation needs an explicit design; this is a cheap, lossy-but-honest one).
4. **Release the execution lock across provider I/O and treat arriving user messages as first-class
   events** (`local_conversation.py:2373, 2311`): Zero V2's steer vocabulary should ride the same
   mechanism — admission writes, the running drain observes — instead of a separate wake channel.
5. **Per-run cost ceilings across all auxiliary LLM calls** (`:720-726`): one budget covering
   agent + condenser + router is trivial to implement and the only honest spend guard when local models
   make token costs meaningless (baseline finding 7).
6. **Sub-agents as recursive instances of the same durable conversation primitive**
   (`task/manager.py:96-132`) — simpler than a bespoke subagent runtime and free-rides resumability.

## Durable execution: OpenHands vs Zero V2

OpenHands' "durable" = append-only event log + auto-persisted control state + resumability in a fresh
process, with steering via events and pause/confirmation as persisted statuses. Zero V2 additionally has
what OpenHands lacks: durable *admission* (session_input) distinct from execution, an explicit
promote-at-boundary vocabulary, and provider-turn-scoped reload of history; OpenHands has what Zero V2
lacks: persisted execution status that survives and is checkable from another process, conversation
branching, and a file-store backend abstraction that makes "resume in a different container" a
configuration change. The gap between the two models is one design: Zero V2's session_input +
base-state-style execution status would reach OpenHands durability without adopting its loop.

## Classification

All file:line claims read in source at head `a430c778bd8a` (source-verified). Container/agent-server
runtime behavior, RemoteConversation consistency, and eval claims: documented-only.
