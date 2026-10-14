# Sources

Repository provenance for the harness research. Clone location:
`~/DevData/Projects/zero-code-next/Datasets/harness-research/repos/<name>` (shallow, default branch). Cloned
2026-10-13/14. "Head" is the tip commit examined (date in YYYY-MM-DD). Branches/history fetched only where noted.

| Project | Listed URL | Resolved/clone URL | Clone path | Head commit (date) | Notes |
|---|---|---|---|---|---|
| OpenCode | github.com/anomalyco/opencode | same | repos/opencode | 3ec856b6c265 (2026-10-14) | |
| Aider | github.com/Aider-AI/aider | same | repos/aider | 5dc9490bb35f (2026-05-22) | default branch activity slowed; check dev branches later |
| Pi | github.com/earendil-works/pi | same | repos/pi | 3daa20280f6a (2026-10-13) | |
| OpenAI Codex | github.com/openai/codex | same | repos/codex | 6786dccbe726 (2026-10-13) | |
| Claude Code | github.com/anthropics/claude-code | same | repos/claude-code | b0b7bed98203 (2026-10-13) | repo is issues/docs/extension surface, not core source; black-box dominated |
| Qwen Code | github.com/QwenLM/qwen-code | same | repos/qwen-code | 6212189f1f83 (2026-10-13) | |
| DeepSeek Harness | github.com/deepseek-ai/deepseek-harness | same | repos/deepseek-harness | d74326738864 (2026-10-09) | release 0.2.1-alpha.2 |
| Antigravity CLI | github.com/google-antigravity/antigravity-cli | same | repos/antigravity-cli | 9eebbedf051f (2026-10-13) | v1.3.4 changelog; check what source is actually published |
| Gemini CLI | github.com/google-gemini/gemini-cli | same | repos/gemini-cli | 42c51822084a (2026-10-13) | v0.65.0-preview.0 |
| OpenHands Agent SDK | github.com/OpenHands/software-agent-sdk | same | repos/openhands-sdk | a430c778bd8a (2026-10-13) | |
| Goose | github.com/aaif-goose/goose | same | repos/goose | e9e0ab62c273 (2026-10-13) | listed URL resolves; verify relation to block/goose |
| mini-SWE-agent | github.com/SWE-agent/mini-swe-agent | same | repos/mini-swe-agent | 04d809ceab9d (2026-09-03) | |
| Cline | github.com/cline/cline | same | repos/cline | 26cea2fa8bc4 (2026-10-13) | |
| Roo Code | github.com/RooCodeInc/Roo-Code | same | repos/roo-code | b867ec914575 (2026-05-15) | default branch stale since May; inspect branches |
| Kilo Code | github.com/Kilo-Org/kilocode | same | repos/kilo-code | 4ee8d1b3b1f8 (2026-10-13) | |
| Crush | github.com/charmbracelet/crush | same | repos/crush | d6edc3222231 (2026-10-13) | |
| Kimi Code CLI | github.com/MoonshotAI/kimi-code | same | repos/kimi-code | b510fa9aec85 (2026-10-13) | |
| oh-my-openagent | github.com/code-yeongyu/oh-my-openagent | same | repos/oh-my-openagent | 5275771d8b38 (2026-10-14) | v5.1.33 |
| oh-my-pi | github.com/can1357/oh-my-pi | same | repos/oh-my-pi | bcd11cb1ccb (2026-10-13) | |
| Superpowers | github.com/obra/superpowers | same | repos/superpowers | bb92a7774141 (2026-10-09) | v7.0.0 |
| SWE-agent | github.com/SWE-agent/SWE-agent | same | repos/swe-agent | 3ea751c087f3 (2026-07-16) | |

## Installed CLIs (black-box probes)

| CLI | Version | Date checked |
|---|---|---|
| claude | 2.1.296 | 2026-10-13 |
| gemini | 0.36.0 | 2026-10-13 |
| codex | codex-cli 0.162.1 | 2026-10-13 |
| opencode | 1.18.30 | 2026-10-13 |
| antigravity | not on PATH | 2026-10-13 |

## Local reference material (verified present)

- `docs/research/2026-09-15-harness-and-engine/` — Pi talk transcript (`pi-armin-ronacher-transcript.md`), RSI harness
  survey (`rsi-harnesses.md`), engine measurements.
- `docs/research/2026-10-13-research-plan.md`, `docs/research/2026-10-09-qwen-vs-sonnet-ladder/`,
  `docs/research/2026-10-06-local-inference/`, `docs/research/2026-10-07-engine-decode/`.
- `packages/zero-eval` — evaluation harness with long-horizon tasks and `--context` override.
- `~/DevData/Projects/pi-eval` — contains `agent/`, `Dependencies/`.
- Local model server for same-model comparisons: `http://127.0.0.1:11234/v1`, model `qwen3.8-flash` (shared with this
  session — timing confounded while this session is active).
