# P2 subsystem study — Providers and model tables

DRAFT — P2 study, written 2026-10-14, awaiting cross-check.

Scope: provider adapters, model metadata/quirk tables, effort/variant handling, streaming/error normalization, timeout
policy, and the mlx-serve engine integration incl. memory/outage economics. Zero Code behaviour is cited from
`docs/harness-engineering/spec/providers-engine.md` and `spec/agent-loop.md` (source read only for gaps); owner rules from
`PRIOR_WORK.md`. Picked audits: aider (ModelTable), qwen-code (compat quirks), codex (responses wire discipline) — the
strongest on model handling; kimi-code cited secondarily for budget auto-sizing. Repos read read-only at the §7 pins; the
the owner's earlier prototype untouched. Competitor paths are repo-root-relative (§7); Zero Code paths relative to this repo.

## 1. How each implementation works

### Aider — quirks as data, one YAML table (pin 5dc9490bb35f)

- **ModelTable shape.** `ModelSettings` dataclass, 21 typed fields — `edit_format`, `weak_model_name`, `lazy`/`overeager`,
  `reminder` ("sys"|"user"), `use_temperature`, `reasoning_tag`, `system_prompt_prefix`, `accepts_settings`, editor pairing
  (`aider/models.py:128-156`) — 357 entries loaded at import from `aider/resources/model-settings.yml`
  (`models.py:154-158`). Exact name match wins.
- **Format coercion.** Exact match → data; else `apply_generic_model_settings`, a ~160-line name-substring if-chain mutating
  the table (`models.py:437-598`): `/o3-mini` → diff + `"Formatting re-enabled. "`; `qwen3…235b` → `/no_think` prefix +
  sampling extras. Quirks are data *first, if-chain second*.
- **Effort.** No tier abstraction; reasoning models get `use_temperature=False` and `accepts_settings` appended inside the
  if-chain (`models.py:430-436`) — a capability list, the closest predecessor to a knob descriptor.
- **Retry/timeout.** Typed retryability table (one `retry: bool` + user description per litellm error,
  `aider/exceptions.py:12-56`); an unenumerated litellm class raises at *import* (`exceptions.py:76`). Retry 0.125 s
  doubling, stop at `RETRY_TIMEOUT=60` (`aider/coders/base_coder.py:1449-1488`, `models.py:26`); whole-request
  `request_timeout=600` (`models.py:28`) — a wall-clock timeout our owner rule forbids. `ContextWindowExceededError` ends
  the turn (no compaction); `FinishReasonLength` continues via `prefix=True` assistant prefill (`base_coder.py:1492-1505`).
  History cap `min(max(input/16,1024),8192)` (`models.py:358`).

### Qwen Code — capability descriptors + exactly-one-knob arbitration (pin 6212189f1f83)

- **ModelTable shape.** Presets are descriptors: `{id, contextWindowSize: 1M, capabilities.reasoning: {thinking,
  efforts:[low,medium,xhigh], defaultEffort:xhigh, disableField:'reasoning_effort'}, modalities}`
  (`packages/core/src/providers/presets/alibaba-standard.ts:109-140`); older hybrids use `disableField:'enable_thinking'`.
  Wire coercion gated by `isTieredEffortWireModel(model)`.
- **Effort/variant (the core lesson).** `selectDashScopeThinkingKnob` resolves *exactly one* of `enable_thinking` /
  `reasoning_effort` / `thinking_budget` across config layers because "qwen3.8 rejects reasoning_effort with
  thinking_budget" (`core/openaiContentGenerator/provider/dashscope.ts:56-135,426-437`); `withoutNullishThinkingKnobs`
  strips null-knob leakage (`:137-165`); an `enable_thinking:true` on-switch blocks lower off-switches without picking the
  tier; vision forces `preserve_thinking:true` for multi-turn reasoning continuity (`:438-476`).
- **Budget auto-sizing.** `clampOutputTokensToWindow = min(ceiling, window − prompt − margin)`, `margin = max(10_000, 5% of
  window)`, floored at `min(4_000, window − prompt)` (`core/tokenLimits.ts:50-100`): `prompt + max_tokens ≤ window` becomes
  structural (issues #5950 and #13252 closed by this shape) so compaction thresholds run against the full window.
- **Stream normalization / timeouts.** Nonstandard `reasoning_content` round-tripped with a guard for providers that 400 on
  echo (`openaiContentGenerator/converter.ts:66-81,843-852`); overflow errors force re-compaction (`llm-chat.ts:4432-4545`).
  No wall-clock turn timeout; caching is call-site engineering: session-keyed `prompt_cache_key` + two explicit breakpoints
  (`prefix-caching.ts:73-117`).

### Codex — one versioned table + provider-scoped policy (pin 6786dccbe726)

- **ModelTable shape.** `ModelInfo` (`codex-rs/protocol/src/openai_models.rs:408-505`) is the *only* behaviour source:
  `supported_reasoning_levels`, `apply_patch_tool_type` (freeform vs function), per-model `truncation_policy {mode,limit}`,
  `context_window`/`max_context_window`, `auto_compact_token_limit` (derived/clamped to 90 %, `:461,530-540`), `comp_hash`
  whose change forces compaction (`session/turn.rs:1374`), `effective_context_window_percent` (default **95**) with
  `usable_context_window()` (`:393-395,470-471,523-527`), `supports_parallel_tool_calls`, `tool_mode`. Shipped as
  `models-manager/models.json`, cached with identity (`models-manager/src/cache.rs`).
- **Timeout policy.** One per-provider object: `stream_idle_timeout_ms` default 300 000 ms, `stream_max_retries` default 5
  (cap 100) (`model-provider-info/src/lib.rs:66-73,506-517`) — both match Zero's spec numbers (300 000 ms inactivity abort;
  ≤5 retries, 2 s ×2 + 25 % jitter, 30 s cap, `retry-after` honoured): independent convergence with `spec/agent-loop.md`.
- **Error taxonomy → retry decision.** `CodexErr::retry_delay` enumerates every variant: terminal list
  (`ContextWindowExceeded`, `Interrupted`, `InvalidRequest`, …) → `None`; `ServerOverloaded`/`RetryLimit` honour *server
  advice only* (Retry-After, never local backoff); everything else exponential+jitter (`protocol/src/error.rs:410-458`).
- **Stream-error normalization.** `handle_response_stream_error` (`core/src/responses_retry.rs:55-210`): content-filter errors
  record a model-specific `ContentFilterGuidance` fragment *into the conversation* before retrying (`:68-83`); a
  feature-gated **unbounded connection-retry** path for external sampling waits 5 s→60 s cap with no attempt budget
  (`:22-23,90-118`) — the same doctrine as our engine outage wait, for cloud; exhausted budgets switch transport without
  bypassing the server's Retry-After deadline (`:120-140`).

### Kimi Code (pin b510fa9aec85, via audit) — traits as data

Declarative per-model trait hooks on generic OpenAI/Anthropic bases: auto-sized `max_completion_tokens = window − estimated
used context` when unset (`llm-kimi/trait.ts:83-105`), thinking `keep`, call-ID normalization; a models.dev-style remote
catalog maps entries into limits, effort ladder and capability flags (`human/models-dev/models-dev.ts:175-195`) — a new
model is a catalog edit, not a code change.

### Zero Code today (spec) — the baseline

Variants applied only when `user.model.variant` set and not `small`; output capped `min(limit.output, 32000)`; local
one-request serialization with body-read release (`provider/provider.ts:96-145`); flat in-stream refusal shaping incl.
context overflow + ≥500 retryable (`provider/error.ts:149-170,188-209`); engine manager adopt/health/restart/warm-up
(`engine/manager.ts:184-434`); memory waits judged against `/props` + refusal figures (reserve RAM/8 clamped 2–8 GB, no
deadline except session interrupt, `engine/memory.ts:91-160`); outage `/health` poll 5→30 s, retry uncounted
(`engine/outage.ts:36-59`, `retry.ts:232-242`); mlx timings survive via raw chunks → `providerMetadata.zeroEngine`
(`session/llm/engine-timings.ts:21-98`).

## 2. What is genuinely best, and why

1. **Codex's single versioned table as the only behaviour source** — decisions resolve through `ModelInfo` + `comp_hash`;
   changes are reviewable JSON diffs and comp_hash→forced-compaction keeps cached prefixes honest when the table changes.
2. **Codex's provider-scoped timeout/retry object** (idle 300 s, retries 5, server-advice supremacy) — precisely the shape
   our no-wall-clock owner rule generalizes to.
3. **Qwen's exactly-one-knob arbitration with the provider reject documented at the resolver** (`dashscope.ts:426`) —
   documented rejects become named, tested precedence rules instead of request-build races.
4. **qwen/kimi `completion = window − used` with margin and small-window floor** (`tokenLimits.ts:83-100`) — overflow
   becomes structurally impossible rather than a caught error.
5. **Aider's typed retryability table** — one row per error: retry?, user-facing cause — delivered as data, without the
   import-time raise over a third-party exception zoo.
6. **Our engine memory/outage waits** — no audited harness distinguishes "provider throttling" from "this Mac has no free
   RAM"; codex's unbounded connection-retry is closest and retries a refusal that will never self-heal. Keep ours; promote
   it into the shared taxonomy.

## 3. What to combine

- **Data-driven ModelTable replacing the prompt forks and gpt-\* if-codes.** The nine prompt `.txt` forks are "per-model
  behavior as data expressed as fork-code — every new model costs a file and drift between forks is invisible"
  (`05-AGENT_ENGINEERING_FINDINGS.md` §3 item 2). apply_patch is offered by model *name* — `gpt-*` minus `oss`/`gpt-4`
  (`spec/tools-editing.md`, gating `registry.ts:331-334`) — which an `applyPatchStyle` trait retires, exactly as codex's
  `apply_patch_tool_type` does. Shape = codex `ModelInfo` scope + kimi trait hooks + aider's reminder/prefix/lazy fields;
  resolution = exact id → *regex pattern in data* (aider's if-chain never returns) → provider default.
- **Budget auto-sizing.** `completion = window − used` (kimi `trait.ts:83-105`, qwen clamp) on top of the existing
  usage-or-estimate path (`session/overflow.ts:15-31`); keep `min(limit.output, 32000)` as the ceiling, add qwen's margin
  and ≥1 floor so a 32k-context local run cannot self-truncate.
- **Timeouts: preserve the owner rule, adopt the object.** No wall-clock on turns/tools/subagents/compaction (`PRIOR_WORK.md`;
  `spec/providers-engine.md` — a rebuild *must* preserve); express header/chunk inactivity (default 300 000 ms) plus the
  retry allowance as one provider-scoped `TimeoutPolicy` so a local engine widens them (the plugin already sets 30-minute
  streams) without code-path branches.
- **Engine memory/outage waits as provider-level policy, not plugin patches.** Today wired ad hoc at `processor.ts:724-805` /
  `retry.ts:232-242`; fold into the retry layer as first-class `WaitForCapacity{memory|outage}` decisions, generalizing
  codex's server-advice supremacy so engine `/props` + refusal figures act as the local provider's Retry-After. Input toward
  09 Phase 4's ModelTable-as-data idea (input, not plan).
- **Effort.** Keep the variant merge/apply gate (`user.model.variant`, not `small`); move per-family knob choice (Anthropic
  adaptive, Gemini budgets, OpenAI/OpenRouter efforts, GLM/Kimi/MiniMax cases) into table fields — `efforts`,
  `defaultEffort`, `disableField`, plus qwen-style documented rejects as `knobExcludes:
  [["reasoning_effort","thinking_budget"]]`.
- **Stream normalization.** Keep the flat-refusal classification (`provider/error.ts:149-170`); add codex's content-filter
  guidance injection, and use the engine's new prefill-refusal figures (`2026-10-07-engine-decode/README.md` §1, mlx-serve
  PR #784) as classification input for memory vs filter vs overflow.

## 4. Proposed design for the new core

One versioned table resolved exact→pattern→provider-default, plus one policy object each for timeouts and retries.
`ModelV2.Info` additions (today's schema has `limit.{context,input,output}`, `cost`, `variants` and no cache/quirk fields —
`packages/schema/src/model.ts:60-85`):

```ts
traits:     { applyPatchStyle: "codex"|"none"; editFormat; reminderPlacement; promptPreset /* replaces .txt forks */ }
context:    { window; usablePercent /* codex-style */; autoCompactLimit; compHash /* change forces compaction */ }
completion: { ceiling; autoSize: true; margin: {min, percent} /* qwen tokenLimits shape, floored ≥1 */ }
effort:     { knob; efforts; defaultEffort; disableField; excludes }
cache:      { mode: "explicit" | "implicit-prefix" | "none"; accounting }
  // per-provider cache accounting: explicit = cache_control-style cost rows; implicit-prefix = local engines
  // + OpenAI-style: bill cached_tokens from usage where reported, else derive from mlx-serve timings
  // (cached/prefill, engine-timings path); expose cache-state so the context engine can protect the stable
  // prefix (the 411x warm/cold gap, §5).
retry:      { maxAttempts; backoff {initial, factor, jitter, cap}; serverAdviceSupremacy: boolean }
timeout:    { headerInactivityMs; chunkInactivityMs; wallClock: "none" }  // owner rule encoded, not folklore
```

Error taxonomy matching `spec/agent-loop.md` behaviours one-to-one: `Fatal` (invalid request, auth); `RetryWithBackoff`
(5xx/transport/transient patterns — ≤5, current parameters); `RetryOnServerAdvice` (429/overloaded; Retry-After honoured;
engine refusal figures act as local advice); `WaitForCapacity{memory|outage}` (uncounted, no deadline except session
interrupt); `CompactThenRetry` (window overflow once, never on a memory refusal already judged "would not help"). Engine
manager/lifecycle/timings stay as spec'd; the manager publishes capacity so the provider layer pre-judges refusals instead
of discovering them mid-turn.

## 5. Measurable targets

- **Prefill:** cold ≤ measured 192.5 s at 100k (only on genuine first fill) and warm ≤ 1 s (measured 0.5 s at 100k,
  `docs/research/2026-09-15-harness-and-engine/measurements.md:163`). Gate: any table/variant change to the prompt prefix
  emits a cache-invalidation event (one tool-list edit recompute ≈ cold, 153.2 s, `measurements.md:164`); zero unintended
  invalidations per turn, measured on AC.
- **Decode floor:** sustained ≥ 28–29 tok/s on local turns (today's P0 runs, 2026-10-14; single-stream warm reference
  56.3–57.1 tok/s, `2026-10-07-engine-decode/README.md` §2 — the floor reflects shared-engine load; battery halves decode,
  `PRIOR_WORK.md`).
- **Zero spurious turn failures from engine refusals:** on the memory-ceiling repro (`2026-10-07-memory-ceiling/README.md`,
  hog.swift + the 2026-10-01/07 incidents) every memory refusal and outage resolves via wait or exactly one justified
  compaction, and waits never consume the 5-retry allowance (already spec'd).
- **Table change cost:** a new model = a table entry, zero harness-code diff (kimi's bar).

## 6. Beyond / correcting the audits

- **Codex audit §4 is wrong on a number:** `effective_context_window_percent` defaults to **95**, not 90
  (`openai_models.rs:393-395,470-471`); 90 % is only the `auto_compact_token_limit` derivation/clamp (`:461,530-540`) — the
  audit conflates two knobs.
- **Aider audit §8/§9 overstates "zero per-model if-trees inside the loop":** the loop is clean, but
  `apply_generic_model_settings` mutates the table through a ~160-line substring chain (`aider/models.py:437-598`); our
  resolution must allow *patterns in data* or the chain returns.
- **Codex audit §9.2/§12 misses that its own retry core already implements our owner rule:** the unbounded connection-retry
  path (`responses_retry.rs:22-23,90-118`, 5 s→60 s, feature-gated, internal sessions and Bedrock excluded) is a
  wall-clock-free wait inside a retry-budget system — evidence the two coexist; our memory/outage waits fit as
  capacity-classified decisions in the same mechanism.
- **Qwen audit §4/§8.1 understates tokenLimits.ts:** the clamp ladder with the #13252 small-window fix
  (`tokenLimits.ts:50-100`) is the complete version of the one-line "output clamp margin" lesson; take the whole function
  including the floor-cap.
- **None of the three model tables covers cache accounting** — aider has only `caches_by_default`/`cache_control` booleans,
  codex's table predates local prefix-cache economics, qwen's caching is call-site code, not table data. Per-provider cache
  *accounting* (explicit vs implicit-prefix, incl. mlx-serve `/props` + timings figures) is our extension, driven by the
  measured 411× warm/cold gap.
- **Aider audit §7 is right** that aider has no automatic edit-format fallback (reflection capped at 3,
  `base_coder.py:101,936-944`); its import-time litellm enumeration (`exceptions.py:66-76`) is why we keep a typed taxonomy
  instead of porting the pattern.

## 7. Sources and commit pins

- Audits head-pinned per `docs/harness-research/SOURCES.md`: aider `5dc9490bb35f`, qwen-code `6212189f1f83`, codex
  `6786dccbe726`, kimi-code `b510fa9aec85`, under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`
  (read-only, never re-cloned). - Repo files read this session: aider `models.py`, `exceptions.py`, `coders/base_coder.py`,
  `resources/model-settings.yml`; qwen-code `packages/core/src/core/openaiContentGenerator/provider/dashscope.ts`,
  `core/tokenLimits.ts`, `providers/presets/alibaba-standard.ts`, `core/openaiContentGenerator/prefix-caching.ts`; codex
  `codex-rs/protocol/src/openai_models.rs`, `protocol/src/error.rs`, `core/src/responses_retry.rs`,
  `model-provider-info/src/lib.rs`. - Specs: `docs/harness-engineering/spec/providers-engine.md`, `spec/agent-loop.md`,
  `spec/tools-editing.md`; gap check `packages/schema/src/model.ts`. Research:
  `docs/research/2026-09-15-harness-and-engine/` (measurements.md, mlx-serve.md, idle-eviction),
  `2026-10-07-engine-decode/README.md`, `2026-10-07-memory-ceiling/README.md`;
  `docs/harness-research/05-AGENT_ENGINEERING_FINDINGS.md` §3; `09-IMPLEMENTATION_ROADMAP.md` Phase 4;
  `docs/harness-engineering/PRIOR_WORK.md`.
