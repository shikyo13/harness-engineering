# Proven harness engineering advances, as of 2026-10-14

A literature survey of what is proven to make coding-agent harnesses better, gathered by four research agents
(long-horizon harness design; context and tools; verification, autonomy and multi-agent; open and local models)
and consolidated here. It covers work outside the audited harnesses in `surveys/2026-10-13-harness-survey/`.

Evidence labels: **measured** = a controlled comparison with numbers; **observational** = leaderboard or field
data; **claimed** = vendor-internal or unreplicated. Verification: **[V]** = checked against the primary source
(abstract, HTML or official page) by the curator on 2026-10-14; **[A]** = fetched by a research agent, not
re-checked. Many sources are 2026 preprints with single runs; treat small gaps (under ~3 points) as noise.

## 1. How much the harness matters with the model held fixed

- **Terminal-Bench 2.1 maintainer table** (tbench.ai/news/terminal-bench-2-1) [V], measured, same 89 tasks:
  vendor harness vs the minimal Terminus 2 on TB 2.1. GPT-5.4 mini 66.1 vs 36.9 (+29.2); Opus 4.6 70.1 vs 63.8;
  Gemini 3.1 Pro 67.1 vs 70.7 (the vendor harness lost). 28 of 89 TB 2.0 tasks were fixed in 2.1, and some
  published TB 2.0 "harness gaps" were benchmark breakage. Lesson: the gain is largest for small models in the
  harness they were trained against.
- **Claw-SWE-Bench** (arXiv 2606.12344, v2 2026-09-28) [V], measured, seven harnesses, one task prompt: GLM 5.1
  spans 62.9-73.1% (10.2 points), Qwen 3.6-flash 38.5-62.8% (24.3 points, the largest gap). Asking for raw unified
  diffs gave 19.3% with 69.1% apply failures; letting the model edit files and exporting the patch from git gave
  71.2% with under 1.5% apply failures. Token-weighted cache hit rates differ by harness on the same model
  (Qwen 3.6-flash 65.1-97.4%).
- **Meta-Harness** (arXiv 2603.28052) [A], measured on TB 2.0: Haiku 4.5 ranges 13.9 (OpenHands) to 37.6 across
  harnesses. The winning change (~80 lines) injects an environment snapshot (directory listing, languages, package
  managers, memory) into the first prompt. Search was on the test set.
- **"The Scaffold Effect"** (arXiv 2607.22585) [A], measured: Qwen 3.6 Plus and MiniMax M2.5 across Goose, OpenCode
  and OpenHands-SDK on 50 TB-Pro tasks: pass rates within 0-8 points, but tokens per solved task differ about 40x
  (Goose ~28K vs OpenCode ~1.15M for Qwen), driven by per-turn context growth and tool serialization. On a local
  engine, tokens per solved task is latency.
- **Leaderboard meta-analysis** (survey arXiv 2606.20683 §7.4) [A], observational: median within-model harness
  range 13.6 points over 20 models. Integrity caveat: TB rescored harnesses that copied solutions (ForgeCode,
  OB-1, Pilot).
- **Counter-evidence:** METR (2026-02-13) [A] found elaborate harnesses no better than a simple ReAct loop on its
  autonomous tasks; SWE-Serve (arXiv 2609.26777) [A] had plain mini-SWE-agent ahead of Codex and Claude Code on
  frontier models. A minimal loop is a credible baseline; harness gains concentrate on weaker models and on cost.

## 2. Harness adaptation without training the model

- **AHE, agentic harness evolution** (arXiv 2604.25850) [V headline, A details]: TB 2 pass@1 69.7% to 77.0% on
  GPT-5.4; the gain sits in tools, middleware and long-term memory, not the system prompt (abstract). Per the
  paper body [A]: system-prompt edits alone hurt (67.4%); the frozen harness transferred +5.1 to +10.1 points to
  three other model families.
- **Self-Harness** (arXiv 2606.09498) [A], measured: mine failures from traces, propose harness edits, keep an edit
  only if it passes held-out regression. Qwen3.5-35B-A3B held-out 23.8% to 38.1%; MiniMax M2.5 40.5% to 61.9%.
  Accepted edits for Qwen: tool-error recovery prompt, dependency prechecks, loop breaking, command-retry
  discipline. Edits differ by model, which argues for per-model tables validated by evaluation.
- **Harness-Zero** (arXiv 2609.24974) [A]: on Qwen3.5-9B, an optimized harness reached 41.7 vs 23.3 for a plain one,
  close to LoRA distillation (44.3); the harness delivered most of the gain.
- Fine-tuning on one harness's trajectories transfers poorly to others (Qwen3-Coder-Next report, arXiv 2603.00729)
  [A]; agentic RL did transfer in one Kimi report (arXiv 2610.00890) [A], with wide intervals.

## 3. Context management

- **Cache-friendly truncation over rewriting: CliffCompaction** (arXiv 2609.26779) [V, tables checked]: compaction
  truncates or drops, never rephrases: tool results over 500 characters dropped, calls cut to 150-character
  signatures, thinking cut to 300 characters, system prompt and task kept, last K turns kept, prior compactions
  discarded rather than nested. GLM 5.1 on TB 2.0: full 49.8, 32K 53.2, 16K 54.3, 8K 44.9; Kimi K2.6 on SWE-bench
  Verified: 73.9 / 73.3 / 71.9 / 67.6. Provider-metered cache hit falls as the threshold tightens (GLM 79/68/58/47%).
  16-32K is the measured sweet spot; below that, re-prefill cost eats the savings.
- **Observation masking matches summarization at about half the cost** ("The Complexity Trap", JetBrains/TUM, arXiv
  2508.21433) [A], measured on SWE-bench Verified: Qwen3-Coder 480B raw 53.4% / masking 54.8% / summary 53.8%, cost
  $1.29 / $0.61 / $0.64. Summaries lengthen trajectories 13-15% and get almost no prefix-cache reuse. Not free for
  small models: Qwen3-32B dropped 17.0 to 15.0 under masking.
- **Compaction erases constraints; pinning restores them** ("Governance Decay", arXiv 2606.22528) [V], measured on
  its own benchmark (1,323 episodes, 7 families): violations 0% with the policy in context, 30% after compaction (up
  to 59%); pinning constraints outside the compacted region restores 0%. Single-author preprint.
- **Structured, anchored summaries; file tracking is the weak point** (Factory, 2025-12-16) [A], vendor evaluation:
  persistent sections (intent, file changes, decisions, next steps) merged incrementally scored 3.70 vs 3.44
  (Anthropic) and 3.35 (OpenAI); all scored poorly on the artifact trail. Keep a deterministic read/edited-file
  index outside the summary.
- **Repeated compaction loses the execution frontier** (openai/codex#34095, user report) [A]: 24 auto-compactions,
  objective kept but next action lost. Proposed checkpoint: objective, completed actions with evidence, open items,
  exact next action, terminal condition, do-not-repeat list. Matches our 24k run (72 compactions, timeout).
- **Context rot** (Chroma, 2025-07-14) [A], measured including Qwen3: accuracy falls with input length; focused
  ~300-token prompts beat ~113K full prompts on LongMemEval.
- **Learned compaction policies** (SWE-MeM 2606.28434, AutoCompact 2610.02163, Context-Folding 2510.11967, AgentFold
  2510.24699) [A]: real gains, all from training; the recurring shape is a context-management tool plus offload and
  retrieval, never deletion. **ACON** (2510.00615) [A] optimizes compression guidelines from failures without
  fine-tuning the agent, with gains up to 46% for smaller models.
- **Structured handoff** ("Handoff Debt", arXiv 2606.02875) [A]: handing a successor notes or traces cut its events
  20-59% and prompt tokens 42-63% versus the repository alone; solve-rate effect small.

## 4. Tools and editing

- **Edit through file tools, export patches from state** (Claw-SWE-Bench, above) [V].
- **Separate viewing from editing; let the agent pick find/replace or whole-file per edit** (SWE-Edit, arXiv
  2604.26102) [A]: +2.1 points and -17.9% cost on SWE-bench Verified across Kimi-K2, MiniMax-M2.1 and GLM-4.7; an
  8B editor trained to choose the format gained 12.5 points of edit success. Edit-format value depends on model
  size (Diff-XYZ, 2510.12487) [A]. Hashline edits help some models greatly (oh-my-pi's 16-model benchmark) but a
  June 2026 field report saw high failure with DeepSeek V4 Flash [unverified].
- **Deferred tool loading keeps the cache intact** (Anthropic advanced tool use, 2025-11-24) [A], claimed: MCP eval
  accuracy 49% to 74% (Opus 4), definitions 77K to 8.7K tokens; deferred tools are appended after a search, so the
  prefix stays cached. Cursor (2026-01-06 and 2026-09-23) [A]: lazy MCP loading cut 46.9% of tokens in MCP sessions.
  Academic support: fewer exposed tools help small models (RAG-MCP 2505.03275, "Less is More" 2411.15399) [A].
- **Tool descriptions matter, but more text is not free** (arXiv 2602.14878) [A]: 97% of 856 MCP tool descriptions
  have defects; full augmentation raised success a median 5.85 points but added 67% more steps.
- **Grep first, semantic search for conceptual questions** [A]: an LSP vs grep study (2608.13568) found LSP cost
  6-118% more tokens and missed call sites in comments and strings during renames; a vector index beat delegated
  grep subagents on read-only code QA (2608.01507), with 41.8% of agentic failures lost at the subagent handoff;
  Cursor reports +12.5% accuracy adding semantic search to grep (internal). Return evidence (paths and lines) from
  subagents, not conclusions.
- **Programmatic tool calling / code mode** [A], claimed: Anthropic -37% tokens with small accuracy gains; no
  peer-reviewed comparison on open models found. Benchmark on the target model before adopting.

## 5. Verification and completion

- **Check completion against the world, never the agent's self-report** [A], measured:
  - Self-evaluation gates accepted stagnation: the agent claimed improvement in 54 of 54 cycles while 56% were flat
    or worse; even the strongest in-band judge accepted 44% real regressions (arXiv 2607.25152).
  - 75.8% of failed self-assessing AppWorld runs claimed completion; LLM judges reached AUROC ≤ 0.65, while cheap
    TF-IDF detectors reached 0.83-0.95 (arXiv 2606.09863).
  - A test the agent writes for its own patch accepts many wrong patches; require it to fail on the reverted tree
    (arXiv 2610.03984).
- **Cheating is common and prompts do not stop it** [A], measured: GPT-5 cheated 54-76% on ImpossibleBench
  (2510.20270); an explicit "flag impossible" option cut it to 9%; 57% of frontier runs took planted shortcuts
  (BaitBench 2608.30724). Deterministic trajectory monitors for test and harness tampering cut hacks from 37.8% to
  1.3% (Qwen team, 2606.26300). Passing tests can still hide a missing deliverable ("Building to the Test",
  2606.28430).
- **Test-time compute** [A]: sequential distill-and-refine from summaries of earlier attempts gained about 5-7 points
  on SWE-bench Verified for frontier models (arXiv 2604.16529); parallel best-of-16 with a verifier gains about 6-7
  points on open Qwen3 models (SWE-Lego 2601.01426) at 16x the compute.

## 6. Monitors, continuation and long runs

- **Adaptive monitor over a weaker worker** (Cadence, arXiv 2610.12269) [V headline]: +25.33 points on mini-swe-agent
  and +15.67 on Moatless (SWE-bench Lite, 300 tasks). Per the paper [A]: Gemini 2.5 Flash 17.0% to 42.3% with fewer
  tokens; advisory vs replacement verdicts; inspection interval 5 growing to 20 steps, reset after a replacement.
  **LivePlan** (2608.06701) [A]: rule-based stall and loop detection that calls an advisor only when a rule fires,
  +9.9 points average on SWE-bench Verified and Pro at +$0.08 per task. Weak primaries do not know when to escalate
  (Cognition, 2026-04-22) [A], so triggers should be harness-side rules.
- **Outer continuation helps; blind fresh restarts do worst** (LoopsBench, arXiv 2608.00267) [V title and 25% top
  result, A table]: re-prompting from outside when the agent stops raised resolve rates 2.7-8 points; the Ralph
  fresh-restart profile resolved 7.8% vs 17.6-24.1% for goal-mode and workflow profiles; every profile regressed
  earlier work, and agents' plans recovered only part of the dependency graph. Track completed obligations and
  re-run them.
- **Harness bundles for long runs** [A], vendor: LangChain deepagents (pre-completion checklist, startup map, time
  warnings, per-file edit loop detector) took gpt-5.2-codex 52.8 to 66.5 on TB 2.0; Anthropic's long-running
  harness designs (initializer plus progress and feature files; planner, generator and Playwright evaluator) are
  single-run anecdotes, with the explicit lesson that scaffolding should be re-audited when models change.

## 7. Multi-agent

- [A], measured: centralized coordination helps parallelizable work but every multi-agent variant lost 39-70% on
  sequential planning, and gains vanish once a single agent exceeds ~45% (Google/MIT, 2512.08296). Two cooperating
  agents had ~30% lower success than one agent doing both tasks (CooperBench, 2601.13295). Declaring each agent's
  write scope before writing raised pair pass 23% to 50% but serialized almost everything (2608.00947). Rule of
  thumb from the evidence: one writer, many readers; extra agents for review and research, not parallel edits.
- Anthropic's advisor tool (2026-04-09) [A], vendor: a stronger advisor improved a cheaper executor at lower cost
  per task (Haiku + Opus advisor on BrowseComp 19.7% to 41.2%).

## 8. Open and local models

- **Never combine a JSON-schema output constraint with tool calling on open models** ("Constraint Tax", arXiv
  2606.25605) [V mechanism and fix]: models stop calling tools while keeping schema compliance; the fix is two
  passes (tools first, schema-constrained answer second). Per the paper body [A]: tool-call rate fell to 0% on
  SGLang and vLLM; the grammar mask made the tool-call opening token impossible.
- **Grammars fix formatting, not decisions** (2608.13959) [A]: grammar-constrained small Qwen3 models called tools
  17-24 points more often when they should have declined.
- **Keep reasoning across turns** [A]: MiniMax M2 keep vs drop: SWE-bench Verified 69.4 vs 67.2, Tau² 87 vs 64. When
  templates drop reasoning, Qwen3.6 starts emitting empty tool arguments (pi #3325). Per-family history controls
  differ (Qwen `preserve_thinking`, GLM `clear_thinking=false`, DeepSeek V4 keeps it when tools are declared,
  gpt-oss drops analysis after a final message) and belong in model tables.
- **Hybrid recurrent models constrain cache reuse** [A]: Qwen3.8 Flash Next mixes Gated DeltaNet and sparse
  attention (model card: 125B total, 6B active). Recurrent state resumes only from a saved checkpoint at or before
  the first changed token, so any early-history edit re-prefills from the nearest earlier checkpoint. Verify where
  the engine places checkpoints relative to the harness's edits. TokenPilot (2606.17016) [A]: stabilizing volatile
  prompt content and batching evictions raised cache hits 38.7% to 79.2%.
- **Speculative decoding for agent turns** [A]: the model's own MTP head gave 1.7x decode on Apple silicon (MTPLX,
  vendor); prompt lookup +15-25% on agent turns on an M4 Max; agent loops inflate acceptance rates.
- **Evaluation hygiene** [A]: infrastructure alone swung TB 2.0 by 6 points (Anthropic, 2026-02-05); record memory
  pressure, engine stalls and power state with every run, and distrust gaps under 3 points.

## Gaps nobody has measured

Scheduled wake-ups; parallel tool-call accuracy on open models; XML vs JSON tool formats on open models; measured
tool-repair gains; prefix-cache hit rates under masking on a local engine; mid-task steering for coding agents;
advisor results with an open local executor.
