# Kilo Code — source audit

Head `4ee8d1b3b1f8` (2026-10-13, `Kilo-Org/kilocode`, shallow single-commit clone — lineage from structure + in-repo
fork markers, not git history). Source-verified claims cite paths relative to `packages/`.

## Identity / lineage — the Roo/Cline premise is stale

The VS Code extension is **no longer the Roo/Cline fork**. `packages/kilo-vscode/AGENTS.md` states the current
architecture outright: all editor clients are thin clients over the CLI (`packages/opencode/`, published
`@kilocode/cli`), which is "a fork of upstream OpenCode with Kilo-specific additions". The extension spawns
`kilo serve --port 0` and talks HTTP REST + SSE via the generated SDK (AGENTS.md "Products and How They Relate").
The Roo lineage survives only as migration plumbing: `kilo-vscode/src/roo-import/service.ts` (imports Roo task
history), `src/legacy-migration/sessions/` (old `condenseId` format, `legacy-types.ts:23`), and
`opencode/src/kilocode/rules-migrator.ts` / `workflows-migrator.ts`. There is no orchestrator/condense code left in
the extension itself (grep: zero hits in `kilo-vscode/src` outside legacy dirs).

Fork discipline is explicit: **273 files in `opencode/src` carry `kilocode_change` markers** (262 files count), and
`script/check-kilocode-duplication.ts` runs jscpd over kilo-only trees. Kilo additions live mostly in
`opencode/src/kilocode/**` plus separate `packages/kilo-*` packages — a deliberately upstream-diffable fork, unlike
Zero Code's unmarked divergence.

## Architecture

CLI (opencode fork: agents, tools, sessions, MCP, LSP, HTTP server) ← TUI in-process; VS Code + JetBrains thin
clients over `kilo serve`. Kilo-domain packages: `kilo-gateway` (auth + model routing + autocomplete FIM),
`kilo-memory`, `kilo-indexing`, `kilo-sandbox`, `kilo-telemetry`, `kilo-i18n`, `kilo-ui`, plus
`plugin-atomic-chat` and `kilo-console`. Cloud Agent + adoption dashboard are hosted (client plumbing in repo,
server out of scope).

## Key techniques (what it genuinely adds beyond the OpenCode fork)

- **Goal loop with durable scheduled waits** — `opencode/src/kilocode/session/goal/`. `/goal <objective>` keeps a
  session working autonomously; the model must park on time via `schedule_wakeup` / `cron_create` /
  `background_process` instead of polling (instructions.ts:16-20; caps 10 wakeups + 10 cron + 7-day horizon,
  instructions.ts:22). `goal/link.ts:8-14` is the single classification table wiring the six scheduling tools to
  the goal state machine, serializing a fire against an in-flight turn. Goal completion is self-reported
  (`goal_report`), paused on restart.
- **Wave-based orchestrator** — `opencode/src/agent/prompt/orchestrator.txt`: plan → classify subtasks into
  dependency waves → launch each wave as parallel task-tool calls → synthesize; same-file edits must serialize into
  different waves (lines 13-19). Source-verified as Kilo-authored, not upstream:
  `opencode/src/agent/agent.ts:327` marker "rename build→code, add debug/orchestrator/ask". This is a real
  improvement over Roo's sequential orchestrator (see roo-code.md).
- **Agent Manager** — parallel agents each in its own git worktree: `opencode/src/kilocode/agent-manager/` +
  `kilo-vscode/src/agent-manager/` (WorktreeManager, GitOps, PR/CI status pollers, fork-handoff, ~30 files). Kilo's
  Conductor-equivalent, first-party.
- **Checkpoints** — `opencode/src/kilocode/snapshot/` (prepare/seed/track/materialize/lock/diff-full/cleanup),
  top line marked `kilocode_change - new file`: a shadow-snapshot engine with full-diff and cleanup, feeding
  `session/revert.ts`. Source-verified.
- **Memory** — `packages/kilo-memory/src`: capture (session digest, typed-consolidation prompts, redact/reject),
  recall (topics, token budget, index), audit log, all root-bound markdown + state
  (`memory.ts` facade; `prompts/typed-consolidation.txt`, `recall/budget.ts`). Wired into the CLI via
  `opencode/src/kilocode/memory/` (ports/runtime/turn). A real file-based memory subsystem, not a memo file.
- **Sandbox** — `packages/kilo-sandbox/src`: macOS Seatbelt profiles (`seatbelt-base.ts`, `seatbelt-network.ts`),
  Linux bubblewrap (`bubblewrap.ts`), network relay with raw TLS client-hello SNI parsing
  (`tls-client-hello.ts`, `kilo-sandbox-network-relay.ts`), mutation worker/protocol. Genuine OS-sandbox
  engineering, more than upstream OpenCode ships in-repo.
- **Provider gateway** — `packages/kilo-gateway/src`: device-flow auth, OpenRouter-backed `createKilo()` provider
  (README.md), usage/balance/profile API, and autocomplete via FIM (`fim.ts`, `mistral-fim-endpoint.ts`,
  `autocomplete.ts`) with VS Code-side context tracking (`kilo-vscode/src/services/autocomplete/`). "500+ models
  zero markup" is a hosted-catalog claim — distribution, not harness.
- **Inline assist / autocomplete in CLI** — `opencode/src/kilocode/suggestion/` (tool + TUI bar): completion
  suggestions inside the terminal, not just the editor.
- **Marketplace** — `opencode/src/kilocode/marketplace/` (plugin-spec, companions, installed-metadata,
  detection across scopes) + `kilo-vscode/src/MarketplacePanelProvider.ts` + `services/marketplace/`
  install/remove actions. Installable plugins/skills/extensions; the catalog itself is hosted.
- **Transcript import** — `opencode/src/kilocode/session-resume/import.ts`: `/resume-claude`, `/resume-codex`
  import foreign transcripts through the server API so every client benefits; plus `claude-migration.ts` for
  Claude config. Aggressive migration surface.

## Strengths

- Fork hygiene: `kilocode_change` markers + dedicated `src/kilocode` tree + duplication scanner → upstream pulls
  stay tractable; this is the maintenance story most forks lack.
- Real agent machinery added on top of OpenCode: goal loop with scheduled waits, wave orchestrator, worktree
  multi-agent, checkpoints, memory — each small but source-verified.
- One runtime behind all clients (CLI-as-server, SDK-generated clients) — same shape Zero Code's V2 targets.

## Weaknesses

- The agent loop itself is inherited OpenCode; no durable admission/delivery semantics beyond upstream (their
  goal/scheduler layers sit on top, not underneath).
- Goal completion is model self-report ("Completion is your report, not independent verification",
  goal/instructions.ts:22); waits are capped and pause across restarts.
- Depth asymmetry: heavy investment in distribution (gateway billing, 16 locales, JetBrains, cloud, marketplace,
  adoption dashboards) vs harness internals; memory/indexing quality is unquantified in-repo.
- Shallow clone prevents fork-diff archaeology; `kilocode_change` markers are the only diff map.

## Works (source-verified) vs unproven

Works in-source: goal loop + scheduling tools, wave orchestrator, worktree Agent Manager, snapshot checkpoints,
memory capture/recall, OS sandbox, autocomplete plumbing, marketplace machinery, transcript import.
Unproven / hosted: gateway catalog + zero-markup billing, cloud agents, hosted indexing backend, adoption
dashboard, memory efficacy, parallel-wave orchestration quality in practice.

## What Zero Code should learn

1. The goal-loop pattern — admit an objective, require explicit `schedule_wakeup`/`cron`/process-wait parking with
   hard caps, resume on fire — is the cleanest durable long-horizon primitive seen so far in TS land and maps
   directly onto V2's durable inbox + advisory wakes without touching the provider loop.
2. Keep the wave-orchestrator prompt (dependency classification + same-file-serializes rule) as a drop-in for
   Zero Code subagents.
3. Copy the fork-discipline convention (`kilocode_change` markers, isolated addition tree, duplication CI) —
   Zero Code diverges from OpenCode lineage unmarked.
4. Autocomplete-as-tool (CLI-side suggestion + gateway FIM) is a cheap inline-assist channel Zero Code lacks.
