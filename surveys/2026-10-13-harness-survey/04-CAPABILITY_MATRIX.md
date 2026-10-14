# 04 — Capability matrix

Built 2026-10-13 from the head-pinned audits in `02-COMPETITOR_AUDITS/` and `01-ZERO_CODE_BASELINE.md`. Every
rating is anchored to cited source or probe evidence, not reputation. Scale per capability: **A** best-known
implementation, **B** strong, **C** adequate/partial, **D** weak or absent, **?** evidence too thin to rank.
A superscript `s` = flag-off/unshipped at audited head (don't credit shipping quality), `c` = closed-source,
inferred/observed only. Uncertainty is called out per row where it matters.

Participating heads: zero-code `123cb9262f` (branch), opencode `3ec856b6`, aider `5dc9490b`, pi `3daa2028`,
codex `6786dccb`, claude-code `b0b7bed9`+cli 2.1.296, qwen-code `6212189f`, deepseek-harness `d7432673`,
antigravity `9eebbedf`+binary 1.3.4, gemini-cli `42c51822`, openhands-sdk `a430c778`, goose `e9e0ab62`,
mini-swe `04d809ce`, cline `26cea2fa`, roo-code `b867ec91`, kilo-code `4ee8d1b3`, crush `d6edc322`, kimi-code
`b510fa9a`. Extensions (omO/oh-my-pi/superpowers) noted inline, not as columns.

## A. Agent-loop architecture

Capability splits into sub-dimensions because no harness is uniformly best.

| Sub-dimension | Leader | Runner-up | Ratings (others, best-of-column trimmed) |
|---|---|---|---|
| Loop clarity / testability | **Codex** A: typed Op/EventMsg protocol, per-step `StepContext`, one loop shape | **pi** A: pure `runLoop` over message arrays with policy hooks | Zero C^s (V2 clean, V1 monolith ships), OpenCode B, kimi A (XState machines), gemini B (layered), Cline C (2,910-line god object), goose B (21-op state machine), OpenHands C (duplicated run/arun loops) |
| Continuation decision quality | **OpenCode** A: tagged outcome enum (retry/synthetic-continue/output-cap/overflow/max-steps-force-summary) | **Zero V1** B+ (positive-signal exit in V2; V1's triple-condition heuristic is fragile) | Antigravity B^c, codex B, qwen B− (LLM next-speaker call, inherited, off-the-cost in Gemini: same trick was defaulted **off** upstream), aider B (reflection loop capped 3) |
| Mid-turn steering | **Antigravity** A^c (durable queue + send-immediately + /btw + remote interrupt) ≈ **Zero V2** A^s (durable steer/queue, cutoff promotion, allowance reset) | **OpenCode** A (same design, live), **Codex** B+ (mid-turn drain at safe boundaries), **goose** B+ (ops_steer between turns), kimi B (journaled steer/queue) | V1 Zero C (implicit history reload), gemini C (none mid-turn), pi B (poll-based steering queue, clean but lossy) |
| Cancellation/interruption settlement | **Zero V2** A^s (`uninterruptibleMask` + fail-unsettled + next-drain repair), **OpenCode** A (write-ahead claims + restart settlement) | Codex B+ | kimi B (machine refold), claude B^c, antigravity B^c (auto-resume active turns) |
| Stall/loop detection | **Gemini CLI** A (heuristic + LLM double-check + recovery turn, source-verified) | **oh-my-pi** A (origin of doom-loop; argument canonicalization, cross-turn) | Zero B (doom-loop in V1 only; V2 none), kimi B+ (truncation digest), aider C (reflection cap only), mini-swe D, opencode C |
| Retry discipline | **Zero V1** A (side-effect-aware canRetry, header-honoring backoff, memory-refusal ladder) | Gemini B+ (dual budgets, mid-stream) | Codex B (policy + guardian), deepseek B (caps not retries), aider C, pi D (none) |

**Strongest known implementation overall: Codex for the shipped loop skeleton; Zero/OpenCode V2-generation design
for the durable-admission variant (Zero's is flag-off, OpenCode's ships).** Uncertainty: Antigravity's BattleMode
ranked B by changelog consistency only — internals unavailable.

## B. Tool interfaces and execution quality

| Sub-dimension | Leader | Notes |
|---|---|---|
| Edit reliability | **Zero** A: 9-stage exact-first cascade, uniqueness gate, disproportionate-span refusal, near-miss excerpts, auto LSP diagnostics (edit.ts:695-749) | Codex A (apply_patch fuzz ladder + `seek_sequence` relaxation), crush A− (enforced read-before-edit via read tracker + `lsp_replace_symbol`), pi B+ (exact→normalized fuzzy, unchanged-lines byte-preserving, no similarity tier), claude B+^c (exact old/new + replace_all), gemini B (4-tier + telemetry), aider B (per-model format cascade; whole-file fallback risk), goose D (exact-string only) |
| Fuzzy-tier corruption risk | lowest: **Codex** (bounded fuzz) / **crush** (enforced freshness) | highest: aider whole-file, Zero/gemini similarity tiers without *enforced* read freshness (Zero's guard plugin keeps a read-table and appends an advisory `[zero-guard]` nudge — measured, non-blocking; 01 D2) |
| Tool-output shaping | **Zero** A (single truncate service, spill files, capability-aware next-action hints, empty≠error everywhere) | Gemini A− (masking-to-disk with protection window — different, complementary), Codex B+, opencode B, pi B (bash-centric, head/tail), mini-swe B (head5k/tail5k honest elision) |
| Shell/terminal | **Codex A** (OS sandbox + unified exec + execpolicy DSL + classified denials) | Zero A− (no persistent shell but jobs with incremental reads + push notices + tree-sitter permission parse), goose A− (persistent PTY sessions), gemini B+ (node-pty + background tools), claude B+^c (run_in_background + sandbox flag), aider C, mini-swe C (sandbox-only) |
| Sandbox enforcement | **Codex A** (seatbelt/landlock/bwrap + protected subpaths + network allowlists) | Gemini A− (deny-default seatbelt now, source-verified), Kilo A− (seatbelt/bubblewrap + TLS-SNI relay), Antigravity B^c (per-OS sandboxes + blocked-request recording), Claude B^c (sandbox flag), everyone else C/D (prompt-level caution) |
| Minimal toolset discipline | **pi A** (4 tools + prompt-adaptive bash), **mini-SWE A** (1 tool), **Codex B+** (tiny core + BM25 tool_search) | Zero C+ (≈20 builtins, ~6k-token declarations, four reach-layers), claude B (28 builtins but ToolSearch deferral), kimi B (select_tools), aider C (per-model format sprawl instead) |
| Code-execution composition | **DeepSeek Harness A** (PTC presentation mode, sandboxed per-run processes, full-pipeline sub-dispatch, durable dispatch events; effectiveness self-admittedly unmeasured) | opencode A− (CodeMode execute over own tools, live in dev), goose B+ (`code_execution` platform ext), Kilo/OmO B (task guidance), Zero C^s (codemode MCP-only, uncapped wiring) |
| Repo/structure intelligence via tools | **crush A** (8 default-on LSP tools incl. rename + symbol-range replace), **Zero B+** (strongest *automatic* diagnostics but LSP tool flag-off; repomap flag-off), aider B (repomap on), Claude B^c (`@conversation:`-style search absent but Explore agents), kimi D (no LSP at all) |
| MCP | Gemini A (tools+prompts+resources, OAuth), Zero A− (same breadth + crowded-catalog deferral), Claude A^c (elicitation, marketplace), antigravity B^c (meta-tool dispatcher), goose B (extensions-as-subprocess) |

## C. Repository intelligence & context engineering

| Sub-dimension | Leader | Notes |
|---|---|---|
| Repo map / ranked context | **aider A** (personalized PageRank with documented weight constants, binary-search token fitting, cache-ordered insertion — the reference implementation) | Zero B+ (aider-style PageRank port exists but flag-off; seed-boost variant), crush B (LSP-driven), omO B (ledger), Claude/Codex B^c (Explore/grep-driven) |
| Compaction design | **Gemini CLI A** (measured token ground truth + processor pipeline + masking) ≈ **Zero V1 A** (cache-continuation summaries, verbatim tail sizing with measured constants, retained user messages, factual recovery checkpoint) | OpenCode A− (incremental update-don't-rewrite checkpoints + overflow ladder), Cline A− (file-ops-preserving), kimi B (85%+50k reserve handoff), goose B (escalating ladder), aider C (summarize-only), pi B (reserve/keep with incremental merge + abandoned-branch summaries), deepseek D (none) |
| Prompt-cache engineering | **Claude Code A^c** (1h ephemeral writes, snapshot/hoist defenses, measured economics) ≈ **Zero A** (frozen+persisted prefix, stable-index notes, continuation summaries, affinity headers — with dated cost comments) | aider B+ (cache-ordered map pair + warming), pi B+ (proactive warmer with explicit $ math), codex B+ (comp_hash-forced compaction), antigravity B^c (cache_read_tokens in usage), kimi B (DashScope breakpoints) |
| Context-loss safety | **Zero B+** (verbatim tail + retained user msgs + archive — but archive path only surfaced on overflow) | OpenHands A− (condensation as replayable event, hard/soft severity), omO B (preemptive + degradation monitor), oh-my-pi B (snapcompact rasterization — exotic, recall-tuned), most others C |
| Cross-session memory | **omO B+** (git-markdown memory + Kibitzer watchdog), Kilo B (capture/recall files), Zero B (budgeted `<project_memory>` ≤500 tok + lessons adoption), claude B^c (CLAUDE.md + auto-memory), aider C, most C |
| Multi-hour drift control | **omO A−** (done-ness predicate at session.idle + todo continuation), **Kilo B+** (durable goal + scheduled wakes), Antigravity B^c (/goal + schedule), Zero B (todo carry-over + memory handoff + executive-s-c) |

## D. Editing & SWE workflow

| Sub-dimension | Leader | Notes |
|---|---|---|
| Checkpoints/rollback | **Zero A** (shadow-git with alternates seeding, per-step hashes, per-file revert), Codex A (per-step snapshots + fork truncate-at-boundary), Antigravity A^c (SQLite step snapshots, sweep-forward recovery) | Cline B− (**regressed** to stash-only), gemini B+ (shadow git + $rewindTo), aider B (git-native undo), OpenHands B (event replay) |
| Self-correction after build errors | **Zero A−** (auto LSP diagnostics on every edit + formatter loop) | crush A (diagnostics tool + rename consistency), aider B+ (lint/test hooks), codex B^c (model-family dependent), mini-swe C |
| Multi-file consistency | **Codex A** (one patch call, many hunks), Zero B+ (apply_patch path exists but gpt-5-only gating), claude B^c (multi Edit calls), pi C+ (one edit call per pair) |
| Architect/editor split | **aider A** (architect picks, editor applies with empty history + map_tokens=0, cheap editor — the only harness with this *shipped and measured*) | DeepSeek B+ (PTC as alternative), Zero C (executive designed, unwired), Claude B^c (advisor model on transcript rows — observed), Antigravity B^c (reviewing agent, 1.3.5) |
| Completion determination | **omO A−** (durable done-ness predicate), mini-swe B (sentinel + reviewer-gated submit in SWE-agent), **Zero B− plugin-side** (goal plugin ships `--verify` exit-0 oracle + marker-vs-open-todos + continue drivers, default-on and live-verified — but in-memory, restart-fragile, count-budgeted; 01 D1; core V2 has no task judgment, executive unwired), most C/D (finish=stop means done) *(corrected 2026-10-14)* |

## E. Long-horizon autonomy

| Sub-dimension | Leader | Notes |
|---|---|---|
| Durable execution substrate | **OpenHands SDK A** (stateless step + event log + fork/branch + base_state status checkable cross-process) | Zero V2 A−^s (admission/epochs/coordinator; no status column, no startup recovery), Codex A− (rollouts + persist policy + meta freeze; resume not auto-restart), kimi A− (journal + machine refold on restore), Antigravity B+^c (auto-resume active turns), Cline B+ (durable sessions rebuilt in sdk), goose B, aider/mini-swe D |
| Scheduled/async wake | **goose A** (recipe × cron scheduler × agent-callable manage_schedule — full circle, source-verified) | Kilo A− (goal loop + schedule_wakeup/cron/process-wait parking), Antigravity B+^c (schedule tool + MaxIterations + remote-control daemon), omO B (ralph-loop), Zero C+ (background jobs; desktop recurrence scheduler starts saved prompts as new sessions with run history — no session-level park/wake, nothing wakes an existing session; 01 D6.1) |
| Recovery after crash/model failure | **Gemini CLI B+** (history self-repair + dual retry) among open loops; **OpenHands A−** (resume-by-bootstrap) for cross-process | Zero V1 B (parts replay) / V2 B^s (lazy failInterruptedTools, no eager restart by design), claude B^c |
| Progress evaluation & strategy change | weakest category overall: best is **omO/oh-my-pi B+** (solutionSpace openness field, Kibitzer), Codex Guardian B^c (risk triage), **Zero B− plugin-side** (goal verify + guard unfinished-reply continue + design review gate all fire at idle with measured blind gains — but non-durable and no strategy change; core executive advisory-logged) — no system has a durable measured strategy-change mechanism; this remains the open field (see 07) |
| Human steering without restart | Antigravity A^c / Zero V2 A^s / OpenCode A (see steering row); V1 C |
| False-completion detection | **SWE-agent/mini-swe lineage B+** (reviewer-gated submit, hidden tests in eval harness), Zero zero-eval B (recall probes + `verified_after_timeout` diagnostics) — *harness-side product detection exists only in Zero Code* (goal `--verify` + design gate, non-durable, 01 D1/D4) and is embryonal everywhere else; strongest implementations remain eval-side |

## F. Multi-agent orchestration

| Sub-dimension | Leader | Notes |
|---|---|---|
| Subagent ergonomics | **Antigravity A^c** (markdown agents, worktree branch/share modes, model tiers, roster auto-injection, @addressing, inbox, kill_all) | kimi A− (swarm tool: 128 subagents, template+resume map), Claude B+^c (isolation: worktree|remote, backgrounded, anti-injection envelope), Zero B (task tool, depth caps, deny-by-default nesting, background inject-as-steer flag-off), goose B (child sessions), OpenHands B (recursive LocalConversations as frontmatter+skills), aider D |
| Parallel-code safety (worktrees/claims) | **Codex A−** (RwLock parallel gate + sandbox per exec; multi-agent via external tools), Kilo B+ (worktree Agent Manager, wave orchestrator), Antigravity B+^c (worktree modes), Zero C (worktrees exist as feature but not per-subagent default), omO B (DAG ledger) |
| Access-conflict-aware tool scheduling | **kimi A** (declared file accesses → conflict-aware scheduler, source-verified) | Codex B (rw-lock classes), Zero C (none — tools race), most D |
| Cross-model delegation | aider A (architect/editor with measured economics) | Antigravity B^c (model tiers), Claude B^c (advisorModel rows observed), Zero C+ (model-per-agent config, executive routing unwired) |

## G. Model independence & model-specific optimization

| Sub-dimension | Leader | Notes |
|---|---|---|
| Per-model adaptation as data | **aider A** (YAML per-model edit format/dialect/reminder placement; 324+35+4 model rows) | **kimi A−** (trait-ized quirks: empty-content deletion, schema deref, thinking knobs, completion-budget = window−used), **qwen A−** (per-model capability presets, disableField arbitration "qwen3.8 rejects reasoning_effort with thinking_budget", preserve_thinking round-trip), Zero B+ (per-model prompt .txt forks + repair aliases + apply_patch gating — real but fork-code not data), opencode B (persona plugins), Gemini C+ (model availability service) |
| Local/weak-model support | **Zero A** (repair/coercion/doom-loop/truncate-hints/50KB caps tuned over measured local runs) | aider B (reflection), pi B (low format bar + exact edit), oh-my-pi B (solutionSpace budgets), mini-swe C (bare), everything frontier-first C |
| Token/window honesty | Gemini A (measured ground truth), kimi B+ (window-derived budgets), antigravity B^c (window-sized budgets), Zero B (declared-metadata + estimate fallback) |

## H. Extensibility & composition

| Sub-dimension | Leader | Notes |
|---|---|---|
| Extension surface power | **Claude Code A^c** (hooks, plugins, marketplace, skills, subagents, MCP, SDK mirror), **Antigravity A−^c** (plugins bundle everything, agent.md frontmatter, statusline JSON contract) | OpenCode A− (its event surface let omO build ~80% of a harness *without* owning the loop — then omO forked the loop anyway: extensibility ceiling demonstrated), pi A− (deepest typed contract: cancelable pre-events, tool-input revision, cache_warming_decision economics), Zero B+ (plugins/hooks/custom tools/skills, but executive/tools-freeze seams still internal), goose B (extensions-as-subprocess + ACP), gemini B (extensions + SDK) |
| Roles as configuration | **Roo A−** (declarative modes = roleDefinition+capability groups+fileRegex; enforceable half = permission profiles; caveat: role-play half has zero measured value) | antigravity B+ (agent.md), OpenHands B (frontmatter subagents), Superpowers B (skills-as-process, writing-skills TDD protocol worth porting), Zero C+ (agents + per-agent/per-session declarative permission rulesets ship, e.g. the design plugin's deny-list; named reusable profile groups absent) |
| Provider abstraction | **Zero A−** (AI-SDK breadth + native Effect protocols side by side, repair layer) | codex B+ (model metadata table discipline), openhands B (RouterLLM + budget), aider B− (litellm — rejected pattern), goose B (YAML providers + ACP) |

## I. Observability & instrumentation

| Sub-dimension | Leader | Notes |
|---|---|---|
| Harness-diagnostic instrumentation | **Zero** (telemetry ledger: prefix_broken, engine-vs-observed decode delta, duplicate-read/noop-edit flags, local dashboard) — *the only surveyed harness measuring its own cache-breakage and overhead*; gap: the ledger DB itself is unread by the loop (the engine plugin does consume live probes into measured policy — thinking suppression for title/compaction with before/after decode numbers; 01 D3) | Antigravity B^c (cache_read_tokens, denied_actions, transcript.jsonl, statusline state), kimi B (per-step decode telemetry journaled), omO B+ (ledger with 17 event types + fingerprint reuse), claude B^c (cost-state rows, toolStats), aider C+, mini-swe C |
| Trajectory/machine readability of runs | Antigravity A−^c (closed step_type NDJSON), Claude A−^c (stream-json), codex B+ (rollout items), OpenHands A (event JSON log) | Zero C+ (durable events exist; no stable replay contract), pi B (JSONL tree), aider C |

## Cross-cutting: strongest known implementation per capability (one line each)

- Loop skeleton: **Codex**; durable-admission loop: **OpenCode** (Zero V2 equal design, unshipped).
- Tool reliability layer: **Zero Code** (edit cascade + output shaping + repair).
- Minimal toolset: **pi** (product) / **mini-SWE** (eval).
- Repo map: **aider** (constants documented; Zero port exists flag-off).
- Context pipeline: **Gemini CLI** (ground-truth triggering) with **Zero V1** (cache-aware compaction) close behind.
- Prompt-cache engineering: **Claude Code**, then **Zero** (open source, measured).
- Edit application for weak models: **Codex** patch ladder / **aider** per-model formats / **Zero** cascade — different valid answers; no evidence one dominates on identical models (untested — 06 E4).
- Checkpoints: **Zero / Codex / Antigravity** (three independent convergent shadow-snapshot designs).
- Durability substrate: **OpenHands SDK** (open) / **Antigravity** (closed).
- Scheduled autonomy: **goose**; goal/wake parking: **Kilo**; done-ness: **omO**.
- Multi-agent ergonomics: **Antigravity**; conflict-aware scheduling: **kimi**; measured architect/editor: **aider**.
- Model-quirk data tables: **aider / kimi / qwen**.
- Extension surface: **Claude Code**; extension depth per surface area: **pi**.
- Self-observability of harness effects: **Zero Code** (sole claimant; unused).

## Honest uncertainty ledger

1. Frontier closed-source ranks (Claude, Antigravity) are inferred from published/probe evidence; loop internals
   unverified — treat as ±1 rank.
2. All benchmark-driven claims (SWE-bench scores, "beats Claude Code" talk claims, pi transcripts) were
   **excluded** from ratings; nothing in this matrix depends on a leaderboard.
3. Same-model harness-vs-harness effect size is *unmeasured* across this whole survey except Zero's own ladder
   results (docs/research/2026-10-09-qwen-vs-sonnet-ladder); the matrix therefore compares *mechanisms*, and 06
   E1-E5 are designed to attach effect sizes to the top rows.
4. Several leaders are flag-off at their own heads (marked `s`); Zero's V2 ranks are design-credit, not
   shipping-credit — deliberate, and reversed by the roadmap's phase 1.
5. **Correction pass 2026-10-14:** rows touched by the plugin/desktop-layer blind spot (completion, wake,
   strategy-change, false-completion, read-freshness, permission profiles, telemetry consumption) were revised
   in place with *(corrected)* markers; full verdict table in PROGRESS.md, evidence in 01 Part D.
