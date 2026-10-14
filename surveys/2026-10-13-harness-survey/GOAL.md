# Goal: harness research for Zero Code

Started 2026-10-13. The owner's assignment is below, verbatim, after these operating notes.

## Operating notes for this run

- Work in `~/Documents/Dev/zero-code-next` (branch `long-horizon`). All deliverables go in `docs/harness-research/`.
  Commit only files under `docs/harness-research/`, in focused commits such as `docs(research): audit aider repo map`.
  Never stage or revert other changes in the repository; other work happens on this branch at the same time. Do not
  push.
- Keep durable progress in `docs/harness-research/PROGRESS.md`: the work packages, each one's status, the commit SHAs
  examined, and the next step. Update it after every work package, so a compaction or restart loses nothing. Keep
  the todo list current with todowrite.
- Clone repositories under `~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/<name>`, never inside
  the repository. Start shallow and fetch branches or history only when needed. Record each repository's URL, commit
  and date in `docs/harness-research/SOURCES.md`. Some listed repositories may have moved, been renamed or not exist:
  verify each one, record what you find, and never invent a location or an implementation.
- Prior work to reuse rather than repeat (verify before relying on it): `docs/research/2026-09-15-harness-and-engine/`
  (includes a Pi talk transcript and an RSI harness survey), `docs/research/2026-10-09-qwen-vs-sonnet-ladder/`,
  `docs/research/2026-10-13-research-plan.md`, the evaluation harness in `packages/zero-eval` (it already has
  long-horizon tasks and a `--context` override), and `~/DevData/Projects/pi-eval`.
- Installed CLIs: `claude`, `gemini`, `codex`, `opencode`. Use only accounts that are already signed in. Never create
  accounts, enter credentials, enable paid plans or make paid API calls; if a black-box test needs any of these,
  record it as blocked and move on. Never print credential files, auth files or environment values. Short black-box
  probes with the signed-in CLIs are fine; long benchmark runs on them need the owner's go.
- Ask the owner before downloading or running any binary, installer or package that is not already on this Mac, and
  never remove macOS quarantine attributes (added 2026-10-13 after the Antigravity probe; source clones, reading
  files and the installed CLIs are unaffected). If a black-box test needs such a download, record it as blocked.
- Same-model comparisons can use the local model: OpenAI-compatible at `http://127.0.0.1:11234/v1`, model
  `qwen3.8-flash`. This session runs on that same engine, so another harness using it competes with you. Run such
  experiments one at a time as background shell jobs, compare pass/fail, steps, tokens and tool errors, and mark any
  wall-clock timing taken while the engine was shared as confounded. Experiments that need clean timing go into the
  experiment design deliverable (06) as planned runs for a quiet night instead.
- Subagents share the same single engine: use them to keep contexts small and findings separate, not for speed.
- This goal is research and planning only. Do not change Zero Code's code. Record Zero Code bugs or limitations you hit
  while working in `docs/harness-research/ZERO_CODE_ISSUES.md`.
- When section 13's definition of completion holds and all ten deliverables exist, end your reply with the line
  `GOAL COMPLETE`. Not before.

## Correction pass (owner review, 2026-10-14)

The first pass finished on 2026-10-14 at 00:15. The owner's review found a blind spot that changes conclusions, so
the goal is open again until these are done:

1. **Audit Zero Code's plugins at source level.** The baseline never read `packages/zero-plugins/` beyond telemetry:
   `goal`, `guard`, `engine` and `design` are unaudited. The goal plugin keeps an objective, sends continue prompts,
   verifies completion (the `GOAL COMPLETE` check), detects refusals, restates the goal after compaction and tracks
   todos; this research ran under it, so your own session in `~/.local/share/zero/zero-main.db` is direct evidence
   of how it behaves. Add the findings to `01-ZERO_CODE_BASELINE.md` as a new part, with `file:line` references and
   honest weaknesses.
2. **Correct every conclusion that rested on the gap.** `09-IMPLEMENTATION_ROADMAP.md` says "no wake primitive, no
   objective/done-ness anywhere"; `10-EXECUTIVE_ASSESSMENT.md` says the autonomy organ is absent. Fix those and every
   dependent claim in 04, 05, 07, 08, 09 and 10. The autonomy phase must start from what the goal plugin already
   does: extend, harden or replace it on evidence, not build from scratch.
3. **Re-verify every other "Zero Code lacks X" claim** in the matrix, findings, synthesis, architecture, roadmap and
   assessment against the whole monorepo (`packages/*`, including `app`, `desktop`, `zero-plugins`, `zero-eval`),
   not only `packages/zero` and `packages/core`. List each claim with its verdict in `PROGRESS.md`.
4. **Reflect code changed since the audit.** `3d338bc44f` (2026-10-14): end-of-loop pruning now runs only when the last
   turn filled 75% of the usable window, because it broke the prompt cache. `dc49a6f04a` fixed eval boot. Update any
   text that describes the old behaviour, including the "prune-before-compaction risk".
5. **Small repairs:** mark WP02 done in `PROGRESS.md`; the Roo Code audit is the thinnest (6.8 KB), so deepen it if any
   synthesis or roadmap item relies on it, or say plainly that it is a light audit.
6. **E3's multi-compaction cell** (planned in 06): if it can run as one background job on the shared engine, run
   `long-qa-spec` once at a context small enough to force compaction, and record the result honestly; otherwise
   leave it planned.

When all six are done and committed, recheck section 13 and end with `GOAL COMPLETE`.

---

# ZERO CODE — Project Synthesis
## Deep Architectural Research: Building the Ultimate Agentic Coding Harness

### Mission

You are conducting a comprehensive engineering investigation into the most capable agentic coding harnesses currently available.

Your assignment is to analyze their actual implementations, architectures, algorithms, execution models, agent loops, tools, context management, orchestration, and demonstrated performance.

Then design a concrete technical plan to evolve **Zero Code into a coding harness that is greater than the sum of the strongest existing implementations**.

This is not a request to collect features, imitate competitors, or write a generic product roadmap.

I want to understand **what makes certain coding harnesses substantially more capable than others, even when running identical underlying language models**, and how we can combine those advantages into one superior system.

Zero Code is the existing project and starting point. Its architecture is not sacred. Preserve what works, improve what does not, and recommend deeper redesigns wherever evidence justifies them.

The priority is real engineering capability, not marketing, feature counts, or architectural elegance for its own sake.

**This assignment is local research, deep comparison, architectural design, and implementation planning. Do not begin a large rewrite or implementation campaign yet.**

---

# 1. Research targets

Investigate the following projects using their current source where available, relevant development branches, recent PRs, technical documentation, issues, and reproducible demonstrations.

## Tier 1 — Mandatory deep technical audits

These are the primary systems I explicitly want compared.

| Project | Repository |
|---|---|
| OpenCode | https://github.com/anomalyco/opencode |
| Aider | https://github.com/Aider-AI/aider |
| Pi | https://github.com/earendil-works/pi |
| OpenAI Codex | https://github.com/openai/codex |
| Claude Code | https://github.com/anthropics/claude-code |
| Qwen Code | https://github.com/QwenLM/qwen-code |
| DeepSeek Harness | https://github.com/deepseek-ai/deepseek-harness |
| Google Antigravity CLI | https://github.com/google-antigravity/antigravity-cli |

For Claude Code and Antigravity CLI, distinguish publicly available implementation details, SDKs, documented behavior, experimentally observable behavior, and inaccessible internal code.

Where internal source is unavailable, conduct controlled black-box investigations rather than inventing implementation details.

For open-source projects, inspect actual implementation paths rather than treating README descriptions as established functionality.

## Tier 2 — Additional serious competitors and architectural references

| Project | Repository |
|---|---|
| Gemini CLI (original open-source predecessor) | https://github.com/google-gemini/gemini-cli |
| OpenHands Agent SDK | https://github.com/OpenHands/software-agent-sdk |
| Goose | https://github.com/aaif-goose/goose |
| mini-SWE-agent | https://github.com/SWE-agent/mini-swe-agent |
| Cline | https://github.com/cline/cline |
| Roo Code | https://github.com/RooCodeInc/Roo-Code |
| Kilo Code | https://github.com/Kilo-Org/kilocode |
| Crush | https://github.com/charmbracelet/crush |
| Kimi Code CLI | https://github.com/MoonshotAI/kimi-code |

Conduct meaningful architectural and implementation-level reviews of these projects.

Promote especially innovative implementations to full Tier 1 depth when warranted.

## Tier 3 — Orchestration, extensions, and alternative approaches

Investigate these for particularly valuable engineering ideas:

- oh-my-openagent (formerly oh-my-opencode): https://github.com/code-yeongyu/oh-my-openagent
- oh-my-pi: https://github.com/can1357/oh-my-pi
- Superpowers: https://github.com/obra/superpowers
- SWE-agent: https://github.com/SWE-agent/SWE-agent

Discover other genuinely innovative coding harnesses, research implementations, and architectural experiments that may be missing from this list.

Do not assume GitHub popularity correlates with technical superiority.

Prioritize projects with unusual execution strategies, independently verified benchmark results, exceptional long-horizon performance, low complexity, or novel model/tool interactions.

Exclude irrelevant general-purpose chatbot frameworks unless they introduce techniques directly useful to coding-agent capability.

Verify repository identities and current upstream locations before beginning substantial analysis.

---

# 2. Special investigation: Antigravity CLI versus Gemini CLI

Google transitioned its consumer coding terminal experience from Gemini CLI to Antigravity CLI in June 2026.

Antigravity CLI is a new implementation built around the Antigravity agent platform, rather than merely a renamed Gemini CLI.

This transition is a particularly valuable architectural case study.

Treat **Antigravity CLI as the primary contemporary Google competitor** and **Gemini CLI as an independently valuable open-source reference implementation**.

Investigate:

### Architectural differences

- What motivated the transition?
- What changed in the agent execution architecture?
- What changed in language, runtime, and process management?
- How is Antigravity's shared agent engine structured, to the extent publicly observable?
- How does the CLI relate to Antigravity's desktop agent environment?
- Which components are shared across those environments?
- What functionality was retained, replaced, or removed?
- Which differences reflect true architectural improvements rather than UI or packaging changes?

### Long-running execution

Study Antigravity's support for asynchronous workflows and background agent activity.

Investigate:

- How tasks continue after the immediate interactive turn.
- Session persistence and resumability.
- Agent steering during execution.
- Task interruption and cancellation.
- Background process ownership.
- Multi-step and multi-agent orchestration.
- Session transfer between CLI and desktop environments.
- Recovery after failure.
- Maintaining task state across long-running work.

Compare these mechanisms against Codex, Claude Code, OpenHands, Pi, and Zero Code.

### Tool behavior

Compare tool dispatch, terminal execution, code editing, permissions, asynchronous operations, and error handling.

Determine whether Antigravity introduces genuinely better execution behavior than Gemini CLI.

### Migration analysis

Use the original open-source Gemini CLI to establish a source-grounded architectural baseline.

Use publicly available Antigravity documentation, release notes, observable execution, issues, and controlled experiments to assess the successor.

Do not assume the two share internal source or execution machinery merely because features have similar names.

Identify which Gemini CLI techniques remain worth adopting independently.

### Comparative outcomes

Where environments permit, compare both systems on equivalent coding tasks using similar models and budgets.

Determine whether the new architecture improves:

- Task completion.
- Long-horizon reliability.
- Tool-call correctness.
- Responsiveness.
- Context efficiency.
- Recovery from failures.
- Multi-agent effectiveness.

Clearly mark comparisons that cannot isolate harness effects from model or infrastructure differences.

The key question is:

**What did Google learn from Gemini CLI that led it to build Antigravity CLI differently, and which of those lessons can actually improve Zero Code?**

---

# 3. First audit Zero Code itself

Before deciding what should be adopted, thoroughly inspect Zero Code's current implementation.

Establish exactly how it works today.

Identify and examine:

- Main agent execution loop.
- Model provider abstraction and tool calling.
- System prompt construction.
- Tool schemas and dispatch.
- File reading, searching, editing, and command execution.
- Context construction, reduction, and compaction.
- Session state and persistence.
- Long-running task execution.
- Cancellation, interruption, and steering.
- Failure recovery and retries.
- Subagents and parallel task execution.
- Shared project and task knowledge.
- Repository awareness.
- Model switching and provider behavior.
- Resource consumption and latency.
- Existing observability, metrics, and diagnostics.
- Known reliability problems and technical limitations.

Read the actual implementation and trace complete execution paths.

Do not infer capabilities from filenames, configuration flags, comments, or documentation alone.

Identify what Zero Code already does particularly well and what is unnecessarily complicated or inefficient.

Determine whether its architectural foundation is suitable for the intended system or whether a different foundation would produce better results.

A complete rewrite must be justified by measurable advantages over incremental improvement.

---

# 4. Reverse-engineer the competing harnesses

For each major project, reconstruct how the harness actually operates.

At minimum, trace:

1. How a user request enters the system.
2. How the model receives instructions and context.
3. How available tools are selected and represented.
4. How model responses and tool calls are parsed.
5. How tool execution happens.
6. How errors, partial output, and interruptions are handled.
7. How observations re-enter the model's context.
8. How the system determines its next action.
9. How sessions persist across turns and restarts.
10. How completion is determined.
11. How the system behaves after repeated failures.
12. How multiple agents or workers coordinate, when supported.

Identify actual classes, functions, modules, interfaces, and execution paths responsible for these behaviors when source is available.

Compare the designs, not just their advertised features.

For every important source-level finding, provide implementation references and relevant commit identifiers.

Inspect relevant development branches and PRs. A project's default branch may not represent its most advanced implementation.

Do not declare functionality absent simply because it exists on another development branch.

Classify findings as:

- Implemented and experimentally verified.
- Implemented with source-level evidence.
- Partially implemented.
- Documented but not substantiated.
- Experimentally observed in a closed-source system.
- Experimental.
- Not supported.
- Unavailable for inspection.

---

# 5. Compare the components that actually determine agent capability

This is the heart of the investigation.

## A. Agent-loop architecture

Compare how each harness structures its reasoning/action loop.

Investigate:

- Simple loops versus explicit state machines.
- Event-driven versus sequential execution.
- Streaming and asynchronous tool handling.
- Parallel and batched tool calls.
- Turn-level and task-level state.
- Recovery after interrupted turns.
- Infinite loops and repeated-action detection.
- Graceful cancellation and reliable stopping.
- Error propagation and automatic retry selection.
- Agent continuation after failed tools.
- Checkpointing and deterministic replay.
- Separation of model decisions from host execution.

Determine which mechanisms increase actual task-completion reliability.

Investigate whether some harnesses give models substantially better tool-use behavior through the structure of the loop itself.

## B. Tool interfaces and execution quality

Compare tools and their model-facing interfaces.

Examine file operations, patching, code search, terminal execution, persistent shells, Git, AST/LSP capabilities, browser control, computer use, MCP, and custom tool extensions.

Investigate the engineering differences behind reliability.

Important questions:

- Which tool schemas are easiest for weaker models to use correctly?
- Which file-editing approaches produce fewer corrupted edits?
- How should failed patches be repaired?
- How much tool output is actually useful to the model?
- Can tools return focused results without obscuring relevant information?
- Which harnesses preserve state across terminal operations?
- How are timeouts, cancellation, and process lifetimes managed?
- Do tool errors lead to intelligent recovery or repeated guessing?
- Can a harness reliably distinguish a failed operation from one that succeeded but returned no output?
- Does generating code to compose tools improve flexibility or create avoidable failures?

Pay particular attention to systems that achieve excellent results with unusually small toolsets.

Investigate DeepSeek Harness's code-execution-based tool composition and compare it with conventional tool calling.

Do not assume exposing more tools improves intelligence.

## C. Repository intelligence and context engineering

Study how each harness understands large repositories without exhausting its context window.

Investigate:

- Aider's repository mapping.
- Symbol and dependency discovery.
- Tree-sitter and LSP indexing.
- Dynamic file relevance ranking.
- Search-driven context retrieval.
- Incremental context updates.
- Context compression and summarization.
- Persistent memory across sessions.
- Relevant-history retrieval.
- Model-specific context packing.
- Prompt caching and cache stability.
- Handling very large tool results.
- Protection against important context being lost during compaction.
- Cross-agent context sharing without excessive duplication.

Determine whether a hybrid indexing and retrieval system can outperform the techniques used independently.

The goal is not maximum available context. It is **maximum useful information per token**.

Identify how these systems handle multi-hour or multi-day work without progressively losing the original objective.

## D. Editing and software-engineering workflow

Compare how systems generate, review, and apply code modifications.

Study:

- Whole-file replacement.
- Search-and-replace edits.
- Structured patches.
- AST-aware transformations.
- Model-generated diffs.
- Architect/editor approaches.
- Incremental changes.
- Checkpoints and rollback.
- Multi-file consistency.
- Merge conflict handling.
- Self-correction after build errors.
- Determining when a change is actually complete.

Evaluate Aider's architect/editor architecture especially carefully.

Investigate whether separating reasoning from code-edit application materially improves local-model reliability.

Compare this against more direct agent implementations.

## E. Long-horizon autonomy

This is one of the highest priorities.

I want Zero Code to run useful coding tasks for hours or days with little human intervention.

Determine how the strongest harnesses handle:

- Persistent objectives and subtasks.
- Task decomposition.
- Continuation after context exhaustion.
- Recovery after crashes or model failures.
- Durable checkpoints.
- Multi-session continuation.
- Progress evaluation.
- Detecting lack of progress.
- Strategy changes after repeated failures.
- Avoiding duplicated work.
- Human steering without restarting.
- Independent verification of completion.
- Maintaining project coherence over many edits.
- Preventing agents from declaring success prematurely.

Examine Ralph-style iteration and persistent agent loops critically.

Distinguish genuine autonomous progress from agents simply repeating prompts or endlessly generating activity.

Develop a better approach if current systems are inadequate.

## F. Multi-agent orchestration

Study actual multi-agent implementations, including OpenHands, Roo Code, DeepSeek Harness, OpenCode extensions, Antigravity, and other relevant systems.

Compare:

- Parent/child agents.
- Peer-to-peer collaboration.
- Shared versus isolated context.
- Independent worktrees.
- Concurrent coding.
- Task ownership.
- Dependency coordination.
- Agent-to-agent communication.
- Result verification.
- Conflict resolution.
- Shared progress and memory.
- Cross-model delegation.
- When parallel agents outperform one strong agent.
- When orchestration overhead makes outcomes worse.

Do not automatically prescribe artificial roles such as architect, reviewer, or programmer.

Models should be able to collaborate flexibly according to actual task requirements.

Determine whether multiple independent agents, one strong agent, or a hierarchical architecture is best for each workload.

## G. Model independence and model-specific optimization

Zero Code must work well with both local open-weight models and frontier models.

Study how harness behavior should adapt to differences in:

- Reasoning capability.
- Tool-call reliability.
- Context-window size.
- Prompt caching.
- Inference speed.
- Latency.
- Parallel request capacity.
- Output formatting reliability.
- Supported tool schemas.
- Recovery from malformed responses.
- Per-token or per-request cost.
- Long-running operational limits.

Do not assume the same prompts, tool interfaces, or context policies are optimal for every model.

Investigate the difference between adapting the harness to the model and adapting the model to the harness.

Evaluate whether small local models could perform specialized subtasks more efficiently than the primary model.

Consider local-heavy workflows where a stronger frontier model occasionally reviews, guides, or corrects a weaker autonomous agent.

However, do not hardcode this hierarchy without evidence.

## H. Extensibility and composition

Compare:

- Plugin architectures.
- Middleware.
- Hooks.
- Skills.
- MCP.
- ACP.
- Agent SDKs.
- Custom tool registration.
- Runtime extension loading.
- Event buses.
- Provider abstraction.
- Replaceable context strategies.
- Agent-loop customization.

DeepSeek Harness and Pi deserve particularly deep attention here.

Investigate whether the architecture can remain minimal while allowing advanced functionality to be composed when needed.

The target is not an enormous mandatory core.

It is a small, exceptionally reliable foundation capable of supporting sophisticated behavior.

## I. Observability and optimization

Investigate how the systems expose internal performance and failure information.

Measure or recover:

- Model reasoning and tool latency.
- Time to first token.
- Prompt and completion tokens.
- Cache hit rates.
- Tool success and failure rates.
- Repeated operations.
- Idle time and stalls.
- Context consumption.
- Compaction frequency.
- Edit success and rollback rates.
- Agent continuation effectiveness.
- Completion accuracy.
- Cost and compute consumption.

Determine what instrumentation Zero Code needs to diagnose why one model or harness performs better than another.

Metrics must inform useful engineering decisions rather than become decorative dashboards.

---

# 6. Determine what makes the strongest harnesses strong

Do not stop at a capability matrix.

I want a causal technical explanation for why certain approaches work better.

Examples of hypotheses to investigate:

- Does Aider's repository mapping reduce context waste?
- Does Pi's minimal tool interface improve tool-call reliability?
- Does OpenCode's architecture improve autonomy or primarily extensibility and usability?
- Does Codex achieve advantages through execution infrastructure, prompts, model specialization, or some combination?
- How much of Claude Code's apparent agent reliability can be explained by observable harness behavior?
- Does Antigravity's new shared agent engine improve long-horizon execution compared with Gemini CLI?
- Are Antigravity's asynchronous workflows fundamentally better than the background-task mechanisms of other harnesses?
- Does Qwen Code contain techniques particularly well suited to Qwen models?
- Does DeepSeek Harness's compositional architecture improve actual agent effectiveness or mainly customization?
- Does mini-SWE-agent demonstrate that much of the complexity in larger frameworks is unnecessary?
- Can OpenHands-style durable execution improve local long-running tasks?
- When does multi-agent orchestration improve real results, and when does it merely increase cost?

Do not accept these hypotheses as facts.

Test them where possible.

Distinguish model advantage from harness advantage.

When evaluating one harness against another, avoid conclusions based on results produced by substantially different models.

---

# 7. Design controlled comparative experiments

Develop a practical evaluation methodology for isolating harness capability.

Where possible, run identical models against identical tasks under different harnesses.

Use:

- Same model and model settings.
- Same starting repository state.
- Same task specification.
- Comparable context and tool access.
- Comparable token or computation budgets.
- Clear success criteria.
- Multiple trials where randomness could alter the outcome.

Include both local and frontier models when available.

Use tasks that measure actual software-engineering performance:

1. Bug diagnosis and repair.
2. Implementing a multi-file feature.
3. Large-repository navigation and comprehension.
4. Architectural refactoring.
5. Recovering from an incorrect early assumption.
6. Long-horizon feature development.
7. Debugging a failure introduced by the agent itself.
8. Tool-call and edit reliability.
9. Resume-after-interruption behavior.
10. Concurrent or multi-agent work on a shared project.
11. Mid-task steering without context loss.
12. Detecting false completion and unsupported success claims.

Include representative external benchmark tasks where useful, but do not rely entirely on published leaderboards.

Analyze task validity, contamination risks, and the difference between reported benchmark scores and reproducible local results.

Avoid manufacturing a massive testing framework. Build only the minimum evaluation machinery required to distinguish competing implementations and validate major architectural decisions.

Report failures and inconclusive results honestly.

---

# 8. Design a system greater than the sum of its parts

This is the most important deliverable.

Do not merely identify the strongest implementation in each category and place them beside one another.

Look for combinations that create new capabilities.

For example:

**Aider's repository intelligence + Pi's minimal agent loop + Codex's execution patterns** might produce a system with better context quality and fewer tool failures.

**OpenHands's durable execution + DeepSeek Harness's composability + Zero Code's existing autonomy** might yield stronger unattended operation.

**Antigravity's asynchronous task model + Pi's session control + a persistent project-level objective system** might improve background coding and reliable human steering.

**A model-aware tool interface + fine-grained observability + adaptive context selection** might let small local models complete tasks previously requiring much stronger models.

These are illustrative possibilities, not architectural requirements.

Find combinations supported by actual evidence.

For each proposed synergy, describe:

- Which existing approaches contribute.
- What new behavior becomes possible.
- Why the combination should outperform the original systems.
- Architectural requirements.
- Potential disadvantages.
- How to test the hypothesis.
- How to distinguish genuine improvement from added complexity.

I want at least five substantial synthesis opportunities that go beyond simple feature adoption.

Prioritize improvements to effective agent intelligence, tool reliability, autonomy, and software-engineering results.

---

# 9. Choose the right architectural foundation

Evaluate whether Zero Code should:

**Option A:** Retain its current architecture and selectively incorporate better algorithms and components.

**Option B:** Restructure major internal subsystems while preserving existing functionality and interfaces.

**Option C:** Adopt a superior existing agent runtime as an internal foundation and build Zero Code's differentiated capabilities around it.

**Option D:** Build a new core informed by the strongest implementations where existing foundations impose fundamental limitations.

Evaluate each option against actual technical requirements.

Do not favor a rewrite because a competing architecture looks cleaner.

Do not preserve Zero Code's current implementation merely because it already exists.

Consider performance, simplicity, operational reliability, long-horizon behavior, and future extensibility.

Where different runtimes offer complementary advantages, evaluate whether internal composition is superior to replacing everything with one monolithic implementation.

I want the smallest architecture that can deliver the strongest capabilities.

Avoid building a sprawling framework with overlapping abstractions, unnecessary layers, or hundreds of superficially distinct tools.

---

# 10. Develop the implementation roadmap

Produce a concrete, technically actionable plan for evolving Zero Code.

Organize the work by engineering dependencies and measurable impact.

Every phase must explain:

- Current limitation or opportunity.
- Specific architectural or code changes.
- External implementation or technique informing the solution.
- Dependencies.
- Engineering difficulty.
- Expected benefit.
- Concrete success criteria.
- Risks of performance or reliability regression.
- Appropriate order of implementation.

Identify which components can be developed independently.

Distinguish conventional engineering work from research requiring experimentation.

Identify high-value changes that can be implemented without restructuring the entire project.

Prioritize by expected improvement in **successful autonomous coding work per unit of inference, time, and compute**, not feature count.

Design an initial vertical slice that demonstrates meaningful improvement over the current Zero Code harness.

The roadmap must leave Zero Code functional at every major milestone.

---

# 11. Research execution requirements

This investigation is substantial. Work systematically and retain durable progress.

### Source-level investigation

Clone or use local checkouts of the identified repositories.

Record the exact commit examined.

Inspect actual implementation files, integration paths, and relevant tests.

Do not pretend to understand an implementation after skimming its README.

Review important branches and PRs.

Prefer primary source evidence and reproducible observations over community speculation.

For closed-source harness components, use observable behavior and explicitly state the limitations of that evidence.

### Research organization

Divide the investigation into independently reviewable work packages.

If Zero Code supports effective parallel research agents, use them where doing so improves throughput.

Coordinate findings through durable files rather than relying on one enormous conversational context.

Avoid duplicating research.

Use one consistent comparison framework across repositories.

Do not spend the entire research budget on the first few projects.

### Critical evaluation

Challenge assumptions continuously.

Identify areas where conventional wisdom is unsupported.

Do not declare one project superior solely because it is more popular, uses more tools, has more sophisticated architecture, or publishes a higher benchmark score.

Do not assume a feature described in documentation is implemented.

Do not claim architectural improvements are proven without validating them.

### Scope

This is a local engineering investigation focused exclusively on coding-agent capabilities and building a superior combined harness.

Do not divert into licensing, commercialization, unrelated product strategy, or administrative analysis.

Do not start implementing the complete redesign.

Complete the analysis, architecture, and executable development plan first.

---

# 12. Required deliverables

Create a research directory in the Zero Code repository:

`docs/harness-research/`

Produce:

### 01 — ZERO_CODE_BASELINE.md

Complete technical audit of the current Zero Code architecture, its strengths, weaknesses, capabilities, and known limitations.

Include key execution paths and concrete source references.

### 02 — COMPETITOR_AUDITS/

Detailed individual analyses for the Tier 1 repositories and substantial audits for the strongest Tier 2 candidates.

Each audit should explain:

- Actual architecture.
- Key implementation techniques.
- Distinctive strengths.
- Weaknesses.
- What genuinely works.
- What is unproven.
- Useful source files and implementation patterns.
- What Zero Code should learn from it.

Where source is unavailable, identify the evidence as black-box findings rather than a source audit.

### 03 — ANTIGRAVITY_VS_GEMINI_CLI.md

Dedicated comparative case study covering:

- Gemini CLI's original architecture.
- Antigravity CLI's documented and observable architecture.
- The motivations for Google's transition.
- Runtime and agent-loop differences.
- Asynchronous execution.
- Session persistence.
- Multi-agent behavior.
- Tool and context management.
- Retained and discarded features.
- Measured performance differences where possible.
- Lessons Zero Code should adopt.
- Remaining unknowns caused by inaccessible source.

### 04 — CAPABILITY_MATRIX.md

Comprehensive comparative matrix of the harnesses.

Rank meaningful capabilities by quality and demonstrated effectiveness, with uncertainty explicitly recorded.

Identify the strongest known implementation of each capability.

### 05 — AGENT_ENGINEERING_FINDINGS.md

Explain the deeper engineering lessons discovered.

Which designs genuinely improve agent effectiveness?

Which commonly praised features are mostly cosmetic?

Which mechanisms help local models?

Which approaches increase long-horizon reliability?

Which patterns should Zero Code explicitly avoid?

### 06 — EXPERIMENT_DESIGN.md

Practical evaluations to isolate harness effects from model effects.

Include reproducible tasks, measurements, comparison methodology, resource requirements, and success thresholds.

Include any actual experimental results obtained during research, clearly distinguished from planned evaluations.

### 07 — SYNTHESIS_OPPORTUNITIES.md

At least five well-reasoned proposals combining capabilities from multiple systems to create something none of them accomplish as effectively independently.

Provide technical rationale and experimental acceptance criteria.

### 08 — TARGET_ARCHITECTURE.md

Concrete proposed Zero Code architecture.

Include:

- Agent execution engine.
- Tool architecture.
- Context and memory systems.
- Repository intelligence.
- Persistent execution and recovery.
- Asynchronous task management.
- Multi-agent coordination.
- Provider integration.
- Observability.
- Component boundaries and migration strategy.

Use diagrams where they clarify real execution relationships.

### 09 — IMPLEMENTATION_ROADMAP.md

Prioritized and phased engineering plan with specific changes, dependencies, measurable objectives, and recommended implementation order.

This must be sufficiently detailed for coding agents to begin executing it without another round of broad architectural research.

### 10 — EXECUTIVE_ASSESSMENT.md

A direct and concise assessment answering:

1. How competitive is Zero Code today?
2. Which existing harness has the strongest overall architecture?
3. Which has the best agent loop?
4. Which has the most effective tool design?
5. Which handles context most intelligently?
6. Which has the best long-horizon execution architecture?
7. Which offers the most useful multi-agent coordination?
8. Which is particularly effective with weaker local models?
9. What did Google improve or change by transitioning to Antigravity CLI?
10. What are the five biggest mistakes in current coding harness design?
11. Which implementations should Zero Code adopt, adapt, or reject?
12. Should Zero Code keep its current core or adopt a different foundation?
13. What combination of techniques has the greatest potential to outperform every individual system?
14. What should be built first?

All substantive claims must be traceable to inspected source, documented behavior, or experiments.

---

# 13. Definition of completion

Do not confuse producing the requested documents with completing the investigation.

The assignment is complete when:

- Zero Code's current implementation is understood.
- The eight primary harnesses have received meaningful source-level or appropriately limited black-box audits.
- Strong additional competitors have been evaluated.
- Antigravity CLI and Gemini CLI have been compared as distinct implementations.
- Major architectural claims are supported by evidence.
- Important tradeoffs and uncertainties are recorded.
- Genuine cross-project synthesis opportunities have been identified.
- The proposed architecture is internally consistent and grounded in demonstrated capabilities.
- A practical implementation roadmap exists.
- The resulting plan explains how Zero Code can become measurably better than its inspirations.

Do not declare the full engineering mission complete merely because the research phase is finished.

This assignment ends with a researched design and actionable implementation plan, not a completed new harness.

## Final instruction

Treat this as a serious technical research assignment from an engineering owner who wants to build a superior autonomous coding system.

Be ambitious, skeptical, and technically specific.

Do not flatter existing projects.

Do not protect Zero Code from criticism.

Do not propose unnecessary complexity.

Do not mistake extra features for extra intelligence.

**Find out what makes the best coding harnesses genuinely effective, figure out how their strengths can reinforce one another, and design a Zero Code architecture that produces substantially better results than any of them alone.**

Begin the source-level and black-box investigation now.
