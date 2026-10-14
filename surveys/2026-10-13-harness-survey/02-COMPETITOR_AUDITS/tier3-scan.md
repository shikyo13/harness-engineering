# Tier 3 scan — oh-my-openagent · oh-my-pi · Superpowers · SWE-agent

Audited 2026-10-13. Four shallow clones under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/`.
All `file:line` refs are relative to each repo root and were read from real source/docs at the pinned heads:
oh-my-openagent `5275771d8b38` (2026-10-14), oh-my-pi `bcd11cb1ccb` (2026-10-13), superpowers `bb92a7774141`
(v7.0.0, 2026-10-09), swe-agent `3ea751c087f3` (2026-10 repo, last commit 2026-07-16). Judged against the Zero Code
baseline (`01-ZERO_CODE_BASELINE.md`): what already exists here, what is genuinely new, what ports.

Positioning correction the task brief needs: **oh-my-openagent is no longer only an OpenCode extension.** At this
head it is "OmO", a standalone `omo` binary running on "senpi" — the team's own fork of pi (README.md:37-42) — with
the legacy OpenCode plugin shipped in parallel (`packages/omo-opencode`, plus `packages/omo-codex`). The OpenCode
edition is the best available map of OpenCode's extension surface, and its migration is itself a finding: the
plugin surface (below) was rich but bounded, and the team eventually forked the harness instead.

---

## 1. oh-my-openagent (OmO) — the reliability-hook maximalist

### Architecture and entry points

- Monorepo: ~60 TS packages + Rust `crates/` (desktop/computer-use backends per OS). The OpenCode edition entry is
  `packages/omo-opencode/src/index.ts` → plugin factory composing `create-tools.ts`, `create-hooks.ts`,
  `create-managers.ts`, `create-runtime-tmux-config.ts`; edition-specific hosts are `packages/omo-senpi`,
  `packages/omo-codex`.
- OpenCode APIs consumed (grep over `src/plugin/`): `chat.message` (29 sites), `tool.execute.before` /
  `tool.execute.after` (22/27), `session.idle` (11), `chat.params`, `experimental.chat.messages.transform` (3),
  `session.status/error/deleted`, `message.updated/part.*`. Plus tool registration (`ToolDefinition`), custom
  agents, slash commands, and plugin-injected MCP servers (docs/reference/features.md:985-1037).
  That is the whole extension surface OpenCode exposes — enough to bolt on ~58 composed hook slots but not enough
  to change the loop itself, which is precisely the constraint OmO keeps routing around.

### Hook machinery — the most complete "harness sidecar" catalog anywhere

`docs/reference/features.md:839-985`: 58 composed hook slots (23 session, 18 tool-guard, 8 transform, 7
continuation, 2 skill). Genuinely valuable entries, not marketing:

- **Continuation enforcement family.** `goal` re-injects a goal continuation prompt on `session.idle` while a goal
  is active (features.md:882, :363); `todo-continuation-enforcer` "yanks idle agents back to work" from incomplete todos
  (features.md:936); predecessor `ralph-loop` iterated until `<promise>DONE</promise>` with state in
  `.omo/ralph-loop.local.md` and crash recovery (hooks/ralph-loop/AGENTS.md:11-33; now deprecated in favor of
  `goal/`). The engineering: done-ness is a *durable predicate checked at idle*, not a model claim. Zero Code V1
  has none of this; V2's "drain has no transcript boundary" makes it the natural place for it.
- **preemptive-compaction** (`src/hooks/preemptive-compaction.ts`, 197-line core + degradation monitor + per-provider
  tests): compacts *before* the token wall, with a monitor that detects when compaction itself starts degrading.
  Compare Zero Code's reactive compaction; the degradation-monitor idea is the transferable part.
- **comment-checker** (features.md:892): PostToolUse blocks AI-slop comment patterns mechanically, with
  `// @allow` escape hatches. Cheap, enforceable-taste — exactly the class of rule that should not be a prompt.
- **runtime-fallback / model-fallback** (features.md:903): fallback chains triggered by retryable API errors and
  provider retry signals, applied on the *next* `chat.message`. Zero Code V2 lacks retry/fallback entirely
  (baseline finding: runner TODO list).
- **tool-output-truncator** dynamically sizes truncation per tool from the context window (features.md:911).

### Mass-ulw: DAG orchestration with a durable event ledger

`docs/reference/mass-ulw-protocol.md:1-80` + `packages/senpi-task/src/dag/types.ts`. A "mass ulw" prompt becomes a
graph of per-node agents; the run is a **WAL-journaled event stream** (seq-numbered `omo.dag.event`, 17 journaled
payload types) separated from best-effort channels (heartbeat, activity, wholesale snapshot). Node states
`pending|blocked|scheduled|running|completed|failed|cancelled|skipped`; waves; per-node model routing;
`dag.node.reused` carries a node's completed task across runs by `definitionFingerprint`; `dag.definition.amended`
tracks fingerprint changes with `invalidatedNodeIds`; steering delivery on a node is typed
`"steer" | "revive"` (`dag.node.steered`). This is the most rigorous durable-DAG wire contract found in any Tier-3
project, and its vocabulary is nearly isomorphic with Zero Code V2's steer/queue delivery + durable admission design.
Transferable directly: the lane split (sequenced ledger vs unsequenced telemetry), fingerprint-based node reuse for
re-runs, and `journal_corrupt` as an explicit diagnostic event type.

### Kibitzer memory + boulder continuation state

- Kibitzer (README.md:76): memory is a **git repo of markdown** (`packages/memory-core/src`: `facts/ journal/
  recall/ reflection/ receipts/ reminders/`), and a *small cheap model* reads it and nudges the main agent "before
  it repeats a mistake". Two separable ideas: (a) receipts/reflection as durable write-path, (b) a cheap-model
  watchdog on the main transcript (see also oh-my-pi's advisor — same pattern, different mechanism).
- Boulder (`packages/boulder-state/src`): plan/checklist continuation state with stale-work detection so
  `/ulw-execute` resumes a crashed plan in the same session (manifesto.md:128). Zero Code has shadow-git
  checkpoints but no durable "what was checked" ledger; this is the missing counterpart.
- Monitor tools (README.md:82 "waiting on a build or deploy is a subscription, not polling";
  `src/tools/monitor/`, `src/features/monitor/` with batcher + ring buffer + watchdog kill): agent-visible
  long-lived watchers that push. Zero Code's background shell jobs already push completion notices; the delta is
  *filter/batcher* (subscribe to pattern X in a stream, delivered in batches) as an agent tool.
- Team mode (docs/guide/team-mode.md:1-40): 12 `team_*` tools modeled on Claude Code Agent Teams, OFF by default;
  tmux-visualization. Late, guarded, unglamorous — evidence the team-parallel idea needs a task-list protocol more
  than more agents.
- Categories (`features.md:151+`): task-category → model-chain routing with per-model tuned prompts; cheap
  "quick" rungs. Zero Code's model resolution has no category tiering.

### What OpenCode's surface power looks like from here

A plugin that gets `chat.message`, `tool.execute.before/after`, `session.idle`, `chat.params`, and message-transform
can build: continuation loops, compaction policy, model fallback, output truncation, injection, per-tool guards, and
a task system — i.e. 80% of a harness, all without owning the loop. What it cannot build: durable admission,
delivery-mode vocabulary, cache-economics control, or provider replay. OmO's own trajectory (plugin → senpi fork)
is the empirical answer to "is OpenCode's extension surface enough?": for a maximalist, no. For Zero Code the mirror
question is whether V2's event/flag surface is enough for our own plugin story.

---

## 2. oh-my-pi (omp) — the deepest extension contract and the weirdest compaction

oh-my-pi (Stencil Labs' fork of pi-mono, ~80k lines Rust core) is now a full agent, not a bolt-on, but its
extension docs define pi's extension surface better than pi's own docs (docs/extensions.md; `src/extensibility/`).

### Doom-loop detection: origin claim verified

`packages/ai/src/utils/tool-call-loop-guard.ts:67-68` — `ToolCallLoopGuard`: canonicalizes tool-call arguments
(sorted keys, drops the `INTENT_FIELD` so restated intent doesn't defeat matching), detects consecutive identical
assistant tool calls **across turns** (threshold + exempt-tool set), and carries a result/argument summary for the
correction. The corrective is a rendered template — `packages/coding-agent/src/session/tool-call-loop-redirect.ts:21`
— injected as a structured `tool-call-loop-redirect` record, and explicitly shared by "the primary session's
LoopGuards and the advisor's own loop guard" so both speak one wording (:13-18). Upstream `badlogic/pi-mono`
packages contain no loop guard (grep clean), so the mechanism with the *shared corrective + per-agent guards* shape
is genuinely oh-my-pi's. Zero Code V1 already has doom-loop redirection; the deltas worth taking: exempt lists,
intent-field canonicalization, and running the same guard over subagents/advisors.

### Extension surface (what pi lets a plugin touch — richer than OpenCode's)

docs/extensions.md:451-560: cancelable `session_before_{switch,branch,compact,tree}` pre-events (a pre-event can
return a *replacement compaction result*); `context` message replacement that chains without rewriting persisted
history; `assistant_message` awaited text replacement with `textSignature` invalidation on edit (:472+);
`tool_call` pre-exec **input revision** (revalidated, seen by concurrency scheduling and the approval gate) and
passive `additionalContext` delivered after tool results in call order; `tool_result` middleware with failure-aware
context; and **`cache_warming_decision`** (:488): before each prompt-cache warming refresh the host exposes
`warmCost`, `missCost`, `continuationProbability` and a policy extension answers `warm|stop` (2s deadline, main
loop only, replay must stay byte-identical). That last one is the first *cache-economics control hook* seen in any
audit — Zero Code persists frozen prefixes for cache stability but makes the warm/skip decision implicitly. Also:
`session_stop` bounded-advisory continuation (8 max, blocking refusals don't consume the allowance), file-write /
file-delete fallbacks a tool can register, `ctx.runEphemeralTurn`, `ctx.setInterval`, `ctx.invokeTool` delegation to
builtins, custom renderers, RPC/ACP/extension-answerable plan review.

### Compaction: three ideas above the state of the art

- **snapcompact** (`packages/snapcompact/README.md:3-5`): instead of an LLM summary, discarded history is
  serialized and rasterized into dense PNG frames of pixel-font glyphs that vision models read back directly —
  local, deterministic, no summarizer call. Frame shapes are **per-provider tuned by SQuAD recall evals against
  real provider billing** (Anthropic 11on16, Gemini 8on22@2048 because image cost is flat, detail:"original" for
  OpenAI), token-priced by model lineage, with a Silver TTF fallback for CJK. Genuinely novel; worth a spike with
  our vision-capable local models where decode cost is free and summarization quality is the binding constraint.
- First-class compaction entries (`docs/compaction.md:24-55`): `CompactionEntry` with `firstKeptEntryId`,
  `tokensAfter`, and `providerReplayThroughEntryId` (ties the summary to the last entry covered by a native replay
  snapshot); `BranchSummaryEntry` captures *abandoned* branch context during `/tree` navigation, so exploration
  branches leave useful sediment. Zero Code has neither branch summaries nor replay-coupled compaction.
- `shake.ts` "mechanical content elision" and provider-native streaming compaction as separate strategies; the
  maintenance orchestrator picks per session.

### Task system and advisor

- `task` tool (docs/tools/task.md:31-60): batch spawn with **required shared `context`** rendered into every child
  system prompt; per-item `agent`, `outputSchema` + `schemaMode` (permissive accepts invalid payloads with warning
  after schema-retry exhaustion, strict fails); per-spawn `effort: lo|med|hi` mapped onto the resolved model's real
  thinking range; `tools` parameter that runs parent-kernel tools inside the parent's eval kernel for the child;
  `isolated` workspaces (Rust `pi-iso` PAL) whose patches merge back, retained across idle/parked transitions and
  revivable. **`solutionSpace`** (:45) is the standout: a required field describing *how open-ended the problem is*
  ("one fix: rename, names given" vs "deadlock cause open"), fed as the sole input to an `auto` thinking classifier
  — a first-class, auditable "thinking budget" argument instead of vibes. Async guidance is conditionally templated
  into the tool description itself (head commit PR #15477; `src/prompts/tools/task.md:1-8`), so sync deployments
  don't pay the tokens.
- Subagent lifecycle registry `running|idle|parked|aborted` with idle-TTL parking + revival
  (src/registry/agent-lifecycle.ts via docs/tools/task.md:22-23); outputs addressable as `agent://<id>` and
  transcripts as `history://<id>` internal URLs — reference-by-URL instead of context injection for big subagent
  results. UI labels auto-generated by a tiny model fire-and-forget.
- **Advisor** (docs/advisor-watchdog.md:1-20): optional reviewer models watch the primary transcript, run read/grep
  (grantable to edit/bash) in an isolated `ToolSession`, and inject concise advice; emission-guard dedupes advice
  by message fingerprint. Same family as OmO's Kibitzer; oh-my-pi's version is cleaner (roster config, own loop
  guard, never mutates primary state).

### Benchmark control plane: metaharness

`packages/metaharness/README.md:1-70`: one manager for Harbor/edit/snapcompact benchmarks behind one
experiment→run→trace model, SQLite store, REST/SSE + dashboard. Transferable ops ideas: bind-mount the repo
read-only into trial containers ("TS edits apply to the next trial with no rebuild", zero outbound network per
trial), auth never enters containers (generated `models.yml` points baseUrls at a host auth-gateway), and in-place
`resume` that reuses completed trials *including their spend*. This is the shape Zero Code's `zero-eval` grows into.

---

## 3. Superpowers — methodology packaged as skills

### What it actually is

Skills (SKILL.md + frontmatter) + one SessionStart hook + per-harness plugin manifests. README.md:66-76 states the
pipeline: brainstorm a spec → show it in digestible chunks → write an implementation plan "clear enough for an
enthusiastic junior engineer with poor taste, no judgement, no project context, and an aversion to testing" →
subagent-driven execution with per-task review. 15 skills, ~3.8k lines total (`wc -l skills/*/SKILL.md`).

### The engineering inside the prompts (this is not just prose)

- **Iron Laws** as hard gates: "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST… Delete means delete"
  (skills/test-driven-development/SKILL.md:31-38); "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST"
  (systematic-debugging/SKILL.md:14-16); "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE — if you haven't
  run the verification command in this message, you cannot claim it passes"
  (verification-before-completion/SKILL.md:14-17). Paired with "Red Flags" tables that pre-rebut the model's exact
  rationalizations — an empirically accumulated adversarial artifact, not decoration.
- **Process encoded as graphviz `dot` graphs inside the markdown** (subagent-driven-development/SKILL.md:61+): the
  per-task loop is a state machine — dispatch implementer → dispatch reviewer → fix round R of 5, **R≤3 resumes the
  same implementer, R≥4 gets a fresh implementer on a more capable model** → scoped re-review → at R=5 adjudicate
  findings, park non-load-bearing ones in a durable ledger. Bounded escalation with model-tier promotion on stall is
  a portable orchestration policy. The dot graphs are machine-rendered and tested (tests/writing-skills/test-render-graphs.sh).
- **writing-skills meta-method = TDD for process docs** (skills/writing-skills/SKILL.md:10-50): write a pressure
  scenario run with a subagent, watch the baseline fail (RED, recording the exact rationalizations), write the
  minimal skill (GREEN), close loophole-rationalizations (REFACTOR). "If you didn't watch an agent fail without the
  skill, you don't know if the skill teaches the right thing." Also: don't write a skill for anything regex/validation can
  enforce mechanically. This is the most directly transferable single artifact in the whole tier.
- **Bootstrap discipline**: using-superpowers/SKILL.md (65 lines) is injected in full every session; "1% chance a
  skill applies ⇒ must invoke, IF A SKILL APPLIES YOU DO NOT HAVE A CHOICE"; process skills outrank implementation
  skills; subagents dispatched for a task are exempted (`<SUBAGENT-STOP>`).
- **Portability mechanics**: one SessionStart bash hook (hooks/session-start) that survives a broken PATH
  (anthropics/claude-code#43127 — uses `${0%[/\\]*}` instead of `dirname`, bash `$(< file)` instead of `cat`) and
  emits the *platform-specific* context field (`additional_context` vs `hookSpecificOutput.additionalContext` vs
  top-level `additionalContext`) because some hosts read both without dedup. Per-harness manifests:
  `.claude-plugin/`, `.antigravity-plugin/plugin.json` (v7.0.0 with `suggestedPrompts`, :3-10), and the Antigravity
  marketplace manifest `.agents/plugins/marketplace.json` (source url + `policy.installation: AVAILABLE`,
  `authentication: ON_INSTALL`). Per-platform tool-mapping reference files (`references/{claude,codex,pi,antigravity}-tools.md`)
  solve tool-name divergence in prose. tests/ exists per harness (tests/{codex,pi,antigravity,opencode,devin,kimi,…})
  plus tests/claude-code/analyze-token-usage.py: per-agent token breakdown (main vs each subagent, cache read/creation)
  — skill systems need cost measurement or they sprawl.

### Judgment: capability or packaging?

~85% prompt-packaging — no loop changes, no new tools, one hook. But it is packaging *of a real process spec with
built-in adversarial tests*, which is more than most prompt packs, and the packaging layer itself (dot-graph process
specs, rationalization tables, pressure-scenario skill TDD, per-platform reference files) is engineering.
What survives porting to Zero Code's existing skill system: (1) writing-skills pressure-testing protocol — adopt as
our skill authoring standard and wire into zero-eval; (2) Iron-Law + rationalization-table format for our highest-risk
processes (edit-before-read, completion claims); (3) the SDD fix-loop policy (resume ≤3, fresh+stronger model ≥4,
ledger-parked findings); (4) per-platform tool-mapping reference files inside skills that name tools. What does NOT
survive: the mandatory-invocation bootstrapping doctrine ("cannot rationalize your way out") — it fights Zero Code's
registry/visibility machinery, and on weak models it taxes every turn for methodology the model may not need.

---

## 4. SWE-agent — the research ancestor, and its own retirement memo

### ACI design lessons (paper findings, still in the code)

`docs/background/aci.md:11-15` (condensed from arXiv 2405.15793): lint gate that *rejects* syntactically invalid
edits; purpose-built 100-line windowed file viewer with scroll/search instead of `cat`; search output deliberately
terse ("list each file that had at least one match — showing more context confused the model"); explicit
"command ran successfully and produced no output" on empty results. All four remain true in 2026 and all four are
already mirrored in Zero Code's truncation/edit-guard design — the confirmation matters more than the novelty:
our 9-stage matcher cascade and spill-file hints are this lineage's descendants.

### Mechanisms worth keeping from the scaffold

- **Tool bundles as config + executables** (docs/config/tools.md:10-45): `tools/<name>/{bin/,config.yaml,install.sh}`;
  the config's `signature`/`docstring`/`arguments` render into the system prompt via `{{command_docs}}`. Tools are
  plain CLI commands on PATH inside the sandbox — the *interface* is the docstring. Config-driven agents
  (config/sweagent_0_7/*.yaml): system/instance/next templates + action parser (thought-action, XML, function-calling)
  — agent definition as data, multiple interaction formats over one scaffold. This is the direct ancestor of every
  config-driven agent definition since.
- **Stateless commands + an externalized `state_command`** (docs/config/tools.md:47-75): after *every* action the
  harness runs `_state`, which prints JSON (`open_file`, `working_dir`) from a shared `registry` (tools/registry/),
  and that JSON formats the next prompt — persistent UI state lives in the environment, not the model, and the
  prompt reflects it each turn. Cheap, debuggable, and still underrated: Zero Code's "stable-index pressure notes"
  are the same instinct, narrower.
- **History processors as a composable pipeline** (sweagent/agent/history_processors.py): `LastNObservations`
  (the classic), `TagToolCallObservations`, `ClosedWindowHistoryProcessor`, and Anthropic `cache_control` markers
  set/cleared inside the processor (:46-53) — cache breakpoints as a history-processing concern.
- **Reviewer-gated submit** (tools/review_on_submit_m/README.md + sweagent/agent/reviewer.py): `submit` doesn't
  submit; it triggers a review pass (preselector→chooser structured outputs, `AbstractRetryLoop`); only the
  model-hidden `submit -f` exits. An early, explicit instance of "independent verifier before completion" — same
  principle as Superpowers' verification gate and OmO's `<promise>` oracle checks.
- **Trajectory tooling**: web inspector over saved trajectories (sweagent/inspector/server.py), resolved-set
  differencing across runs (sweagent/run/compare_runs.py), `quick_stats.py`, `merge_predictions.py`, and
  `run_traj_to_demo.py` — mine successful trajectories into few-shot demonstrations. The score-diff + cost
  breakdown + demo-mining trio is exactly the loop Zero Code's zero-eval should close (baseline finding 7: we have
  primitives, nothing consumes them).

### The "mini" split — the relevant 2026 research artifact

docs/index.md:16-20 and README.md:20-24: the project itself now recommends **mini-swe-agent** — ~100 lines of
Python, 65% on SWE-bench Verified — over SWE-agent's elaborate ACI. The split's lesson is the finding: *ACI
elaborateness was compensation for weak models; as models improved, most bespoke tools stopped paying rent, and
what survived is (a) the handful of interface lessons above, (b) the eval/trajectory apparatus.* Harness design
should keep the loop dumb and spend complexity where the model demonstrably fails — which in 2026 means context
management, verification, and cost control, not custom file viewers.

---

## 5. Other genuinely innovative harnesses discovered (watch-list only — existence verified via `git ls-remote`, not audited)

1. **lazycodex** (code-yeongyu/lazycodex) — OmO's orchestration layer retargeted onto the Codex CLI (cited in OmO's
   migration docs; OmO ships a "LegacyCodex" importer). Same author's proof that the ultrawork/DAG layer can ride a
   closed-loop vendor harness; watch for what the Codex surface forbids and how he routes around it.
2. **superpowers-evals** (prime-radiant-inc/superpowers-evals) — the evaluation repo behind Superpowers' skills
   (referenced from superpowers' org). Should show how skill pressure scenarios are run/regressed; directly on our
   path for wiring writing-skills-style tests into zero-eval.
3. **openclaw** (openclaw/openclaw) — the personal-agent harness OmO's "Jobdori" maintainer runs on; OmO ships
   `packages/openclaw-core` ("harness-neutral gateway, reply-listener daemon, session registry, tmux injection").
   Unusual execution strategy: a persistent, channel-connected agent outside any coding session; the harness-adjacent
   daemon/registry patterns are worth stealing independent of the project's hype.
4. **brush** (reubeno/brush) — a Rust pure-Rust reimplementation of bash used by oh-my-pi's tooling: an embeddable
   PTY/shell emulator. For a harness this means interactive terminal tools with deterministic, inspectable state and
   no real shell dependency — the most underrated infrastructure item in the tier (relevant to Zero Code's
   background-shell and tree-sitter bash-permission machinery).

Also seen, unverified/deprioritized: omo-herdr-dag (jc01rho — live DAG side-pane viewer over OmO's `omo.dag.*`
channels; cheap visualization of an already-audited ledger), Harbor (the container-native benchmark runner behind
metaharness — verify org before adopting).
