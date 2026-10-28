# Index

`S` = `surveys/2026-10-13-harness-survey/`, `A` = its `02-COMPETITOR_AUDITS/`, `T` = `studies/2026-10-14-subsystems/`,
`E` = `evaluation/`, `L` = `literature/`.

## By harness

| harness | audit | also covered in |
|---|---|---|
| Zero Code (local-first, this project's origin) | `S/01-ZERO_CODE_BASELINE.md` (Parts A-D) | every study in `T/`; `E/results/` |
| OpenCode | `A/opencode.md` | `surveys/2026-09-13-harness-comparison.md`; `T/agent-kernel.md` |
| Codex CLI | `A/codex.md` | `surveys/2026-09-13-harness-comparison.md`; `T/agent-kernel.md`, `T/tools-editing.md` |
| Claude Code (closed; black-box plus published plugin types) | `A/claude-code.md` | `S/04`, `S/05` |
| pi | `A/pi.md` | `surveys/2026-09-13-harness-comparison.md`; `T/agent-kernel.md` (pi-durable); `E/results/2026-09-17-pi-*` |
| oh-my-pi | `A/tier3-scan.md` §2 | `surveys/2026-09-13-harness-comparison.md`; `T/agent-kernel.md` |
| Aider | `A/aider.md` | `T/tools-editing.md`, `T/providers-models.md` |
| Qwen Code | `A/qwen-code.md` | `T/autonomy.md` |
| DeepSeek Harness | `A/deepseek-harness.md` | `S/07` (programmatic tool calling) |
| Antigravity CLI (closed; changelog plus binary strings) | `A/antigravity-cli.md` | `S/03-ANTIGRAVITY_VS_GEMINI_CLI.md` |
| Gemini CLI | `A/gemini-cli.md` | `S/03`; `T/context-engine.md` |
| OpenHands SDK | `A/openhands-sdk.md` | `T/autonomy.md` |
| Goose | `A/goose.md` | `T/autonomy.md` (recipes and scheduler) |
| mini-SWE-agent | `A/mini-swe-agent.md` | |
| Cline, Roo Code, Kilo Code | `A/cline.md`, `A/roo-code.md` (light), `A/kilo-code.md` (CLI is an OpenCode fork) | |
| Crush | `A/crush.md` | `T/tools-editing.md` (enforced read freshness) |
| Kimi Code | `A/kimi-code.md` | `T/multi-agent.md` |
| oh-my-openagent, Superpowers, SWE-agent | `A/tier3-scan.md` | |

## By subsystem

| subsystem | design study | survey and evidence |
|---|---|---|
| Agent kernel, durability, recovery | `T/agent-kernel.md` | `S/05` F1, F7; `S/01` Part A; `E/methods/qwen-harness-study/` |
| Tools and editing | `T/tools-editing.md` | `S/05` F2-F4; `E/results/2026-09-17-tool-search-ab.md`, `2026-09-17-pi-tool-profile.md` |
| State-aware execution memory, revision-bound evidence | `L/2026-10-14-execution-state-evidence-memory.md` | Ledger, MemTrace, StructAgent; original evidence-bound runtime proposal and A/B protocol |
| Context engine, compaction, cache | `T/context-engine.md` | `S/05` F3; `S/06` R3-R5; `E/results/2026-09-23-long-horizon-results.md`; `L/2026-09-15-memory-architectures/` |
| Tool-call protocol reliability (parsers, schemas, streaming, settlement) | `L/2026-10-14-local-tool-protocol-reliability.md` | `E/methods/2026-10-14-local-tool-protocol-ab.md` (test design, not results) |
| Providers, model tables, local engines | `T/providers-models.md` | `L/2026-09-15-harness-and-engine/` (vLLM, llama.cpp, mlx-serve) |
| Autonomy: objectives, verification, wakes | `T/autonomy.md` | `S/05` §4; `S/01` Part D (goal plugin) |
| Multi-agent coordination | `T/multi-agent.md` | `S/05` §2 |
| Extensions: plugins, hooks, skills, MCP | `T/extensions.md` | `S/04` §H |
| Observability | `T/observability.md` | `S/04` §I |
| Server and client protocol | `T/` (protocol study, pending) | `S/03` §8 (Antigravity's machine contract) |

## New research program (2026-10-15)

- [High-leverage local harness research](literature/2026-10-15-high-leverage-local-harness-research.md): evidence matrix and technical research agenda covering adaptive supervision, retrieval, cross-task memory, regression-gated harness evolution and harness-to-model distillation. **Published external findings and proposed experiments; no new local results.**

## By question

- **How much does the harness matter with the model held fixed?** `E/results/SYNTHESIS-2026-09-29.md`,
  `E/results/2026-09-17-scorecard.md` (five harnesses, one local Qwen), `E/results/2026-09-23-long-horizon-results.md`,
  `E/results/2026-10-09-qwen-vs-sonnet-ladder.md`, and `L/2026-10-14-proven-advances.md` §1.
- **How do agents prevent stale read/test evidence from surviving edits and compaction?** `L/2026-10-14-execution-state-evidence-memory.md` (revision-bound evidence, runtime guards, controlled ablations).
- **What makes long-running work fail, and what fixes it?** `S/05` §4, `T/agent-kernel.md`, `T/autonomy.md`,
  `S/06` R5 (compaction thrash at 24k).
- **Can a local model's tool failures come from its template, parser, stream or harness rather than its reasoning?** `L/2026-10-14-local-tool-protocol-reliability.md`; same-model evaluation plan in `E/methods/2026-10-14-local-tool-protocol-ab.md` (no Zero Code measurements yet).
- **What helps local and weaker models?** `S/05` §3, `T/providers-models.md`, `E/results/2026-09-17-pi-tool-profile.md`.
- **How do I compare harnesses fairly?** `E/methods/2026-09-16-harness-comparison.md`,
  `E/methods/2026-09-17-all-around-leaderboard.md`, `E/methods/qwen-harness-study/` (design, decisions, OpenBench
  audit), `S/06`.
- **What does the latest literature prove?** `L/2026-10-14-proven-advances.md` (harness effects, harness adaptation,
  context, tools, verification, monitors, multi-agent, open models), with verification marks per claim.
- **What is cosmetic or overrated?** `S/05` §2 and §5.
- **What did Google change from Gemini CLI to Antigravity CLI?** `S/03`.
- **What was claimed but turned out wrong?** `S/PROGRESS.md` (44-claim verification table) and the "beyond or
  correcting the audits" section of each study.
