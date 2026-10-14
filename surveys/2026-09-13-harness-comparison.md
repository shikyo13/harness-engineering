# Agentic Coding Harness Comparison

Date: 2026-09-13. Prepared as input to the Zero Code Next design spec.

Sources were fresh shallow clones inspected file by file, not marketing pages:

|Project|Commit reviewed|Version|Repo|
|-|-|-|-|
|zero-code (the owner's earlier prototype)|`b45b655` (2026-04-27)|0.1.0|private|
|OpenCode|`df23b7f` (2026-09-13)|1.18.30|github.com/anomalyco/opencode|
|pi (pi-mono)|`71dca87` (2026-09-11)|0.85.1|github.com/earendil-works/pi (was badlogic/pi-mono)|
|oh-my-pi (omp)|`9b2a435` (2026-09-14)|18.1.20|github.com/can1357/oh-my-pi|
|Codex CLI|`3abbf9f` (2026-09-14)|0.0.0 (rolling)|github.com/openai/codex|

## 1. zero-code baseline

The owner's earlier prototype is not reusable as a base, and this rebuild carries none of its code forward. What is worth carrying forward is the owner's own work in it:

|Subsystem|Dir|LOC|Role in the rebuild|
|-|-|-|-|
|Codebase index (SQLite AST and symbol graph)|`src/index`|3,264|Port as a plugin tool|
|Event runtime|`src/events`|3,178|Superseded by OpenCode's bus|
|Transparent API proxy and dashboards|`src/proxy`, `src/usage`, `src/dashboard`, `src/cost`|5,080|Port as telemetry plugin plus local dashboard|
|OpenWolf bookkeeping|`src/openwolf`|2,589|Port as a plugin over the `openwolf` CLI|
|Reversible pruner|`src/pruner`|1,567|Port as a plugin (opt-in)|
|Context quality and intelligence metrics|`src/contextQuality`, `src/contextIntelligence`|1,572|Port as telemetry plugin|
|Defenses (canaries, stop phrases, edit guard, truncation, zombie reaper)|`src/defenses`|~2,000|Port as a plugin|
|Decay memory over markdown|`src/memdir` additions|~1,500|Port as a plugin|
|Provider adapters (Ollama, OpenAI-compatible, Codex, llama-server)|`src/providers`|~1,200|Superseded by OpenCode providers|
|Daemon, SSH, direct-connect, checkpoint, config, auth|various|~4,000|Superseded by OpenCode server and snapshots|

Also worth keeping: the operating rules in `.wolf/cerebrum.md` (show local-model reasoning, never drop it; operator-only cost surfaces; positional slash grammar; no refusal layers beyond destructive-action confirmation; no em dashes) and the upstream harvest notes in `docs/upstream/`.

## 2. Summary matrix

|Aspect|OpenCode|pi|oh-my-pi|Codex CLI|
|-|-|-|-|-|
|License|MIT (opencode)|MIT (Mario Zechner)|MIT (Zechner, Can Bölük, Stencil Labs), cargo-about notices|Apache-2.0 (OpenAI), CLA for contributions|
|Branding limits|Polite "not affiliated" request only|None, rebrand documented via `piConfig`|None|Apache §6 only; "Codex" name not granted|
|Language and runtime|TypeScript, Bun workspaces, server runs on plain Node, Effect|TypeScript, Node 22.19+|TypeScript on Bun 1.4 plus Rust nightly N-API addon|Rust (179 crates), TS and Python SDKs|
|Desktop app|Electron 42 (beta), server as `utilityProcess` sidecar, SolidJS renderer|None|None (React web guest client only)|Closed-source; only the app-server protocol is open|
|Core and UI split|HTTP plus SSE server, OpenAPI, SDK, ACP, TUI, web, VS Code|In-process SDK, stdio RPC and JSON modes|stdio `rpc-ui`, ACP, Python client|JSON-RPC app-server over stdio, UDS, WS with generated TS types|
|Local OpenAI-compatible models|Config block with `@ai-sdk/openai-compatible`|`models.json`, `openai-completions`, broad compat flags|`models.yml`, auto discovery, in-band tool dialect|Responses wire only; chat completions removed|
|Built-in tools|bash, read, glob, grep, edit (9 replacers), write, apply_patch, task, webfetch, websearch, todo, skill, question, lsp, plan|read, bash, edit, write, grep, find, ls|About 30: ast_grep, ast_edit, lsp, browser, eval, web_search, memory, task, hub, computer, more|shell and exec over PTY, apply_patch (grammar constrained), plan, view_image, multi-agent, MCP|
|Permissions|allow, ask, deny rules per tool, doom-loop and external-dir guards|None built in (trust gate plus extension hooks)|read, write, exec tiers, default yolo|untrusted, on-request, granular, never|
|OS sandbox|None|None (example extensions only)|None (copy-on-write clones)|seatbelt profiles, bubblewrap plus seccomp, Windows|
|Sessions|SQLite, fork, revert via git snapshots, share|JSONL tree, branch summaries|JSONL tree, SQL and Redis backends, foreign import|JSONL rollouts plus SQLite index, fork, resume|
|Extensibility|JS plugins (hooks plus tools), agents, commands, skills, MCP client, themes|TS extensions with 36 events, skills, packages|Extensions, skills, hooks, MCP, marketplace|MCP client, skills, plugins, Claude-style `hooks.json`, profiles|
|Subagents|task tool, depth limit, experimental background|Example extension|In-process with COW worktrees and hub|Multi-agent collab|
|LSP|35 servers plus lsp tool|None|Yes, plus DAP|None|
|Browser|None (use MCP)|None|Puppeteer tool plus Chrome relay|None|
|Compaction|Auto with pruning, plugin-replaceable prompt|Threshold plus branch summaries|Six triggers, snapcompact|Inline auto plus server-side v2 (OpenAI only)|

## 3. Per-project notes

### OpenCode

Strengths: the only candidate that ships a desktop app, a headless server with an OpenAPI SDK, a TUI, a web UI, ACP, and a VS Code hook, all speaking one HTTP plus SSE contract. Provider coverage through models.dev, per-model reasoning "variants", git-snapshot undo per step, doom-loop detection, a nine-strategy fuzzy edit replacer, LSP integration, skills discovery that already reads `.claude/skills`, MCP with OAuth, and a documented plugin API with `tool.execute.before/after`, `chat.params`, `permission.ask`, `experimental.chat.messages.transform`, `experimental.chat.system.transform`, and `experimental.session.compacting`.

Weaknesses: the runtime is mid-migration from `packages/opencode` to `packages/core`, is Effect-heavy, and vendors about twenty patched `@ai-sdk/*` packages. No OS sandbox. Web search proxies to hosted Exa and Parallel MCP servers. The desktop app is beta and `scripts/predev.ts` references a script that does not exist. Hosted tendrils (share, console, Zen gateway) are all optional and disableable.

Reuse: fork the whole monorepo at a pinned tag, keep changes additive, and put Zero Code behavior in plugins. Small upstreamable core patches only where a hook is missing (sandboxed shell spawn in `packages/opencode/src/tool/shell.ts`).

### pi

Strengths: `@earendil-works/pi-ai` and `pi-agent-core` are clean libraries with the broadest OpenAI-compatible and thinking-format compat matrix reviewed, tree sessions with branch summaries, steering and follow-up queues, and a documented rebrand path. RPC and JSON modes make it embeddable.

Weaknesses: by design no permissions, no sandbox, no MCP, no subagents, no LSP, no web tools in core. No GUI. Install telemetry defaults on.

Reuse: reference implementations for compat flags and compaction; `pi-ai` is the fallback provider layer if the AI SDK path ever becomes a problem.

### oh-my-pi

Strengths: the most features of any candidate (hashline edits, LSP write-through, DAP, browser tool with relay, in-process subagents on copy-on-write clones, memory backends, 23-provider web search, marketplace compatible with the Claude plugin registry, ACP, RPC, OTLP), and the deepest local-model handling (discovery, in-band tool dialect for weak tool callers, `reasoning_content` healing, effort fallbacks).

Weaknesses: one 480K-line Bun-only package with a Rust nightly addon; not embeddable in Node or Electron; no GUI; approval defaults to yolo; no sandbox. A desktop app would be a thin client over `omp --mode rpc-ui`.

Reuse: none directly in the first build. It is the model for several later plugins (edit auto-repair, in-band tool dialect, web search) and remains an alternative engine if OpenCode's runtime churn becomes a burden.

### Codex CLI

Strengths: deep sandboxing (macOS `sandbox-exec` seatbelt profiles in `codex-rs/sandboxing/src/*.sbpl`, bubblewrap plus seccomp on Linux, Windows backends), grammar-constrained `apply_patch`, an app-server JSON-RPC protocol with generated TypeScript types, Claude-style `hooks.json` support, skills, multi-agent collab, and background PTY terminals.

Weaknesses: Responses wire only (`wire_api = "chat"` was removed), OpenAI-shaped auth and cloud crates woven into `codex-core` (75 internal crate deps), no LSP, closed desktop app, contributor CLA.

Reuse: the seatbelt profile set and the `apply_patch` grammar are the borrow targets, both under Apache-2.0 with NOTICE preserved. mlx-serve does serve `/v1/responses`, so Codex itself can also be pointed at the same local server for side-by-side testing.

## 4. Engine and model

mlx-serve (github.com/ddalcu/mlx-serve, MIT, Zig, v26.9.2 released 2026-09-09) is a native Apple Silicon server exposing OpenAI chat completions, OpenAI Responses (SSE and WebSocket, plus `/v1/responses/compact`), Anthropic Messages, and the Ollama API on one port (default 11234). It supports tools with schema-driven auto-repair, JSON-schema constrained decoding, vision, thinking as `reasoning_content`, `reasoning_effort` and `reasoning_budget_tokens`, prefix caching with an optional SSD tier, KV-cache quantization, MTP speculative decoding, continuous batching, and `GET /v1/models` with capabilities and context size. It also ships preconfigured launchers for pi, omp, OpenCode, Codex, Claude Code, and others.

Install: Homebrew 6 rejects the tap on this machine ("invalid syntax in tap"), and the formula only unpacks the release tarball and fixes dylib paths, so the plan uses the release tarball `mlx-serve-bin-macos-arm64.tar.gz` (67 MB) directly.

Model: `ARC4NUM/Qwen3.8-Flash-Next-Uncensored-MLX-Serve-4bit`, a 4-bit mlx-serve pack of the abliterated Qwen3.8-Flash-Next (125B total, about 6B active, 48 layers, hybrid Gated-DeltaNet plus full attention, 512 experts top-10, 51B-parameter n-gram table memory-mapped from disk, vision tower, MTP head, 262,144-token context). Requirements from the card: 96 GB unified memory minimum, about 107 GB on the internal SSD, mlx-serve 26.8.11 or newer. Measured on an M5 Max: about 56 tok/s serial, 60 tok/s with MTP, about 68 GB resident. Recommended agent settings: `reasoning_effort` low or medium (default is xhigh), thinking sampling temp 1.0 top_p 0.95 top_k 20, instruct sampling temp 0.7 top_p 0.8 top_k 20 presence_penalty 1.5. Model license: Qwen Community 1.0 inherited from the base model; the card is marked Apache-2.0 for the pack itself.

This machine: Apple M4 Max, 128 GB, macOS 26.6.2, 422 GB free on the internal SSD, Xcode 26.5 without the Metal toolchain (only needed for source builds, which the plan avoids).
