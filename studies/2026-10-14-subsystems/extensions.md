DRAFT — P2 study, written 2026-10-14, awaiting cross-check

# Extensions — subsystem study

Scope: plugin APIs and hosts, lifecycle hooks, skills/agents, MCP integration, command surface, config overlays,
extension trust. Strongest peers: **claude-code** (dogfooded hook kernel + marketplace), **pi** (event-rich bus,
trust-gated loading), **codex** (hooks crate + skills + marketplaces + config overlays), **opencode** (our
inherited API; typed failure policy, hot-swap host). Zero behaviour is normative in `spec/plugins.md`,
`spec/server-api.md`, `spec/tools-editing.md` (MCP), `spec/storage-history.md` (config) — cited `spec`; source
read only for gaps. (verified) = re-read at the `SOURCES.md` pins; the owner's earlier prototype not read.

## 1. How each implementation works

### Zero Code today (source, this repo)
- Hook contract `packages/plugin/src/index.ts:222-335`: ~21 points — `config`, `event`, tool defs, `auth`,
  `provider.models`, `chat.message/params/headers`, `permission.ask` (the only veto, :261),
  `command.execute.before`, `tool.execute.before/after` (:266-281), `shell.env`, transforms,
  `experimental.session.compacting`/`compaction.autocontinue` (:305-326), `tool.definition`; input carries a
  full SDK client to its own server (`zero/src/plugin/index.ts:164-169`).
- Host: bundle loads unless `ZERO_DISABLE_BUNDLE` (:97-104,188-201); external plugins load sequentially after
  install/entry/compat staging (`plugin/loader.ts:86-133`; npm-only semver gate :123-131; one file-plugin retry
  :208-230). `trigger()` awaits hooks sequentially in registration order (:315-328); bus events fan out
  fire-and-forget with per-hook isolation (:279-293). No startup replay, durable plugin state, or scheduler
  (01 D5:881-890) — goal rebuilds by transcript scanning (`spec/plugins.md:20`).
- Skills: `SKILL.md` scan of `~/.claude/skills`, `~/.agents/skills`, project-up `.claude`/`.agents` to the worktree,
  config dirs `{skill,skills}/**`, config `skills.paths` + `skills.urls` (HTTP `index.json` pull, version-staged atomic
  swap, `skill/discovery.ts:49-132`); duplicate name warns, last wins (`skill/index.ts:125-134`). Dual invocation:
  `<available_skills>` catalog + `skill` tool (permission ask, body + base-dir contract + rg-sampled files,
  `tool/skill.ts:12-60`) + same-named slash commands (`command/index.ts:134-152`). Agents: config +
  `{agent,agents}/**/*.md` (`config/agent.ts:13`) merged with built-in build/plan (`agent/agent.ts:100-340`). Commands:
  init/review + config + MCP prompts + skills (`command/index.ts:70-152`); plugins get commands only by mutating config
  via the `config` hook (goal's `/goal`, `spec/plugins.md:10`). MCP tools `sanitize(client)_sanitize(tool)`, kept off
  the declared prefix behind `tool_search`/`tool_exec` at ≥8 tools or ≥5 % of context (`spec/tools-editing.md:117-122`, measured).

### pi — 41 events, one in-process bus, trust-gated loading
- `ExtensionAPI.on()` exposes **41** subscribable events at the pin (`extensions/types.ts:1592-1663`; union
  :1405-1437, verified): session lifecycle incl. before-compaction/switch/fork/tree, `context`/`context_with_system`
  rewrite, `cache_warming_decision`, provider request/headers/response/stream taps, agent/turn/message/tool-execution
  lifecycle, model/thinking select, `tool_call`→`{block,reason}` veto, `tool_result` rewrite, `input`, `user_bash`. All
  interception routes through the loop's single before/afterToolCall install; extension failure *blocks* execution
  (`agent-session.ts:654-680,677-679`).
- Durable extension state exists: `appendEntry(customType, data)` writes non-LLM custom entries into the
  session JSONL — per-session, branch-scoped, replayed on resume (types.ts:1746-1748, verified). Loading:
  `pi.extensions` manifest, `index.ts`, or loose `.ts/.js` (`loader.ts:738-760`); a bootstrap pass keeps
  project-local extensions out until the project is trusted — trust gates loading, not tools
  (`resource-loader.ts:505-523`, verified).

### codex — Claude-compatible hooks as a crate, plus skills and marketplaces
- `hooks` crate: 12 named events (`lib.rs:23-36`, verified), matchers meaningful for 9 (:43-53); `hooks.json` loaded
  from the config folder (`discovery.rs:339-343`) converging with TOML hooks on one trust identity (:769);
  per-handler *persisted* enable keys `source:event:group:handler` (`lib.rs:112-123`); `bypass_hook_trust`
  (`registry.rs:46`) and required managed hooks that **refuse to start the session** (:70-86); sync vs async dispatch,
  results drained between turns (`dispatcher.rs:134-172`); failure is an enum `FailedContinue|FailedAbort`
  (`types.rs:15-30`, verified).
- Skills: `SKILL.md` frontmatter + `SkillPolicy{allow_implicit_invocation}`, `SkillDependencies{tools}`, UI
  `SkillInterface` (`model.rs:8-82`, verified); embedded system skills install behind a fingerprint marker
  (`lib.rs:59-80`); `$skill` mentions + implicit invocation per command; plugin install *migrates commands into skills*
  (`utils/plugins/src/lib.rs:45-46`) — skill-as-unit convergence.
- Plugins: manifest paths `skills[] + mcp_servers + apps + hooks` (`plugin/src/manifest.rs:19-24`); marketplaces are
  `.agents/plugins/marketplace.json` (`marketplace.rs:21-22`) with per-plugin install policy (:158);
  `.codex-plugin/plugin.json` *overlays* a plugin's MCP env/transport onto host config
  (`agent_plugin_mcp_overlay.rs:13-89`, verified); everything rides the `ConfigLayerStack` — plugins are config layers.

### claude-code — the hook kernel as product architecture (black-box + published types)
- `mods/types/claude-code.d.ts` (2.1.277): `EngineEventOf` = exactly **40** events (:3255-3685, verified) incl.
  `tool.call` (onion `next(e)`; returning while next is pending aborts beneath) and `tool.check` (decision chain without
  executing); **33** distinct `classic.<Event>` names (:958, verified). Verdict ceiling: "the managed-settings hooks run
  first: their deny is the call's result" (:3262) — user hooks tighten, never loosen (`sec-default` dogfoods it).
  Anthropic's own telemetry/diff/AGENTS-compat are plugins; ralph-wiggum's autonomy = refusing Stop
  (`plugins/ralph-wiggum/hooks/hooks.json`, verified). `claude plugin validate` statically reports a module's hooks.

### opencode — smallest typed hook core, scope-bound hot swap (our lineage)
- The API is domains/nouns, not a big enum: `Context` carries ~20 nouns incl. `storage` and skill/tool/reference
  *transform editors*; only 5 hookable domains (aisdk/session/permission/shell/tool, `core/plugin/hooks.ts:16-22`).
  Failure policy is in the types: **only `tool.execute.before` may fail**, the rest are `NoFailures` (:24-31, verified).
  Registration is Scope-bound — closing a plugin's scope unregisters its callbacks (:68-84) — powering generation-based
  hot reload with per-plugin `failed` state (`supervisor.ts:19-40`, verified). Retry tuning is a hook
  (`effect/session.ts:127,150`); model-persona prompts are ordinary plugins matched by model id (`optimize.ts:17-59`);
  npm install pins revisions (`module.ts:98-125`); plugins get **per-plugin namespaced durable KV**
  (`host.ts:468 storage(kv, pluginID)`, get/set/remove/scan, verified).

## 2. What is genuinely best, and why

1. **Dogfooding (claude-code).** Telemetry, diff, security and AGENTS-compat ride the same bus as third-party plugins,
   so the API cannot rot dishonestly — exactly our position: the five bundle plugins (`spec/plugins.md`) are the API's
   real test, and D5's ceiling (no durable state/replay/scheduler) is the proof it lies today.
2. **Typed failure policy + scope-bound registration (opencode).** Which hooks may fail is a type (`hooks.ts:24-31`);
   Scope-bound callbacks make hot swap and isolation mechanical. Zero's `trigger()` has neither: a hook throw skips
   later hooks *and* fails the caller (`plugin/index.ts:322-326`, verified).
3. **Trust as persisted per-handler state (codex).** Durable enable keys, required hooks that refuse a session,
   marketplace install policies — more than pi's repo-trust or Zero's semver gate can give.
4. **Event breadth without RPC theatre (pi).** 41 honest in-process events, one veto, `appendEntry` durability:
   goal/guard are implementable as pi extensions without core changes; in Zero they are not (D5).
5. **Skills as the convergent unit (codex+claude).** Commands migrate into skills with policy/deps/attribution;
   marketplace + `plugin validate` set the cheapest affordable ecosystem floor.

## 3. What to combine

- **Hook surface: keep gates, make advisories safe, drop cache-hostile taps.** Keep, mapped to new-core boundaries
  (`studies/agent-kernel.md` §4): admission (prompt submit/rewrite — replaces `command.execute.before` as goal's command
  seam), provider-request-built (params/headers/persona), tool-call gate, tool settlement, permission decision
  (tighten-only for user tier, ceiling above), compaction pre/post/autocontinue, model/agent select, turn-settle,
  session start/end, retry decision. Trim: per-delta taps and `experimental.text.complete` stay off the durable bus
  (cache-hostile text mutation, TUI-only frequency); take opencode's providerID-filtered registration
  (`hooks.ts:36-40`) and codex's sync/async split for Stop-style long hooks instead.
- **Skills.** Keep Zero's discovery order (`.claude`/`.agents` global → project-up → config dirs → paths → urls,
  `skill/index.ts:183-227`) — `.claude/skills` compatibility already ships. Namespace plugin-provided skills to end
  silent duplicate overwrites (:125-134); add codex frontmatter policy/deps only with a consumer; keep dual invocation
  (catalog + tool + command) and progressive `references/` with the base-dir contract (`tool/skill.ts:52-58`). Don't
  copy opencode's durable activation event before an eval justifies it.
- **One deferral seam for all extension tools.** Plugin and MCP tools enter the same `tool_search` catalog under
  the measured trigger (≥8 tools or ≥5 % of window, `spec/tools-editing.md:117-121`); extensions never write the
  declared prefix directly (`studies/tools-editing.md` §3). MCP servers become a plugin source with codex overlay
  semantics (`agent_plugin_mcp_overlay.rs:13-89`): a plugin may add/amend a server; the layer stack decides.
- **Trust, layered.** (1) semver compat gate (`plugin/loader.ts:123-131`); (2) pi's trust-to-load bootstrap for
  *project-local* plugins (untrusted project → user/global only, `resource-loader.ts:505-523`); (3) codex's
  persisted per-handler enable + bundle hooks not silently disableable; (4) claude's verdict ceiling at the
  permission gate: bundled policy hooks run last, deny final; user plugins tighten only. No in-process sandbox
  (single-user local project, `PRIOR_WORK.md`); bundle-disable flags stay the escape hatch.

## 4. Proposed design for the new core

1. **One extension bus keyed to durable boundaries.** Events fire at the kernel's durable transitions (admission,
   promotion/boundary, provider-attempt start, tool gate, tool settlement, compaction pre/post, model/agent select,
   turn-settle, session start/end, retry decision, config-set overlay), so a hook observation is a replayable fact,
   not a live-only subscription over a durable table (D5:887-888). Gate hooks veto; advisory hooks get
   `FailedContinue` with per-hook isolation; registration is Scope-bound for hot reload.
2. **Durable extension substrate.** Per-session extension entries through the event store (pi's `appendEntry`)
   replayed on startup; per-plugin cross-session KV (opencode `storage(kv, pluginID)`); `schedule(when,payload)` on
   the autonomy study's durable wake (`studies/autonomy.md`). Goal's iteration count, guard's read-table, design's
   reviewed-flag become substrate, not closures.
3. **Versioned, checkable API.** Publish the hook/event `.d.ts` generated from schema (claude `/plugin-types`
   lesson); keep the semver gate; a manifest lists hook points so `zero plugin validate` reports statically and
   the host orders tiers; plugin config lands as a named layer (`zero.<name>.enabled:false` keeps working,
   `spec/plugins.md:3`).
4. **zero-plugins as reference ports — and the conformance suite.** Public API only: goal (durable goal row + schedule
   + boundary check replaces `session.idle` + transcript-scan restore), guard (durable read table, advisory notes),
   design (post-turn boundary + subtask part), telemetry (event-store subscriber; `telemetry.db` schema-compatible
   because zero-eval reads it, `spec/plugins.md:56-67`), engine (provider overlay: config sync, fail-fast, thinking-off,
   persona-by-model-id replacing duplicate per-model prompt forks, 01:777-780). Every core special case (env flag,
   if-id branch) a port needs is a logged design defect.

## 5. Measurable targets

- **Behaviour change budget across the API redesign = zero.** Every `spec/plugins.md` line ports as a reference-plugin
  integration test: `/goal` surface + `zero_goal.kind=start` parts (:10-13), verify-oracle and `GOAL COMPLETE`/open-todo
  rules (:14-19), restart restore from *substrate* (goal.test.ts equivalents, no transcript scanning), guard advisory
  strings (:30-43), design's once-per-turn UI-file review (blind 29 vs 22 must reproduce within noise, :54), telemetry
  ledger + `/api/summary` (:56-67), engine fail-fast + thinking-off (`spec/providers-engine.md`).
- **Core special cases: 0.** No plugin-specific branches in the kernel; flags remain exactly
  `ZERO_DISABLE_BUNDLE` + per-plugin `enabled`.
- **Durability beats D1's live-DB evidence:** goals lost to restart 3/10 starts → 0; post-compaction goal restatement
  0/ever → fires on the compaction boundary event; budgets count tokens/time, not only continues.
- **Cost guards:** `trigger()` overhead bundle-on vs `ZERO_DISABLE_BUNDLE=1` per provider turn (sequential
  await, `plugin/index.ts:322-326`); prefix-hit ratio stays flat with bundle on (no hook may move the frozen
  prefix); plugin load failure never blocks boot (`plugin/index.ts:220-263`).

## 6. Beyond / correcting the audits

- **"pi has a 36-event extension API" (prior framing) is stale: 41** subscribable at `3daa20280f6a`
  (`types.ts:1592-1663`; union :1405-1437, verified), incl. `cache_warming_decision` no audit documented — and the count
  is the wrong metric anyway; the lesson is one bus, one veto (`tool_call`), plus `appendEntry` durability.
- **claude-code audit §1 ("14 official plugins + `plugins/marketplace.json`") is wrong at the pin:** `plugins/`
  has 13 dirs, 12 with `.claude-plugin/plugin.json` (plugin-dev has none); the registry is the repo-root
  `.claude-plugin/marketplace.json` — 13 entries (name/description/source/category + owner, verified). §2.1's
  "34" classic events: 33 distinct `hook_event_name` literals; `EngineEventOf` = exactly 40 — half right.
- **codex audit §8 tags Hooks [D]; at this pin they are a first-class crate** (verified): 12 events
  (`hooks/src/lib.rs:23-36`), persisted enable keys (:112-123), sync/async dispatch (`dispatcher.rs:134-172`),
  `hooks.json` from the config folder (`discovery.rs:339-343`), required-hook session refusal
  (`registry.rs:70-86`) — "one `notify` hook, no per-event plumbing" understates it.
- **opencode audit §174-189 omits the `storage` domain:** opencode plugins *do* get durable state — per-plugin
  namespaced KV (`host.ts:468`, verified). D5's "no durable state" is a Zero gap, not a law of plugin APIs (pi:
  per-session entries; opencode: cross-session KV). The fix is substrate the rebuild already plans.
- **Zero's own gap no audit sees:** triggered hooks have no error isolation (`plugin/index.ts:322-326`) while
  the event fan-out does (:283-291, verified) — a throwing bundle hook aborts the tool call and skips later
  hooks; the codex enum / opencode typed-NoFailures fix must land before any third-party hook ships.

## 7. Sources and commit pins

- Read-only clones under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`, pins per
  `SOURCES.md`: opencode `3ec856b6c265` (2026-10-14); pi `3daa20280f6a`; codex `6786dccbe726`; claude-code
  `b0b7bed98203` (issues/docs/plugins + `.d.ts`, black-box dominated).
- Audits `02-COMPETITOR_AUDITS/{opencode,pi,codex,claude-code}.md`; `01-ZERO_CODE_BASELINE.md` Part D (D1-D6);
  `spec/{plugins,server-api,tools-editing,storage-history,providers-engine}.md`; studies `agent-kernel.md`
  (boundaries), `autonomy.md` (durable wake), `tools-editing.md` (deferral seam); `PRIOR_WORK.md`.
- Re-verified at pins this session: pi `extensions/types.ts:1405-1437,1592-1663,1746-1748`,
  `resource-loader.ts:505-523`, `extensions/loader.ts:738-760`; codex `hooks/src/{lib,types,registry}.rs`,
  `hooks/src/engine/{discovery,dispatcher}.rs`, `skills/src/{lib,model}.rs`, `plugin/src/manifest.rs`,
  `core-plugins/src/{marketplace,agent_plugin_mcp_overlay}.rs`, `utils/plugins/src/lib.rs:45-46`; opencode
  `core/src/plugin/{hooks,host,module,supervisor}.ts`, `plugin/src/effect/{storage,session}.ts`; claude-code
  `mods/types/claude-code.d.ts`, `.claude-plugin/marketplace.json`, `plugins/ralph-wiggum/`; Zero
  `packages/plugin/src/index.ts:222-335`, `zero/src/{plugin/index,plugin/loader,command/index,tool/skill,agent/agent,config/agent}.ts`, `zero/src/skill/*`.
